- Feature Name: io_uring
- Start Date: 2020-06-21
- RFC PR: (leave this empty)
- Redox Issue: (leave this empty)

# Summary
[summary]: #summary

A low-overhead high-performance asynchronous I/O API, inspired by Linux's `io_uring` (since 5.1). In essence `io_uring` behaves like a regular SPSC channel or queue: the producer pushes entries, which the consumer pops. This interface provides two different rings for each `io_uring` instance, namely the _submission queue_ (SQ) and the _completion queue_ (CQ). The process making the syscall (which from now on is referred to as the _consumer_), sends a _submission queue entry_ (SQE or SE), which the process (or kernel) processing the syscall (referred to as the _producer_) handles, and then sends a _completion queue entry_ (CQE or CE). 

# Motivation
[motivation]: #motivation

Since Redox is a microkernel, context switches will be much more frequent than on monolithic kernels, which do a more miscellaneous work in the kernel. This context switch overhead is even greater when using KPTI to mitigate the recent Meltdown vulnerability; this is also the motivation behind Linux's io_uring API, even if Redox would benefit to a greater extent. By using two separate queues, the _submission queue_, and the _completion queue_, the only overhead whatsoever of the syscall, is to increment two atomic variables (where the second is optional and only for process notification, albeit highly recommended), and write the submission queue entry to a flat shared memory region. Similarly, when a command completes, all that has to be done is the same: incrementing two counters and reading from shared memory. Since the channels are lock-free unless the queues are empty (when receiving) or full (when sending), this also allows two processes that run in parallel to serve each other's requests simultaneously in real-time by polling the rings.

Another reason why `io_uring` is to be considered, is due to the completion-based model for async I/O. While the readiness-based model works greatly for network drivers, where events are triggered by hardware (or rather, polled by the driver), on other hardware such as NVME disks, the hardware will not read from the disk by itself, but has to be started by the driver. The way `xhcid` handles this is by having two files allocated for each hardware endpoint: `ctl` and `data`. Not only is it much more complex and complicated both on for the driver and the driver user, but it also requires two separate syscalls, and thus causes more context switches than it needs to.

# Detailed design
[design]: #detailed-design

## Ring operation
On the technical side, each `io_uring` instance comprises two rings - the submission ring and the completion ring. These rings reside in shared memory, which is either shared by the kernel and a process, or two processes. Every individual reference to a ring uses two memory regions (or regular memory mappings in kernel space). The first region is exactly one page large (4 KiB on x86_64, however so long as the ring header fits within a page, architectures with smaller page sizes would also be able to use those), and contains the ring header, which is used for atomically accessed counters such as indices and epochs. Meanwhile, the second region contains the ring entries, which must for performance purposes be aligned to a multiple of its own size. Hence, with two rings for command submission and command completion, and two regions per ring, this results in two fixed-size regions and two variably-sized regions (dependent on the number of submission and completion entries, respectively).

### Data structures

The current structure of the ring header, simplified, is defined as follows:

```rust

#[cfg_attr(target_arch = "x86_64", align(128))]
#[cfg_attr(not(target_arch = "x86_64", align(64)))]
struct CachePadded<T>(pub T);

// Generic type parameter T omitted, for simplicity.
pub struct Ring {
    // used to relocate the ring from the static MMAP offsets,
    base_rel: usize,

    // the number of accessible entries in the ring, must be a power of two.
    entry_count: usize,

    // the index of the head of the ring, on which entries are popped.
    head_index: CachePadded<AtomicUsize>,

    // the index of the tail of the ring, on which new entries are popped.
    tail_index: CachePadded<AtomicUsize>,

    // various status flags, currently only implemented for shutting down rings.
    status: CachePadded<AtomicUsize>,

    // counts the number of references into the ring
    // TODO: Shouldn't this only be in the kernel?
    refcount: CachePadded<AtomicUsize>,

    // a counter incremented on every push_back operation
    push_epoch: CachePadded<AtomicUsize>,

    // a counter incremented on every pop_front operation
    pop_epoch: CachePadded<AtomicUsize>,
}
```

There are two types of submission entries, which are `SqEntry32` and `SqEntry64`; there are also `CqEntry32` and `CqEntry64` for completion entries. The reason for these dinstinct entry sizes, is to be able to let the entries take up less space when 64-bit addressing is not needed, or for architectures which do not support 64-bit addressing. Apart from most other structures within the `syscall` crate, which mainly use `usize` for integers, these entry structs always use fixed size integers, to make sure that every byte in every struct is used (while still allowing for future expandability). The 64-bit definitions for these entry types are the following (possibly simplified):

```rust
#[repr(C, align(64))]
#[derive(Clone, Copy, Debug, Eq, Hash, PartialEq)]
pub struct SqEntry64 {
    pub opcode: u8,         // the operation code negotiated between the producer and consumer
    pub flags: u8,          // flags specific to entries and how they behave
    pub priority: u16,      // the priority; greater means higher priority
    pub syscall_flags: u32, // the flags specific to the syscall
    pub fd: u64,            // the file descriptor to operate on, for most syscalls
    pub len: u64,           // the length, for most syscalls
    pub user_data: u64,     // an arbitrary number or pointer passed to the completion entry
    pub addr: u64,          // for syscalls taking an address, this will point to that address
    pub offset: u64,        // for syscalls taking an offset, this specifies that that offset
    pub _rsvd: [u64; 2],    // reserved for future use and must be zero
}

/// A 32-bit Completion Queue Entry. Takes up 16 bytes of space.
#[repr(C, align(16))]
#[derive(Clone, Copy, Debug, Eq, Hash, PartialEq)]
pub struct CqEntry64 {
    pub user_data: u64,     // the same value as specified in the corresponding submission entry
    pub status: i32,        // the "return value" of the syscall, optionally the flags field can extend this
    pub flags: u32,         // miscellaneous flags describing the completion
}
```

