- Feature Name: userspace_signals
- Start Date: 2024-02-16
- RFC PR: https://gitlab.redox-os.org/redox-os/rfcs/-/merge_requests/19
- Redox Issue: https://gitlab.redox-os.org/redox-os/kernel/-/issues/113

# Summary
[summary]: #summary

Most of Redox's POSIX signal handling implementation can be moved to userspace,
in a way that in particular, allows changing the sigprocmask without any
syscalls. Sending signals would still be done by the kernel, with regular mutex
synchronization, or later a userspace process manager.

# Motivation
[motivation]: #motivation

Signals are the userspace equivalent of hardware interrupts -- they interrupt
what was previously running, use the same stack, and are mostly maskable. POSIX
requires sigprocmask, which most POSIX systems implement in the kernel, using a
sigprocmask syscall.

However, as Redox is moving more and more of previous kernel functionality to
relibc, it sometimes needs critical sections where signals cannot be delivered.
This is currently the case for _open(3)_, where it would be useful not to need
to wrap each open call in two sigprocmask syscalls. The same issue will become
more significant in the future, should relibc for example emulate POSIX file
descriptors.

Moving the signal implementation to userspace will eliminate the need for the
`sigprocmask`, `sigaction`, and `sigreturn` syscalls. The relibc counterparts
will also likely be faster. Although, there will still need to exist a
`kill`/`sigqueue` syscall, or an equivalent IPC call to a process manager.

# Detailed design
[design]: #detailed-design

## Proc scheme API

The kernel would change `/<proc>/sigstack` to `/<proc>/sighandler`, which would
provide write-only access to the following struct:

```rust
#[repr(C)]
struct SigEntry {
    user_handler: usize,
    excp_handler: usize,
    thread_ctl_region: usize,
    proc_ctl_region: usize,
}
```

The `user_handler` and `excp_handler` fields are function pointers to the
signal trampoline and CPU exception handler, respectively. The
`thread_ctl_region` and `proc_ctl_region` fields point to a control structure
defined below, for thread vs process granularity. Setting `user_handler` to
zero disables user-handled signals completely for the thread. The
`excp_handler` field is however optional, and by default, CPU exceptions will
result in core dumps unless explicitly handled. The thread and process control
region is defined as follows:

```rust
#[repr(C)]
struct SigCtlRegion {
    ctl: [AtomicU64; 2],
    local_ctl: SigatomicU64, // "reentrant-atomic"

    // TODO: on x86, it would be sufficient to only save rIP and rFLAGS
    // (since pushf/popf are slow), and on aarch64, pc and a single register should
    // similarly be sufficient for reading tpidr_el0.

    old_ip: usize,      // eip/rip/pc 
    old_sp: usize,      // esp/rsp/sp
    old_reg_a: usize,   // eax/rax/x0
    old_reg_b: usize,   // edx/rdx/x1
    old_flags: usize,   // eflags/rflags/0
    q: [RtSig; 32],
    qhead: AtomicU8,
    qtail: AtomicU8,
}

const LOCAL_CTL_INHIBIT_DELIVERY_BIT: u64 = 1;

#[repr(C)]
struct ProcCtlRegion {
    allowset: AtomicU64,
    // TODO: ignmask?
}

#[repr(C)]
struct RtSig {
    signo: usize, // all bits except 6:0 are reserved
    sigval: usize,
}
```

The signal control regions, for both process and thread, must be contained
within a single page, and will 16-byte alignment. Userspace is encouraged to
reuse the existing TCB page. The ctl field consists of two _signal groups_,
namely the standard and realtime signals (starting at 33). Each such
`AtomicU64` is divided into a lower _pending set_ and upper _allowset_ half.

Since signal 0 does not exist, this bitset is zero-based. The allowset bits are
not necessarily the same as the inverted sigprocmask, but instead also includes
ignored signals, via _sigaction(3)_. Libc will track manually which signals are
ignored, which is necessary when deciding if newly unblocked signals after
`sigprocmask` should be delivered. The `sigprocmask` and `pthread_sigmask`
functions, as defined by POSIX, can only modify the current thread's mask.

## Kernel implementation of kill/sigqueue

