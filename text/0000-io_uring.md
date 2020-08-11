- Feature Name: io_uring
- Start Date: 2020-06-21
- RFC PR: [redox-os/rfcs#15](https://gitlab.redox-os.org/redox-os/rfcs/-/merge_requests/15)
- Redox Issue: [redox-os/redox#1312](https://gitlab.redox-os.org/redox-os/redox/issues/1312)

# Summary
[summary]: #summary

`io_uring` is a low-latency high-throughput asynchronous I/O API, inspired by
Linux's `io_uring` (since 5.1). In essence `io_uring` behaves like a regular
SPSC channel or queue: the producer pushes entries, which the consumer pops.
This interface provides two different rings for each `io_uring` instance,
namely the _submission queue_ (SQ) and the _completion queue_ (CQ). The process
making the syscall (which from now on is referred to as the _consumer_), sends
a _submission queue entry_ (SQE or SE), which the process (or kernel)
processing the syscall (referred to as the _producer_) handles, and then sends
a _completion queue entry_ (CQE or CE). The Redox implementation also allows
different roles for processes and the kernel, unlike with Linux.

# Motivation
[motivation]: #motivation

Since Redox is a microkernel, context switches will be much more frequent than
on monolithic kernels, which do a more miscellaneous work in the kernel. This
context switch overhead is even greater when using KPTI to mitigate the recent
Meltdown vulnerability; this is also the motivation behind Linux's io_uring
API, even if Redox would benefit to a greater extent.

By using two separate queues, the _submission queue_, and the _completion
queue_, the only overhead whatsoever of the syscall, is to write the submission
queue entry to a flat shared memory region, and then increment two atomic
variables (where the second is optional and only for process notification,
albeit highly recommended), in that order. Similarly, when a command completes,
all that has to be done is the same: reading from shared memory, and then
incrementing two counters. Since the channels are lock-free unless the queues
are empty (when receiving) or full (when sending), this also allows two
processes that run in parallel to serve each other's requests simultaneously in
real-time by polling the rings.

Another reason why `io_uring` is to be considered, is due to the
completion-based model for async I/O. While the readiness-based model works
greatly for network drivers, where events are triggered by hardware (or rather,
polled by the driver), on other hardware such as NVME disks, the hardware will
not read from the disk by itself, but has to be started by the driver. The way
`xhcid` handles this is by having two files allocated for each hardware
endpoint: `ctl` and `data`. Not only is it much more complex and complicated
both for the driver and the driver user, but it also requires two separate
syscalls, and thus causes more context switches than it needs to.

# Detailed design
[design]: #detailed-design

## Ring layer
### Ring operation
On the technical side, each `io_uring` instance comprises two rings - the
submission ring and the completion ring. These rings reside in shared memory,
which is either shared by the kernel and a process, or two processes. Every
individual reference to a ring uses two memory regions (typically regular
memory mappings managed by the kernel). The first region is exactly one page
large (4 KiB on x86_64, however so long as the ring header fits within a page,
architectures with smaller page sizes would also be able to use those), and
contains the ring header, which is used for atomically accessed counters such
as indices and epochs. Meanwhile, the second region contains the ring entries,
which must for performance purposes be aligned to a multiple of its own size.
Hence, with two rings for command submission and command completion, and two
regions per ring, this results in two fixed-size regions and two variably-sized
regions (dependent on the number of submission and completion entries,
respectively).

### Ring data structures

The current structure of the ring header, simplified, is defined as follows:

```rust

#[cfg_attr(target_arch = "x86_64", align(128))]
#[cfg_attr(not(target_arch = "x86_64", align(64)))]
struct CachePadded<T>(pub T);

// Generic type parameter T omitted, for simplicity.
#[repr(C)]
pub struct Ring {
    // the number of accessible entries in the ring, must be a power of two.
    entry_count: usize,

    // the index of the head of the ring, from which entries are popped.
    head_index: CachePadded<AtomicUsize>,

    // the index of the tail of the ring, to which new entries are pushed.
    tail_index: CachePadded<AtomicUsize>,

    // various status flags, currently only implemented for shutting down rings.
    status: CachePadded<AtomicUsize>,

    // a counter incremented on every push_back operation
    push_epoch: CachePadded<AtomicUsize>,

    // a counter incremented on every pop_front operation
    pop_epoch: CachePadded<AtomicUsize>,
}
```

There are two types of submission entries, which are `SqEntry32` and
`SqEntry64`; there are also `CqEntry32` and `CqEntry64` for completion entries.
The reason for these dinstinct entry sizes, is to be able to let the entries
take up less space when 64-bit addressing is not needed, or for architectures
which do not support 64-bit addressing. Apart from most other structures within
the `syscall` crate, which mainly use `usize` for integers, these entry structs
always use fixed size integers, to make sure that every byte in every struct is
used (while still allowing for future expandability). The 64-bit definitions
for these entry types are the following (possibly simplified):

```rust
// 64 bytes long
#[repr(C, align(64))]
pub struct SqEntry64 {
    // the operation code negotiated between the producer and consumer
    pub opcode: u8,
    // flags specific to entries and how they behave
    pub flags: u8,
    // the priority; greater means higher priority
    pub priority: u16,
    // the flags specific to the syscall
    pub syscall_flags: u32,
    // the file descriptor to operate on, for most syscalls
    pub fd: u64,
    // the length, for most syscalls
    pub len: u64,
    // an arbitrary number or pointer passed to the completion entry
    pub user_data: u64,
    // for syscalls taking an address, this will point to that address
    pub addr: u64,
    // for syscalls taking an offset, this specifies that that offset
    pub offset: u64,
    // reserved for future use and must be zero
    pub _rsvd: [u64; 2],
}

// 32 bytes long
#[repr(C, align(32))]
pub struct CqEntry64 {
    // the same value as specified in the corresponding submission entry
    pub user_data: u64,
    // the "return value" of the syscall, optionally the flags field can extend this
    pub status: u64,
    // miscellaneous flags describing the completion
    pub flags: u64,
    // extra eight bytes of the return value (optional)
    pub extra: u64,
}
```

Meanwhile, the 32-bit (same fields and meanings, but with different sizes):
```rust
// 32 bytes long
#[repr(C, align(32))]
pub struct SqEntry32 {
    pub opcode: u8,
    pub flags: u8,
    pub priority: u16,
    pub syscall_flags: u32,
    pub fd: u32,
    pub len: u32,
    pub user_data: u32,
    pub addr: u32,
    pub offset: u64,
}

// 16 bytes long
#[repr(C, align(16))]
pub struct CqEntry32 {
    pub user_data: u32,
    pub status: u32,
    pub flags: u32,
    pub extra: u32,
}
```

### Algorithms
Every ring has two major operations:

* `push_back`, which tries to push new entry onto the ring, or fails if the
  ring was full, or shut down
* `pop_front`, which tries pop pop an entry from the ring, or fails if the ring
  was empty, or shut down

Implementation-wise, these rings simply read or write entries, and then
increment the head or tail index, based on whether the operation is a push or
pull operation. Since the queue is strictly SPSC, the producer can assume that
it is safe to write the value and then increment the tail index, so that the
consumer is aware of the write, only after it has been done. Popping works the
same way, with the only difference being that the head index is written to and
the tail index is read.

The indices, represent regular array indices; the byte offset of the entry is
simply calculated by multiplying the size of the entry, with the index.
However, they must be decoded from the raw value in the atomics, by taking the
raw value, modulo twice the number of entries.

In other words,

```math
i \equiv r \pmod{2n}
```

where $`i`$ is the index into the array, $`r`$ the raw atomic variable, and
$`n`$ the number of entries in the ring.

Additionally, the ring buffer has a _cycle_ condition, which occurs somewhat
frequently when the indices overflow the number of entries. The _push cycle_
and _pop cycle_ flags are either 0 or 1, and are calculated by rounding down
the quotient of the index with the number of entries, to the nearest integer.
Based on the Exclusive OR (XOR) of these two flags, the _cycle_ will flip the
roles of the head and tail index when comparing them, which happens before
every push and pop operation, to check whether the ring is full or empty,
respectively.

Therefore,

```math
c_{head} = \lfloor \frac{r_{head}}{n} \rfloor
```
```math
c_{tail} = \lfloor \frac{r_{tail}}{n} \rfloor
```
```math
c = c_{head} \oplus c_{tail}
```

where $`c`$ denotes the cycle flag, and $`\oplus`$ denotes Exclusive OR.

There is also a push epoch, and a pop epoch, which are global counters for each
ring that are incremented on each respective operation. This is mainly used by
the kernel to notify a user when a ring can be pushed to (after it was
previously full), or when it can be popped from (after it was previously
empty). It also makes it simple for the kernel to quickly check those during
scheduling, if needed. The reason why these are represented as epochs, is to
allow the ring to truncate indices arbitrarily using modulo or bitwise AND, but
also to allow a process to notify itself from another thread, without pushing
any additional entry, which would not be possible for a full ring.

## Interface layer
The Redox `io_uring` implementation comes with a new scheme, `io_uring:`. The
scheme provides only a top-level file, like `debug:`, which when opened creates
a new `io_uring` instance in its _Initial_ state. By then writing the create
info (`IoUringCreateInfo`) to that file descriptor, the `io_uring` instance
transistions into the _Created_ state with basic information such as the type
of submission and completion entries, and the number of entries. In that state,
the four memory regions can then be mmapped, which are located at the following
offsets:

```rust
pub const SQ_HEADER_MMAP_OFFSET: usize = 0x0000_0000;
pub const SQ_ENTRIES_MMAP_OFFSET: usize = 0x0020_0000;
pub const CQ_HEADER_MMAP_OFFSET: usize = 0x8000_0000;
pub const CQ_ENTRIES_MMAP_OFFSET: usize = 0x8020_0000;

// "SQEs" is intentionally omitted, unlike with Linux. This may change in the
// future though.
```
When all of these are mapped, the `io_uring` instance is ready to be attached,
in one of the following ways:

* userspace to userspace
* userspace to kernel
* kernel to userspace

After a successful attachment, the instance will transition into the _Attached_
state, where it is capable of submitting commands.

### Userspace-to-userspace
In the userspace-to-userspace scenario, which has the benefit of zero kernel
involvement (except for polling the epochs to know when to notify), the
consumer process calls the `SYS_ATTACH_IORING` syscall, which takes two
arguments: the file descriptor of the `io_uring` instance, together with the
name of the scheme.

The scheme will however receive a different syscall in a regular packet, namely
`SYS_RECV_IORING`, which includes a temporary kernel-mapped buffer, containing
already-mapped offsets to the four ring memory regions, as well as an
`io_uring` instance file descriptor for the producer, linked to the consumer
instance file descriptor.

This mode is generally somewhat specialized, because the kernel may have to
poll the rings on its own (which perhaps some magic trickery of putting every
`io_uring` allocation in its own adjacent group of page tables, and checking
the "dirty" flag on x86_64), and because the buffers have to be managed
externally.

For that type of memory management to be possible, the kernel provides an
efficient way for consumers to quickly add new preallocated chunks in a buffer
pool; by invoking `SYS_DUP` on a userspace-to-userspace `io_uring` instance
file descriptor, a new pool file descriptor will be created. The pool file
descriptor, can be mmapped from an arbitrary offset, which will allocate
memory, and push an allocation entry to an internal list.

The producer can do the same with its instance file descriptor, with the
exception that the producer can only map offsets, that the consumer has
allocated. The pending allocations that the producer has not yet mapped, can be
retrieved by reading the producer pool file descriptor.

Another constraint is that the stack cannot be used at all, since unpriviliged
processes are obviously not supposed to access other process' stacks. The main
point of this however, is the low latency it offsers; with some intelligent
kernel scheduling, the kernel could place an audio driver and an audio
application on two different CPUs, and they would be to process each other's
syscalls without any context switch (apart from scheduling of course).

In the future there might be scenarios where real addresses are used rather
than offsets in this attachment mode, e.g. physical addresses for an NVME
driver. The consumer and producer are free to negotiate their behavior anyway.

When waiting for notification, the regular `SYS_ENTER_IORING` syscall may be
invoked, that will cause the kernel to wait for the ring to update, blocking
the process in the meantime. However, the primary notification mechanism for
consumers, is using event queues in a primary userspace-to-kernel io_uring; by
using the `FilesUpdate` opcode with the `EVENT_READ` flag (and `EVENT_WRITE`
for notification when a ring is no longer full), with the `io_uring` instance
file descriptor as target, an event will be triggered once the ring gets
available completion entries.

TODO: Right now the kernel polls rings using event queues or
`SYS_ENTER_IORING`, but would it make sense to optimize this by putting the
kernel virtual addresses for the rings, adjacently in virtual memory, and then
using the "dirty" page table flags instead?

### Userspace-to-kernel
The userspace-to-kernel attachment mode is similar; the process still calls
`SYS_ATTACH_IORING`, with the exception of the kernel handling the request. The
kernel supports the same standard opcode set as other schemes do, with the
advantage of being able to communicate with every open scheme for that process.

This is the recommended mode, especially for applications, since it does not
require one ring per consumer per producer, but only at least one ring on the
consumer side. The kernel will delegate the syscalls pushed onto that
`io_uring`, and talk to a scheme using a different `io_uring`. The kernel will
also manage the buffers automatically here, as it has full access to them when
processing syscall, which results in less overhead as no preregistered buffers
or other shared memory is used.

When using an `io_uring` this way, a process pushes submission entries, and
then calls `SYS_IORING_ENTER`, which is a regular syscall. The kernel then
reads and processes the submission entries, and pushes completion entries in
case some requests were already finished or not asynchronous (while `io_uring`
is perfectly suited to async by design, it can also be used to batch regular
syscalls within one syscall to reduce context switching overhead).

TODO: Are schemes that handle the file descriptors referenced when talking to
the kernel, going to be directly scheduled, in or after `SYS_IORING_ENTER`, or
is the kernel going to wait for it to be scheduled regularly first?

### Kernel-to-userspace
The kernel-to-userspace mode is the producer counterpart of the
userspace-to-kernel-mode; it shares the same properties as with the other, but
the roles are reversed. A scheme may at any time receive a request to attach an
`io_uring` where the kernel is the producer, and this will happen at most once
per scheme.

The kernel is free to use this `io_uring` whenever it wants asynchronous I/O on
a scheme that supports it, for example if a userspace-to-kernel `io_uring`
wrote to a file descriptor owned by that scheme.

One noteworthy difference between userspace-to-userspace plus
kernel-to-userspace, and userspace-to-kernel, is that the file descriptor
numbers are local to the target scheme in the userspace-to-userspace case.
Oppositely, in the userspace-to-kernel case, the producer is free to use global
file descriptors (to that process). This is akin to the regular syscall
interface and scheme handlers: the user can specify any file descriptor, and if
it belonged to a certain scheme, the kernel will then translate that file
descriptor to a file descriptor local to that scheme. The only distinction with
the `io_uring` interface is that in the userspace-to-userspace case, the
consumer must distinguish between file descriptors local to a specific scheme,
the its own global file descriptors.

Since the kernel initiates these by itself, and keeps track of state, no
dedicated syscall resembling `SYS_IORING_ENTER` needs to be called, since the
kernel can simply check whether the epochs have been incremented or not.

### Deallocation
The four `io_uring` memory regions are kernel managed, and thus they only
require a close syscall on both the consumer side and the producer side. They
can also be unmapped with `funmap` separately, before closing.

The kernel will then shutdown the opposite ring part, so that the other side of
the `io_uring` receives the last items and then also destroys its instance.

# Drawbacks
[drawbacks]: #drawbacks

Since Linux handles every syscall within ring 0, no extra communication needs
to happen when dealing with addresses; the kernel can simply map a userspace
address (if it is not already mapped), do its work, and then push a completion
entry. Redox has a similar mode: userspace-to-kernel. One downside with this
compared to how Linux works, is that the kernel has to use two rings: one
between the consumer and kernel, and one between the kernel and consumer. There
is also a direct mode, userspace-to-userspace, although it suffers from various
limitations.

Since the Redox `io_uring` in userspace-to-userspace mode has no direct kernel
involvement, all buffers will have to be registered in order for the producer
process to access those buffers. The current workaround here is to use a
different `io_uring` with the kernel as the producer and allocating buffers
there, or to simply mmap them at a somewhat higher cost using regular syscalls.
Although this might not impose that of a greater overhead than simply letting
the kernel deal with this during a syscall, for the most part it completely
eliminates the ability to use the stack for buffers, since that would be a
major issue, both for program security and safety.

An additional drawback is the increased complexity of both using asynchronous
I/O and regular blocking I/O in the kernel. The kernel may also have to keep
track of the state of certain operations, instead of bouncing between userspace
and kernel in a stack-like way, when schemes call schemes. Futures (and maybe
async/await) might help with this: Alternatively, one could limit more complex
syscalls, e.g. `SYS_OPEN` from having to be repeated by only allowing simple
paths without symlinks, which reduces complex state.

# Alternatives
[alternatives]: #alternatives

Even though it may not be strictly necessary aside from increased performance
to use SPSC queues, one could instead simply introduce `O_COMPLETION_IO` (which
would be paired with `O_NONBLOCK`), `EVENT_READ_COMPLETION` and
`EVENT_WRITE_COMPLETION`, and let the kernel retain the grants of the buffers
until events with the aforementioned flags have been sent, to indicate that the
buffers are not longer needed by the scheme process.

Additionally, while the kernel can obviously assist with notifying processes
with lower overhead, this API could be implemented in userspace by using shared
memory for communication, and futexes or even regular event queues (although
with a higher overhead due to syscalls) for notifying.

The major problem with this approach however would be buffer management, since
that would require processes to use regular syscalls every time they want to
register a buffer.  Another feature that would be possible, would be fast
communication between the kernel and userspace processes. The current Redox
`io_uring` design also allows the kernel to be the producer of the rings, which
makes it possible for traditional event queues to be completely replaced. This
also applies for other kernel schemes.

# Unresolved questions
[unresolved]: #unresolved-questions

The API is for the most part actually quite similar to Linux's API, with the
only incompatible difference of not having separated the SQ with its entries
(which Linux does). The problem however, is that `io_uring`s can have different
producers, unlike Linux which only has the kernel. Thus, there might be
motivation for a kernel-managed queue that sends submissions to the respective
schemes, and collects completions from different schemes, so that the consumer
only needs one `io_uring` instance, like on Linux.

If that were implemented, there would be a possibility for source-compatibility
with liburing, especially if the entry types are exactly the same as Linux's
(`struct io_uring_sqe` and `struct io_uring_cqe`). If the syscalls were to be
emulated, this could also result possible complete emulation.

# References
1. [https://kernel.dk/io_uring.pdf](https://kernel.dk/io_uring.pdf) - A
   reference of the io_uring API present from Linux 5.1 and onwards.
