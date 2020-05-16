- Feature Name: ptrace
- Start Date: 2019-06-08
- RFC PR: (leave this empty)
- Redox Issue: (leave this empty)

# Summary
[summary]: #summary

A ptrace-like feature for tracing processes in Redox OS

# Motivation
[motivation]: #motivation

Currently, we have no way for debuggers to work in Redox. We provide
no interface for tracing a process' system calls or instructions, and
no interface for managing another process' memory.

A good first step for implementing `gdb` or a similar utility would be
to implement a Linux `ptrace(...)` alternative for Redox. This should
not only open up the possibility for debuggers, but also system-call
translation processes like WINE, perhaps for Linux compatibility which
would rid us the problem of porting software.

And even with *pure* `ptrace`, without any register or memory reading,
one can still implement the immensely useful tool `strace`, which
could serve as an alternative to recompiling the kernel with system
call debugging turned on or off. This is probably what we should focus
on getting to work initially, before getting to the good stuff.

# Detailed design
[design]: #detailed-design

The Linux ptrace interface is sometimes considered a huge mistake due
to its inconsistence and it being just one massive function. The Redox
interface will have to take that in mind, as well as remove any
duplicate or otherwise redundant functions.

All process-controlling functions are implemented as one kernel
scheme, `proc:`. Opening it up with the `pid` of a tracee and a path
will perform a specific operation on the process. The benefit of using
schemes as opposed to a Linux-style function is that we get the
ability to disallow this feature using namespacing for free. We also
allow multiplexing using `event:` and therefore can be used in a
nonblocking fashion just like any other file descriptor.

## Process trace

Opening `proc:<pid>/trace` will give you a file which you can write
proc-related functions, and closing the file descriptor will detach
from it automatically. If any breakpoint is set when the file is
closed, they are deleted and the process is resumed. Only *one* tracer
can control a process, as I am too close-minded to come up with a
design that would make sense for running multiple tracers on a single
tracee.