The kill and/or sigqueue syscalls will still use mutex-based synchronization,
with other kernel hardware threads. This implies the only lock-free
synchronization is check-mask-then-deliver and unmask-then-check-pending, where
userspace synchronizes with the kernel, possibly on another hardware thread.

The kernel will set the corresponding pending bit, followed by reading the
masked and pending bit simultaneously, the logical AND of which, is the set of
deliverable signals. If nonempty, the kernel will unblock the thread and set an
internal flag indicating that a signal is incoming.

Sending signals to a process rather than thread, is done by first setting the
bit in the process-wide pending set, followed by linearly searching the TCBs
for a thread that has not blocked that signal. A subsequent no-op write to that
thread's `ctl` with Release ordering, should synchronize that earlier write to
the process-wide set, when the trampoline later reads its thread-specific `ctl`
(with Acquire ordering) followed by reading the process-wide mask, and deciding
which signal to deliver.

### Delivery

When the kernel delivers a signal, it unblocks the thread potentially sending
an IPI, and when the context is switched to, it has exclusive access to the
saved registers. The instruction and stack pointers, as well as some
miscellaneous registers, are saved (using nonatomic accesses) to the respective
fields of the thread control region. It will also set the _inhibit flag_.

The _inhibit flag_ allows temporarily preventing the kernel from jumping the
userspace context to the signal trampoline, without affecting how threads are
awoken etc. This flag allows async-signal-safe functions to easily and
efficiently disabling signals during short critical sections.

#### Trampoline

The `user_handler` field, points to the _signal trampoline_. The kernel will
save a few registers, some of which can be used as scratch registers. The
trampoline will need to calculate the new stack pointer, taking into account
the potential alternate signal stack (`sigaltstack`). On x86_64 this looks like

TODO: see sigentry in
https://gitlab.redox-os.org/4lDO2/relibc/-/blob/usignal/redox-rt/src/arch/x86_64.rs?ref_type=heads
until the asm is mature.

## Fork, exec, pthread_create

POSIX requires fork to preserve all sigactions and the sigprocmask, which would
likely be trivial considering the shared address space, as the proc scheme
struct can simply be reapplied.

Exec must preserve the procmask, and remember which signals were ignored, but
otherwise reset all nonignored sigactions. This information is passed in
AT_SIGPROCMASK_{LO,HI}/AT_SIGIGNMASK_{LO,HI} with both lo and hi variants on
32-bit platforms.

`pthread_create` requires the pending set to start as empty, and the mask is
inherited from the 'parent' thread.

## SIGCONT, SIGSTOP(/SIGTSTP,SIGTTIN,SIGTTOU), SIGKILL

POSIX states that SIGKILL and SIGSTOP are not maskable, and cannot be handled
in userspace or ignored. Luckily, this frees up 4 additional bits. The "SIGKILL
masked", "SIGKILL pending", and "SIGSTOP masked" bits, instead indicate whether
the action for SIGTSTP, SIGTTIN, and SIGTTOU, is equivalent to the hardcoded
SIGSTOP action. If they are unset, they can be ignored or handled, and masked,
like the other signals.

The SIGCONT signal cannot be ignored either, in the sense that sending a
SIGCONT will always continue the target process, but userspace can choose
whether or not it will be caught. The pending and masked bits for SIGCONT will
thus have the same behavior as regular signals, except `kill` will
unconditionally transition the thread from _stopped_ to _blocked_ first.

POSIX requires the generation of SIGCONT to discard all pending stop signals,
and vice versa. Since the `kill` implementation is mutex-synchronized, this
should (TODO?) be relatively easy to synchronize.

## Realtime signals

POSIX specifies that the conventional signals (typically 1-31) should be
implemented as a set; that is, multiple undelivered signals of the same number,
should be merged into one. Realtime signals must instead be implemented as a
queue, allowing multiple independent signals of the same number. Realtime
signals must also be able to provide a value, either an `int` or a pointer.

This would be implemented using the `queue` field in the signal control region.
The `qhead` and `qtail` atomic fields together divide the queue array into a
consumer-owned and producer-owned half. The consumer half shall thus be read
nonatomically by the thread, and the producer half written nonatomically by the
kernel. The `qtail` field shall be written only by the producer (kernel), and
the `qhead` field by the consumer (thread).

