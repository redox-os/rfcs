- Feature Name: io_uring
- Start Date: 2020-06-21
- RFC PR: (leave this empty)
- Redox Issue: (leave this empty)

# Summary
[summary]: #summary

An low-overhead high-performance asynchronous I/O API, inspired by Linux's `io_uring` (since 5.1). In essence `io_uring` behaves like a regular SPSC channel or queue: the producer pushes entries, which the consumer pops. This interface provides two different rings for each `io_uring` instance, namely the _submission queue_ (SQ) and the _completion queue_ (CQ). The process making the syscall (which from now on is referred to as the _consumer_), sends a _submission queue entry_ (SQE or SE), which the process (or kernel) processing the syscall (referred to as the _producer_) handles. 

# Motivation
[motivation]: #motivation

Since Redox is a microkernel, context switches will be much more frequent than on monolithic kernels, which do a more miscellaneous work in the kernel. This context switch overhead is even greater when using KPTI to mitigate the recent Meltdown vulnerability; this is also the motivation behind Linux's io_uring API, even if Redox would benefit to a greater extent. By using two separate queues, the _submission queue_, and the _completion queue_, the only overhead whatsoever of the syscall, is to increment two atomic variables (where the second is optional and only for process notification, albeit highly recommended), and write the submission queue entry to a flat shared memory region. Similarly, when a command completes, all that has to be done is the same: incrementing two counters and reading from shared memory. Since the channels are lock-free unless the queues are empty (when receiving) or full (when sending), this also allows two processes that run in parallel to serve each other's requests simultaneously in real-time by polling the rings.

Another reason why `io_uring` is to be considered, is due to the completion-based model for async I/O. While the readiness-based model works greatly for network drivers, where events are triggered by hardware (or rather, polled by the driver), on other hardware such as NVME disks, the hardware will not read from the disk by itself, but has to be started by the driver. The way `xhcid` handles this is by having two files allocated for each hardware endpoint: `ctl` and `data`. Not only is it much more complex and complicated both on for the driver and the driver user, but it also requires two separate syscalls, and thus causes more context switches than it needed to.

# Detailed design
[design]: #detailed-design

TODO

# Drawbacks
[drawbacks]: #drawbacks

Since Linux handles every syscall within ring 0, no extra communication needs to happen when dealing with addresses; the kernel can simply map a userspace address (if it is not already mapped), do its work, and then push a completion entry. Since the Redox `io_uring` functions between user processes through schemes, all buffers will have to be registered in order for the producer process to access those buffers. The current workaround here is to use a different `io_uring` with the kernel as the producer and allocating buffers there. Although this might not impose that of a greater overhead than simply letting the kernel deal with this during a syscall, for the most part it completely eliminates the ability to use the stack for buffers, since that would be a major security issue.

# Alternatives
[alternatives]: #alternatives

Even though it may not be strictly necessary aside from increased performance to use SPSC queues, one could instead simply introduce `EVENT_READ_COMPLETE` and `EVENT_WRITE_COMPLETE`, and let the kernel retain the grants of the buffers until events with the aforementioned flags have been sent, to indicate that the buffers are not longer needed by the scheme process.

# Unresolved questions
[unresolved]: #unresolved-questions

TODO
