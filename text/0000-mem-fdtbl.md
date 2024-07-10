- Feature Name: mem_fdtbl
- Start Date: 2024-07-09
- RFC PR: https://gitlab.redox-os.org/redox-os/rfcs/-/merge_requests/22
- Redox Issue: (leave this empty)

# Summary
[summary]: #summary

This RFC proposes moving the file descriptor table objects, currently an
internal array on the kernel heap, into userspace virtual memory.

# Motivation
[motivation]: #motivation

The file table is currently stored as a resizable array, which all contexts
contain a reference to. This is inefficient. First, POSIX requires that `open`
return the lowest numbered available file descriptor, which along with
`F_DUPFD`, is currently implemented in the kernel. Although alternative
interfaces could bypass this limitation, all calls to `open` will likely
require an exclusive lock, which interestingly is one of the reasons Linux
implements [io_uring-based non-legacy file descriptor
tables](https://lwn.net/Articles/863071/).

Addressing file descriptors using memory directly will improve flexibility and
resource management, and partially unify the concepts of address spaces and
file descriptor (capability) spaces. That would simplify
pthread_create/fork/execve, and more importantly, allow relibc to use
_invisible_ file descriptors, that will not be noticed by the program.
Additionally, it would be possible to swap out and CoW-optimize capability
space, without implementing such features for the kernel half of the address
space.

# Detailed design
[design]: #detailed-design

Each context on Redox, currently has a file table, containing file descriptors,
which themselves contain counted references to _file descriptions_.
Hereinafter, _capabilities_ and _file descriptors_ are synonymous. This RFC
only affects how descriptors are handled; descriptions are currently internal
kernel objects, which are out of scope for this RFC. Capabilities are defined
as _unforgeable tokens of authority_; the _authority_ is the file description,
whereas the _unforgeable token_ is the file descriptor.

Another grant `Provider` type will be added for capability memory. Initialized
pages will be marked present but only accessible by the kernel, and will thus
indicate that it is a capability page. Userspace is responsible for allocating
these grants, using `/scheme/memory/capspace`. The kernel currently assumes a
page is userspace iff it is higher-half, but weakening this to "higher half
implies kernel" is almost certainly feasible.

The contents of these pages, will be an array of pointers to the file
descriptions (currently `Arc`s, but it would be equally valid for this to be
physical pointers to entries on scheme-specific file description slabs).
Although both of these optimizations would be invisible to userspace, both swap
and CoW logic will initially be disabled in the kernel. The latter is easier to
implement however, as it only affects writes to the capability arrays (when
duplicating or opening file descriptors).

On x86, and in general any architecture supporting SMAP semantics (including
RISC V), this pointer can be read in an exception-catching function or
instruction sequence with EFLAGS.AC cleared, since these pages (unless kernel
pages in the user half are introduced for other purposes) are capability pages
if and only if they are kernel pages. On architectures that do not support
this, manual page table walking is equally possible, and capability pages can
simply be marked "ignored" to the CPU, usually offering sufficient bits for it
to still store the relevant pointer.

## Affected syscalls

All syscalls currently taking `fd: usize`, will instead take `cap: *const
usize`, pointing to capability memory. Using non-capability memory will result
in `EBADF`. Syscalls that can currently return file descriptors, namely
`SYS_OPEN`, `SYS_DUP`, will instead take an additional argument
`destination_cap: *mut usize`. This also includes schemes attempting to receive
files sent to them using `SYS_SENDFD`.

## POSIX

Relibc will need to store a separate array (or bit array) containing metadata
associated with file _descriptors_, namely the `O_CLOEXEC` and `O_CLOFORK`
(added in POSIX 2024) flags. This array will be behind a lock, and will be used
to implement _open(3)_ and `F_DUPFD`. In addition, there will likely exist a
contiguous virtual memory region reserved for capabilities, that can be
populated at runtime.

Non-legacy software will thus avoid the locking cost of creating new file
descriptors, namely the requirement for the lowest-numbered file descriptor to
always be selected. Relibc can for example use other pages for internal file
descriptors, such as (future) capabilities for the current root namespace
(dirfd), working directory (dirfd), or process/thread fds.

### Fork and exec

POSIX 2024 has added O_CLOFORK. The behavior of fork is to retain all previous
non-O_CLOFORK file descriptors, and the behavior of exec is defined analogously
for O_CLOEXEC. This will be implemented in fork by duplicating the address
space as usual (including all capability pages), and then calling
SYS_CLOSE(cap_ptr) for each O_CLOFORK capability pointer in the child process
(possibly using a "close range" or "close indirect" syscall for performance).

Exec will close all O_CLOEXEC capabilities before entering the new program.
This does not change the O_CLOFORK/O_CLOEXEC fragmentation issue, since the
kept file descriptor numbers must be preserved in the new context. If all
capabilities in a page are freed, that page can safely be unmapped, possibly
automatically by the kernel (possibly limited when TLB shootdown is necessary).

# Drawbacks
[drawbacks]: #drawbacks

This increases the complexity of the virtual memory system, because on some
architectures, some non-present pages will need to be specially treated.
Additionally, looking up the file description references from those tables,
will likely require page table traversal on at least some architectures, even
though capabilities will not require nearly as much memory as the rest of the
address space, so they can likely be limited to smaller page directories (or
other levels).

# Alternatives
[alternatives]: #alternatives

One alternative to allow _invisible_ internal relibc fds, is to simply use two
file tables, or use another level of indirection for the POSIX file table.
Compared to the suggested approach, those would however be less efficient, and
would suffer from increased complexity, in managing resource limits, and
process API impls.

# Unresolved questions
[unresolved]: #unresolved-questions

KASLR is probably not worth aiming for in a µkernel, due to the significantly
smaller attack surface area, and possibility for a formal proof. So if KASLR is
a non-goal, and there are no other security drawbacks from revealing physical
memory to userspace, it would actually be possible to make these capability
pages user-accessible readonly (and obviously read-write for the kernel). That
would allow checking whether sets of file descriptors intersect, which would be
useful for scheme operations involving supplementary groups and file access
control lists. Otherwise, such checks would need to enter kernel mode, which
would be slower.

Allowing user access can either be done directly to the pages, or
alternatively, by using two contiguous pages per capability frame, one readable
by userspace and one by the kernel. In that case, userspace-readable addresses
can be unique identifiers rather than the same physical addresses to the file
descriptions, in situations where physical memory should not be revealed to
userspace. However, for use cases like CRIU, a possibly implementation might
use restartable sequences during the short critical sections when the
capability physical pointers need to be read, which would be interrupted if the
physical memory is swapped.

Cyclic references will need to be avoided, since there exist file descriptors
representing address spaces, which will themselves may contain file
descriptors, if this RFC is implemented.

Perhaps the `Grant` abstraction could also similarly be eliminated, or at least
reduced. It would be possible to track the `Provider` of nonpresent pages using
the remaining page table entry bits, and using the PageInfo for present pages.
The only exception is file-backed mmaps. Maybe those could also be represented
using file descriptors?