Since the consumer half is exclusively owned by the thread, it can dynamically
update the pending bits accordingly based on which nonmasked realtime signals
are present in the queue. The kernel will set the pending bits as usual when
sending realtime signals, which for synchronization reasons, must be done
_after_ the actual entry is enqueued.

This iteration will likely be very quick, so long as the number of possible
signals does not significantly increase. Should this become a performance
issue, the queue array may be converted to a structure-of-arrays, where the
signal numbers can be packed, and counted quickly using SIMD. POSIX only
requires the implementation to provide 8 realtime signals, and the threading
implementation requires two additional signals (cancellation and timer).

## raise

Raise will initially be implemented using a regular thread-specific kill
syscall, but that should be possible to bypass.

## sigprocmask/pthread_sigmask

Changing the signal mask (equivalently, the inverted allowset), is done fully
in userspace. The inhibit bit is set, the allowsets for each group are
atomically swapped while simultaneously reading the pending set, and there are
pending unblocked signals, then at least one will be delivered before the *mask
function returns.

Since the allowset strictly is writable only by the target thread, it can be
modified without necessitating a CAS loop, on x86 which supports XADD (atomic
fetch_add). Specifically, swapping only the allowset is done using
`word.fetch_add(new_allowset.wrapping_sub(old_allowset))`.

## sigaction

Sigaction will be implemented entirely in libc. With signals temporarily
disabled in the `sigaction` function itself, it can use regular mutex-based
synchronization, including synchronization between `sigaction` and the signal
trampoline running on other threads. Setting the action to SIG_IGN will modify
the ignmask and clear the allowset bit for the respective signal. Setting it to
SIG_DFL will either modify the signal-is-stop bits for SIGTSTP/SIGTTIN/SIGTTOU,
or set it to a builtin default handler.

POSIX allows the sigaction to change between the generation and delivery of a
signal, allowing sigaction to be weakly ordered and only synchronize against
the signal trampoline. (TODO: is this interpretation correct?)

# sigwait, sigsuspend, etc

POSIX does not differentiate between accepting (i.e. `sigwait`ing) and
delivering (i.e. trampoline runs) (TODO: correct?). Thus, it should be valid to
implement these internally with a few additional checks in the trampoline. That
said, these functions can probably avoid the trampoline entirely, by using the
inhibit bit and simply catching `EINTR`.

# Drawbacks
[drawbacks]: #drawbacks

This obviously adds complexity, and partially blurs the line between userspace
and kernel. The TCB will also significantly grow in size, some of which also
needs to store pthread information. However, from a microkernel perspecive, it
would be useful to move as much of POSIX logic as possible to userspace. It
would also likely improve the performance of signals, even compared to existing
monolithic kernels such as Linux.

Since the sigaltstack logic is done in the trampolines, there's a large amount
of assembly, but not significantly more than other libcs' signal trampolines on
monolithic kernels.

# Alternatives
[alternatives]: #alternatives

The kernel already has a basic signal implementation in the kernel. It would be
possible to extend this to include realtime signals, sigwait/sigsuspend, and
implement all the sigaction flags. However, this will severely limit the
ability for relibc to quickly protect critical sections in async-signal-safe
functions.

Alternatively, it would be possible to only implement the logic behind the
_inhibit_ bit. That said, this breaks down if larger critical sections are
used, that may internally block. In those cases, sigprocmask would likely be
used to disable all signals inside that section, which would suffer from the
same base syscall latency (usually a few hundred cycles), and this needs to be
called twice.

# Unresolved questions
[unresolved]: #unresolved-questions

## Who should send the signals?

In this proposal, the kernel will be responsible for sending the signals, and
userspace will merely control sigprocmasking, and raising some signals on its
own. However, if performance is not sufficiently significant for this to stay
in the kernel, it might make sense for _kill(3)_ and/or _sigqueue(3)_ to be
implemented as a (fast synchronous) IPC call to the process manager. A process
manager has been suggested as a way for the kernel to only abstract _contexts_,
and let that manager define _threads_, _processes_, _sessions_, and _process
groups_.

## Memory orderings

Acquire-Release should be sufficient. TODO
