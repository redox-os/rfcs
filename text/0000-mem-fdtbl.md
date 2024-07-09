- Feature Name: mem_fdtbl
- Start Date: 2024-07-09
- RFC PR: (leave this empty)
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

# Detailed design
[design]: #detailed-design

Each context on Redox, currently has a file table, containing file descriptors,
which themselves contain counted references to _file descriptions_.
Hereinafter, _capabilities_ and _file descriptors_ are synonymous.

Another grant `Provider` type will be added for capability memory. Initialized
pages will be marked present but kernel-only, and will thus indicate that it is
a capability page. Userspace is responsible for allocating these grants, using
`/scheme/memory/capspace`. The kernel currently assumes a page is userspace iff
it is higher-half, but weakening this to "higher half implies kernel" is almost
certainly feasible.

The contents of these pages, will be an array of pointers to the file
descriptions (currently `Arc`s, but can be physical pointers to scheme-specific
file description slab). All syscalls currently taking `fd: usize`, will instead
take `cap: *const usize`, pointing to capability memory. Using non-capability
memory will result in `EBADF`.

On x86, and in general any architecture supporting SMAP semantics, this pointer
can be read in an exception-catching function with EFLAGS.AC cleared, since
these pages (unless kernel pages in the user half are introduced for other
purposes) are capability pages iff they are kernel pages.

Relibc will need to store another array (or bit array) containing metadata
associated with file _descriptors_, namely the `O_CLOEXEC` and `O_CLOFORK`
(added in POSIX 2024) flags. This array will be behind a lock, and will be used
to implement _open(3)_ and `F_DUPFD`. Non-legacy software will thus avoid the
locking cost of creating new file descriptors.

# Drawbacks
[drawbacks]: #drawbacks

This increases the complexity of the virtual memory system, since some
non-present pages will need to be specially treated. Additionally, looking up
the file description references from those tables, will likely require page
table traversal on at least some architectures, even though capabilities will
not require nearly as much memory as the rest of the address space, so they can
likely be limited to smaller page directories (or other levels).

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
pages readable by userspace (and obviously read-write for the kernel). That
would allow cross-checking whether sets of file descriptors intersect, which
would be useful for supplementary groups and file access control lists.
Otherwise, such checks may need syscalls.

Alternatively, it would be possible to use two contiguous pages per capability
frame, one readable by userspace and one by the kernel. In that case,
userspace-readable addresses can be unique identifiers rather than the same
physical addresses to the file descriptions. It would probably otherwise make
sense to use restartable sequences during the short critical sections when the
capability memory needs to be read, and stick to physical addresses.

Cyclic references will need to be avoided, since there are file descriptors
representing address spaces, which will themselves contain file descriptors, if
this RFC is implemented.

Perhaps the `Grant` abstraction could also similarly be eliminated, or at least
reduced. It would be possible to track the `Provider` of nonpresent pages using
the remaining page table entry bits, and using the PageInfo for present pages.
The only exception is file-backed mmaps. Maybe those could also be represented
using file descriptors?
