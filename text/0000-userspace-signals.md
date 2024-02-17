- Feature Name: userspace_signals
- Start Date: 2024-02-16
- RFC PR: (leave this empty)
- Redox Issue: (leave this empty)

# Summary
[summary]: #summary

Most of Redox's POSIX signal handling implementation can be moved to userspace,
in a way that allows changing the sigprocmask without any syscalls. Sending
signals would still be done by the kernel, with regular mutex synchronization.

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

# Detailed design
[design]: #detailed-design

# Proc scheme API

The kernel would change `/<proc>/sigstack` to `/<proc>/sigentry`, which would
provide access to the following struct:

```rust
#[repr(C)]
struct SigEntry {
    entry: usize,
    ctl_region: usize,
    altstack_base: usize,
    altstack_size: usize,
    // TODO: cpu_exception_entry?
}
```

The `entry` field is a pointer to the signal trampoline, `ctl_region` will
point to a control structure defined below. The altstack fields specify the
POSIX _sigaltstack(2)_ alternate signal stack, where the stack top is
`base+size`.

```rust
#[repr(C)]
struct SigCtlRegion {
    ctl: AtomicU2x64,   // (128-bit)
    old_ip: usize,      // eip/rip/pc 
    old_reg_a: usize,   // eax/rax/x0
    old_reg_b: usize,   // edx/rdx/x1
    queue: [RtSig; 33],
    // Current signal information
    signo: u8,
}
type AtomicU2x64 = [AtomicUsize; {64 / usize::BITS}];

#[repr(C)]
struct RtSig {
    signo: usize, // all bits except 5:0 are reserved
    sigval: usize,
}
```

The signal control region must be contained within a single page, but will only
require natural usize alignment. Userspace is encouraged to use the existing
TCB page. The ctl field is a semiatomic value, which in theory only requires
CAS of aligned pairs of bits to be supported ("AtomicU2"), but which in
practice must be atomic at least to 32 bits.

The ctl word will, in each AtomicUsize, store two half-usize bitsets `(masked,
pending)`. Theoretically, all odd bits could interleave the bitsets, so the
pending state whereas the even bits store the masked state, but since Redox
does not support any arch without 32-bit atomics, that is unnecessary. On
32-bit-atomic platforms, it will look like

```
[signal 1-16: masked then pending][signal 17-32: masked then pending][rtsignal 33-48: same][rtsignal 49-64: same]
```

and on 64-bit-atomic platforms,

```
[signal 1-32: masked then pending][rtsignal 33-64: masked then pending]
```

Architectures with 128-bit CAS, can simply store the mask in the low half and
pending in the high half.

Since signal 0 does not exist, this bitset is zero-based. The mask bits are not
necessarily the same as the sigprocmask, and instead also includes ignored
signals, via _sigaction(3)_. Libc will track manually, using thread-local
storage, which signals are ignored and which are procmasked, and thus maintain
an internal procmask and ignmask. The sigprocmask/pthread_sigmask functions, as
defined by POSIX, can only be used on the current thread.

## Kernel implementation of kill/sigqueue

The kill and/or sigqueue syscalls will still use mutex-based synchronization,
with other kernel hardware threads. This implies the only lock-free
synchronization is check-mask-then-deliver and mask-then-check-pending, where
userspace synchronizes with the kernel, possibly on another hardware thread.

The kernel will set the corresponding pending bit, followed by reading the
masked and pending bit (directly via the physaddr of the provided page). If
both are set simultaneously, the signal will be delivered.

### Delivery

Delivery entails stopping the target context, potentially using an IPI, and
getting exclusive access to the saved registers. The instruction pointer is
saved into `old_ip`, and two additional scratch registers are similarly saved.
Since the target thread is not running at that time, the mask can be set
nonatomically, knowing that the kernel still needs the same lock to send new
signals. The kernel masks all signals when entering the trampoline.

## Fork and exec

POSIX requires fork to preserve all sigactions and the sigprocmask, which would
likely be trivial considering the shared address space, as the proc scheme
struct can simply be reapplied.

Exec must preserve the procmask, and remember which signals were ignored, but
otherwise reset all nonignored sigactions. This information could be passed in
AT_SIGPROCMASK/AT_SIGIGNMASK (with both lo and hi variants on 32-bit
platforms).

TODO: what about `pthread_create`?

## SIGCONT, SIGSTOP(/SIGTSTP,SIGTTIN,SIGTTOU)

TODO

## Realtime signals

POSIX specifies that the conventional signals (typically 1-31) should be
implemented as a set; that is, multiple undelivered signals of the same number,
should be merged into one. Realtime signals must instead be implemented as a
queue, allowing multiple independent signals of the same number. Realtime
signals must also be able to provide a value, either an `int` or a pointer.

This would be implemented using the `queue` field in the signal control region.
Requiring exclusive access in the kernel to send realtime signals, this queue
can be accessed nonatomically, and would store both the signal number and
sigval.

# Drawbacks
[drawbacks]: #drawbacks

This adds complexity, and partially blurs the line between userspace and
kernel.

# Alternatives
[alternatives]: #alternatives

The obvious, and probably preferred short-term alternative, should be to
implement signal handling in the kernel, with sigprocmask and sigreturn
syscalls.

# Unresolved questions
[unresolved]: #unresolved-questions

## Multithreaded signal delivery

How should multithreaded signal delivery work?

## Partial sigprocmask handling

Since 128-bit atomics cannot be guaranteed even on x86_64, libc must be able to
detect if a signal occurs inside the sigprocmask setter itself. This is not
unsolvable, but nontrivial. Two proposed solutions: (1) specify an instruction
pointer range in which the kernel finishes the sigprocmask atomic write
(holding the kill/sigqueue lock), or (2) specify a similar range, where the
userspace signal trampoline performs the same check, and then either postpones
the signal if the sigprocmask would otherwise have blocked it, or reverts the
procmask, delivers the signal, and then retries the procmask.