That said, if the tracer has the flag `O_EXCL` will instead send
`SIGKILL` to the tracee when the tracer closes its file. This is to
prevent any ptrace-contained processes from breaking out. (`O_EXCL`
can be thought of as meaning the tracer is the only one who controls
the process, and the process can't live on its own)

Another flag used in `open` is `O_TRUNC` which will stop the process
*immediately*. This can be compared to using `PTRACE_ATTACH` on Linux
as opposed to `PTRACE_SEIZE`. (`O_TRUNC` can be thought of as
*truncating*/clearing the file's execution. It's a stretch, but I have
no better idea)

The most important operation of `ptrace` is of course to put a
breakpoints! Redox tries to unify the Linux event system and the
breakpoint system by making the input a bitflags with *one or more*
breakpoints. It will return when the first breakpoint/event specified
within the bitmask is reached, which in case it'll add that event for
reading using the `read` system call (see events below). If an event
is already set, the `write` returns immediately. (slight exception is
`O_NONBLOCK`, see that below as well)

Each breakpoint is set and optionally awaited using the `write` system
call. Each such call will also resume the tracee in case it's stopped
after another breakpoint. So if you write a value with no stop bits
set, the program will run to completion. The exception is manually
specifying `PTRACE_FLAG_WAIT` (even in blocking mode, see below),
which will - unless any new stop is set - only wait for an existing
breakpoint to be reached.

### Breakpoints

- If `PTRACE_STOP_PRE_SYSCALL` is set, the tracee will break on the
  next start of a syscall. This diverges from Linux' way of using
  `PTRACE_SYSCALL` for *both* pre- and post- syscall. However it's for
  a good reason: Signals can occur in the middle of a syscall, and
  unlike Linux which just delays the signal, we should go the simplest
  route to minimize kernel code size and let the user choose the
  behavior they want and not choose for them.
- If `PTRACE_STOP_POST_SYSCALL` is set, the tracee will break at the
  end of a syscall, when the return value has just been set in the
  appropriate register.
- If `PTRACE_STOP_SINGLESTEP` is set, the tracee is stopped after the
  execution of just one assembly instruction. If used together with
  any system-call trace, the system-call method will take precedence
  and allow you to fine-grane how that should work. (Not a special
  case, the syscall trace returns before the instruction returns and
  thus is what is used by the multiplexing trace call!)
- If `PTRACE_STOP_SIGNAL` is set, the tracee is stopped before next
  signal is handled. To the break event (see section on events below),
  the signal number is pushed as the first parameter, and the pointer
  to the signal handler as the second parameter. The pointer can help
  you detect whether a signal will be handled by kernel space or
  userspace, by detecting constants such as `SIG_DFL` and `SIG_IGN`.
- If `PTRACE_STOP_BREAKPOINT` is set, the tracee is stopped when the
  Breakpoint Exception, interrupt number 3, is triggered. This is
  commonly caused by the `int3` instruction with opcode `0xCC` on
  x86_64. The default behavior for breakpoint exceptions is to exit
  the process with the `SIGTRAP` signal, but you can unfortunately not
  catch this with `PTRACE_STOP_SIGNAL` due to the fact that signals
  are sent in a way that never goes through the signal
  handler. Instead of the microkernel working around this just to
  cause an ambigious breakpoint event, the two causes of signals are
  separated. Like the signal breakpoint, the default behavior (to exit
  the process) can be ignored using `PTRACE_FLAG_IGNORE` (read on
  flags below).
- If `PTRACE_STOP_EXIT` is set, the tracee is stopped before the
  process exits. Exits here is from all kinds of contexts, like
  *after* signals (so they will first raise `PTRACE_STOP_SIGNAL` if
  selected), *during* an exit syscall (will never reach
  `PTRACE_STOP_POST_SYSCALL`), as well as when caused by an hardware
  interrupt such as when an out-of-bounds read occurs. You cannot
  abort the exit and continue running the program, but you can inspect
  everything just like any other breakpoint.

### Non-breakpoint events

These events will not stop the tracee, but rather keep running in the
background until whatever breakpoint was set alongside this, was reached.

- If `PTRACE_EVENT_CLONE` was set, the tracer will wake up when the
  traee creates a new child process. An event will be delivered to the
  tracer with the PID as the first parameter. The child process will
  be in a stopped state, but unless attached to with a separate
  tracer, it will be restarted upon the next ptrace invocation.

### Flags

- If `PTRACE_FLAG_IGNORE` is set, the general action being done is
  aborted and returned early. If this is set immediately after a
  pre-syscall breakpoint, the system call is not executed but rather
  by setting the registers, *the tracer* can handle the system
  call. This behavior is known as "sysemu" on Linux. This
  general-purpose flag also lets you ignore tracee signals (except for
  `SIGKILL` which is off-limits and will ignore your wishes).
- If `PTRACE_FLAG_WAIT` is set, the `write` call will not return
  before the breakpoint is reached, but rather await that. This is the
  default behavior whenever `O_NONBLOCK` is not set, but this flags
  lets nonblocking tracers override that behavior. As explained
  briefly above, this flag will not restart a stopped tracee unless a
  new stop bit was set - which is behavior *not* replicated by default
  without `O_NONBLOCK`.

---

Because `ptrace` does **not** rely on signals, when a process is
ptrace-stopped (such as attaching to the tracee with `O_TRUNC`
explained above) you can send `SIGCONT` without actually restarting
the process. The process is restarted only using a ptrace operation or
when the tracer file handle is closed. This signal is instead just
scheduled to get handled whenever the tracee starts, which allows the
tracee to raise `SIGSTOP` and let the tracer to restart it only after
a ptrace operation was completed.

When the tracee exits (after any selected `PTRACE_STOP_EXIT`
breakpoints are invoked), any blocking operation depending on it stops
and instead returns `ESRCH`. It does not, however, reap the zombie
process. Therefore, if the tracee is your own child process you should
invoke `waitpid` immediately after a `ESRCH` error, which will also
allow you to obtain the exit status in the normal fashion, without
putting a breakpoint specifically on exit.

### Events

Events give the tracer information about breakpoints or actions the
tracee has taken. There are two types of events: Breakpoint events,
and non-breakpoint events. Only breakpoint events stop the tracee when
reached, other events only wake up the tracer, while the tracee keeps
going. The way you receive events is by `read`ing a `PtraceEvent`
structure from the file. Reads are not blocking, and will return `0`
when no event was able to be read.

Events are read sequencially, i.e. follow first-in-last-out. The
standard behavior for handling non-breakpoint events is to read them
all and then retry waiting for the breakpoint to be reached using
`PTRACE_FLAG_WAIT`. Any unread events from the last operation will
cause a new one to return immediately, in order to prevent a possible
race condition where you think you've read all events but another one
occurs right when want to retry the wait for a breakpoint to be reached.

The structure has a value `cause` specifying what bit caused the
tracer to wake up, as well as a set of values like `a` (first
parameter), `b` (second parameter), `c` (third parameter), etc. For
example, if the input was `PTRACE_STOP_SIGNAL | PTRACE_EVENT_CLONE`,
the bitmask may be either `PTRACE_STOP_SIGNAL` or `PTRACE_EVENT_CLONE`
depending on which event was hit first. The `a` value of
`PTRACE_STOP_SIGNAL` is the signal number which caused the breakpoint
to be hit, while the `a` value of `PTRACE_EVENT_CLONE` is the PID of
the tracee's new child process.

### Nonblocking mode

In nonblocking mode, a ptrace call without the `PTRACE_FLAG_WAIT` bit
set will return `1` immediately. Any breakpoint specified is set, and
will like usual overwriting any existing breakpoints. Note that the
file will send events to the `event:` scheme, meaning you can
multiplex multiple tracers.

`EVENT_READ` is triggered whenever the first event arrives. Since an
event only gets pushed to the stack if it's within the specified write
bitmask, all events in the stack are of interest and this notification
means you should immediately read them all.

`EVENT_WRITE` is reserved, for now.

## Modify registers

Another important part of Linux `ptrace` is reading and writing
registers. Opening the file `proc:<pid>/regs/int` will allow you to
`read`/`write` a struct consisting of the integer values of all
registers. The same for `proc:<pid>/regs/float`, but for all
floating-point values. This is similar to Linux's `GETREGS`/`SETREGS`.

## Modify memory

There are many different ways to read a process' memory in Linux, but
Redox should put effort in unifying these functions. Namely, the
`proc:` scheme is an excellent candidate for a unified
memory-modifying system. Opening `proc:<pid>/mem` will allow you to
seek/read/write around the memory of another process. This is a
unification of the following calls in Linux:

- `/proc/<pid>/mem`
- `PTRACE_POKEDATA`/`PTRACE_PEEKDATA`
- `process_vm_readv`/`process_vm_writev`

## Security

By default a process should only be allowed to control a process owned
by the current user, as well as being an anchestor of the process,
direct or indirect. The main motivation for allowing indirect
subprocesses is so one can trace threads of a direct subprocess.

This restriction is lifted by processes owned by `root`, which can
trace any process. In the future, a capability-like system could be
put in place to allow specific executables to trace any process owned
by the current user without having root access.

## Example

```rust
let pid = unsafe { syscall::clone()? };
if pid == 0 {
    // This is the child

    // Wait until parent is ready to trace
    syscall::kill(syscall::getpid()?, syscall::SIGSTOP)?;

    println!("Some prints here");
    println!("Some other syscalls");
    eprintln!("One can even print to STDERR");
    // Do things here. `fexec` another process, maybe?
} else {
    // This is the parent

    // Wait for the child-initiated SIGSTOP to complete.
    syscall::waitpid(pid, &mut status, WUNTRACED)?;

    // ptrace attach: Stop the process using internal ptrace mechanism
    // (not SIGSTOP!)
    let mut trace = OpenOptions::new()
        .read(true)
        .write(true)
        .truncate(true)
        .open(&format!("proc:{}/trace", pid))?;
    // obtain a handle to the process registers
    let mut regs = File::open(&format!("proc:{}/regs/int", pid))?;
    let mut status = 0;

    // It is safe to schedule a continuation of child process, because
    // it is still stopped by ptrace
    syscall::kill(pid, SIGCONT)?;

    trace.write(syscall::PTRACE_STOP_PRE_SYSCALL | syscall::PTRACE_EVENT_CLONE)?;
    // Mostly ignore event... usually you can get some interesting
    // data from it
    let mut event: PtraceEvent = PtraceEvent::default();
    trace.read(&mut event)?;
    while event.cause & syscall::PTRACE_EVENT_MASK != 0 {
        // In reality, you'll actually want to handle this event, or
        // it makes no sense to listen for it at all. This is just an
        // example to show you how you can handle non-breakpoint
        // events though.
        trace.write(syscall::PTRACE_FLAG_WAIT)?;
        trace.read(&mut event)?;
    }
    // This assertion is safe because if the process exits, the write
    // call returns ESRCH
    assert_eq!(event.cause, syscall::PTRACE_STOP_PRE_SYSCALL)?;

    let mut registers = syscall::IntRegisters::default();
    regs.read(&mut registers)?;

    println!("System call: {}", registers.orig_rax);
    println!("Replacing with exit!");

    registers.orig_rax = syscall::SYS_EXIT;
    registers.rdi = 0;

    regs.write(&registers)?;

    // trace.write(syscall::PTRACE_STOP_POST_SYSCALL)?; // wait for the completion of the system call

    trace.write(syscall::PtraceFlags::empty())?; // don't set any stops, rather run the program to the end, which is like right now
    syscall::waitpid(pid, &mut status, 0)?; // reap zombie process

    // trace file dropped here: process tracing detached and process
    // implicitly resumed if it hadn't already been, y'know, killed
}
```

# Drawbacks
[drawbacks]: #drawbacks

There is no drawback to introducing more features, except for the mere
size of the microkernel being increased. This is worth it as `ptrace`
will open up a lot of different possibilities, maybe even so that we
can eventually run linux programs on Redox.

# Alternatives
[alternatives]: #alternatives

One could, of course, use the Linux way of doing things. I think
schemes provide excellent interfaces with a clear sense of ownership
("This ptrace invocation operates on this tracee, not anything else"),
and I'm especially convinced of using them when that means we can
leverage the existing, excellent, namespacing support ([see
example](https://gitlab.redox-os.org/redox-os/contain/blob/6a1c070381f2c8b56c688c8cca454a818ee72520/src/main.rs#L21-33)).

An alternative to using a unified `proc:` scheme would be to, of
course, split it up into one for ptrace + registers and one for
writing memory. This was what the original RFC first suggested, before
[@zen3ger](https://gitlab.redox-os.org/zen3ger) mentioned how Solaris
implements a `ptrace(...)` function as a userspace library over their
ProcFS.

There are lots of possible alternatives, one of which was implemented
and tried out. However, out of the ones I've considerd, this one
should be the most scalable over time.
