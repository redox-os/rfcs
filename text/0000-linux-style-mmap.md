- Feature Name: linux-style-mmap
- Start Date: 2018-07-13
- RFC PR: (leave this empty)
- Redox Issue: (leave this empty)

# Summary
[summary]: #summary

Implement a new syscall, `mmap(addr: usize, len: usize, flags: usize, fd: usize, offset: usize) -> Result<usize>`. Deprecate the `fmap()` syscall, but not the `fmap()` scheme call. `flags` will include `PROT_{READ,WRITE,EXEC,NONE}` (perhaps renamed from PROT to MAP) and crucially `MAP_ANONYMOUS` for allocating memory at arbitrary locations.

# Motivation
[motivation]: #motivation

The current method of memory allocation in Redox is the `brk()` syscall.

`brk()` does not support:

* caller-specified memory locations
* allocation outside of `USER_HEAP_PML4`
* caller-specified page attributes - brk always provides read, write, no_exec

The current file mapping syscall is `fmap()`.

`fmap()` does not support:

* same three points as in `brk()`, except it uses `USER_GRANT_PML4`
* lazy file loading - loading pages only when accessed

`mmap()` would allow all of this.

This is needed for an `ld.so` style dynamic linker and (I believe) JIT compilers.

# Detailed design
[design]: #detailed-design

Arguments of `mmap()`:

* `addr: usize`
  * the virtual memory address to map to. if it is zero, the kernel picks an address from `USER_GRANT_PML4`, as is current behaviour on `fmap()`
* `len: usize`
  * how large, in bytes, the mapping will be.
* `flags: usize`
  * `PROT_{READ,WRITE,EXEC,NONE}` for page attributes
  * `MAP_ANONYMOUS` the mapping does not involve a file, it is simply a memory allocation. `fd` is ignored
  * `MAP_SHARED` maps to the same physical address as any identical mappings from other processes, otherwise, mapping is copy-on-write
  * `MAP_LAZY` does not actually call `fmap()` for the relevant scheme. pages are only loaded when they are accessed.
* `fd: usize`
  * if `flags` does not have `MAP_ANONYMOUS` set, the relevant file descriptor, otherwise ignored.
* `offset: usize`
  * the offset into the file to start the mapping at

If `MAP_LAZY` is not set, behaviour is identical to current `fmap()`.

If `MAP_LAZY` is set, the kernel allocates enough physical memory for the mapping and maps all of the pages not present. When they are accessed, either the kernel seeks and reads the data from the relevant scheme, or a new scheme call (or modified `read()`) is used to eliminate the need to seek.

The kernel should keep track of which files have been mapped MAP_SHARED using something like a `HashMap<(usize, usize), SharedMemory>` so future MAP_SHARED calls do actually end up shared.

Ideally, `ipcd` should allow `fmap()` so that programs can create arbitrary purpose shared memory buffers, however this would mean that `ipcd` itself can access these buffers.

# Drawbacks
[drawbacks]: #drawbacks

...effort?

# Alternatives
[alternatives]: #alternatives

As an alternative to `MAP_ANONYMOUS`, mapping descriptors of the `memory:` scheme has been proposed, to adhere to the everything-is-a-file philosophy, however to me this confuses the purpose of the memory scheme, as `fstat()` on it returns data about the whole system's memory, and this functionality relates to the VM space of a single process.

Mapping descriptors of `memory:` means at some point the descriptor would have to be opened; this could happen at each allocation, at allocator initialisation, or before the jump to userspace code, like stdin/out/err. However, this all seems too complicated.

# Unresolved questions
[unresolved]: #unresolved-questions

Should PROT_* and MAP_* be in separate arguments, like in POSIX?

Should MAP_SHARED be omitted, given that it allows simultaneous writes, or should userspace programs be trusted to agree on some kind of locking mechanism when using it?

Should we keep `brk()`?

Are user-specified mapping addresses needed?  
Semi-relatedly, will one process ever allocate more than its grant/heap capacity - if this is possible, user-specified mapping addresses are an alternative to the current system.