Meanwhile, the 32-bit (same fields and meanings, but with different sizes):
```rust
#[repr(C, align(32))]
#[derive(Clone, Copy, Debug, Eq, Hash, PartialEq)]
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

#[repr(C, align(16))]
#[derive(Clone, Copy, Debug, Eq, Hash, PartialEq)]
pub struct CqEntry32 {
    pub user_data: u32,
    pub status: i32,
    pub flags: u32,
    pub extra: u32,
}
```

### Algorithms
Every ring have two major operations:

* `push_back`, which tries to push new entry onto the ring, or fails if the ring empty full, or shut down
* `pop_front`, which tries pop pop an entry from the ring, or fails if the ring was empty, or shut down

Implementation-wise, these rings simply increment the head or tail index, based on whether the operation is a push or pull operation. Since the queue is strictly SPSC, the producer can assume that it is safe to write the value and then increment the tail index, so that the consumer is aware of the write only after it has been done. Popping works the same way, with the only difference being that the head index is written to and the tail index is read. There is also a push epoch, and a pop epoch, which are global counters for each ring that are incremented on each respective operation. Not only is that an easy way to check whether the ring has been updated, but it also supports using a futex for notification, although the preferred notification type is using traditional event queues or event io_urings.

## Software initialization
The Redox `io_uring` implementation comes with a new scheme, `io_uring:`. The scheme provides only a top-level file, like `debug:`, which when opened creates a new `io_uring` instance in its _Initial_ state. By then writing the create info (`IoUringCreateInfo`) to that file descriptor, the `io_uring` instance transistions into the `Created` state with basic information such as which type of submission and completion entries, and the number of entries. In that state, the four memory region can then be mmapped, which are located at the following offsets:
```rust
pub const SQ_HEADER_MMAP_OFFSET: usize = 0x0000_0000;
pub const SQ_ENTRIES_MMAP_OFFSET: usize = 0x0020_0000;
pub const CQ_HEADER_MMAP_OFFSET: usize = 0x8000_0000;
pub const CQ_ENTRIES_MMAP_OFFSET: usize = 0x8020_0000;
// "SQES" is intentially omitted, unlike with Linux
```
When all of these are mapped, the `io_uring` instance is ready to be attached to a scheme, which can be either a kernel scheme scheme, or a userspace scheme. The attachment is done using the `SYS_ATTACH_IORING` syscall, which takes the file descriptor of the `io_uring` instance, together with any file descriptor from the target scheme. The scheme itself receives a different syscall as a regular packet, but `SYS_RECV_IORING`, which includes the target internal file number of the file descriptor that belonged to the scheme, plus a temporary kernel-mapped buffer, which contains already-mapped offsets to the four ring memory regions, as well as some flags and such. The target file descriptor is only used by the kernel to figure out which scheme to call, but the scheme is free to use it to associate `io_uring` instances with different users of that scheme. At this stage, normal ring operation occurs.

TODO: Implement `io_uring` freeing.

# Drawbacks
[drawbacks]: #drawbacks

Since Linux handles every syscall within ring 0, no extra communication needs to happen when dealing with addresses; the kernel can simply map a userspace address (if it is not already mapped), do its work, and then push a completion entry. Since the Redox `io_uring` functions between user processes through schemes, all buffers will have to be registered in order for the producer process to access those buffers. The current workaround here is to use a different `io_uring` with the kernel as the producer and allocating buffers there. Although this might not impose that of a greater overhead than simply letting the kernel deal with this during a syscall, for the most part it completely eliminates the ability to use the stack for buffers, since that would be a major issue, both for program security and safety.

# Alternatives
[alternatives]: #alternatives

Even though it may not be strictly necessary aside from increased performance to use SPSC queues, one could instead simply introduce `O_COMPLETE_IO` (which is paired with `O_NONBLOCK`), `EVENT_READ_COMPLETE` and `EVENT_WRITE_COMPLETE`, and let the kernel retain the grants of the buffers until events with the aforementioned flags have been sent, to indicate that the buffers are not longer needed by the scheme process.

Additionally, while the kernel can obviously assist with notifying processes with lower overhead, this API could be implemented by using shared memory for communication, and futexes or even regular event queues (although with a higher overhead due to syscalls) for notifying. The major problem with this approach however would be buffer management, since that would require processes to use regular syscalls every time they want to register a buffer. Another feature that would be possible, would be fast communication between the kernel and userspace processes. The current Redox `io_uring` design also allows the kernel to be the producer of the rings, which makes it possible for traditional event queues to be completely replaced. This also applies for other kernel schemes.

# Unresolved questions
[unresolved]: #unresolved-questions

The API is for the most part actually quite similar to Linux's API, with the only incompatible difference of not having separating the SQ with its entries (which Linux does). The problem however, is that `io_uring`s can have different producers, unlike Linux which only has the kernel. Thus, there might be motivation for a kernel-managed queue that sends submissions to the respective schemes, and collects completions from different schemes, so that the consumer only needs one `io_uring` instance, like on Linux. If that were implemented, there would be a possibility for source-compatibility with liburing, especially if the entry types are exactly the same as Linux's (`struct io_uring_sqe` and `struct io_uring_cqe`). If the syscalls were to be emulated, this could also result possible complete emulation.

# References
1. [https://kernel.dk/io_uring.pdf](https://kernel.dk/io_uring.pdf) - A reference of the io_uring API present from Linux 5.1 and onwards.
