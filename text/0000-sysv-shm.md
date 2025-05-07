- Feature Name: sysv-shm
- Start Date: 2025-01-29
- RFC PR: (leave this empty)
- Redox Issue: (leave this empty)

# Summary
[summary]: #summary

Implement POSIX [shm](https://pubs.opengroup.org/onlinepubs/9799919799/functions/V2_chap02.html#tag_16_07)
function family.
- [shmget()](https://pubs.opengroup.org/onlinepubs/9799919799/functions/shmget.html)
- [shmat()](https://pubs.opengroup.org/onlinepubs/9799919799/functions/shmat.html)
- [shmdt()](https://pubs.opengroup.org/onlinepubs/9799919799/functions/shmdt.html)
- [shmctl()](https://pubs.opengroup.org/onlinepubs/9799919799/functions/shmctl.html)

# Motivation
[motivation]: #motivation

Enables building of the projects that depend on it (like PostgreSQL).

# Detailed design
[design]: #detailed-design
## Overview
This is an extension to `ipcd` to keep and return some housekeeping data for the memory segments
it allocates and assigns.

`ipcd` keeps list of ShmHandles as this:
```rust
pub struct ShmHandle {
    buffer: Option<MmapGuard>,
    refs: usize
}
```

For every shared memory segment we add these fields:
```rust
pub struct ShmIdDs {
	shm_perm: IpcPerm,    /* Ownership and permissions */
	shm_segsz: usize,   /* Size of segment (bytes) */
	shm_atime: Option<TimeSpec>,/* Last attach time */
	shm_dtime: Option<TimeSpec>, /* Last detach time */
	shm_ctime: Option<TimeSpec>, /* Creation time/time of last modification via shmctl() */
	shm_cpid: usize   ,/* PID of creator */
	shm_lpid: usize,   /* PID of last shmat(2)/shmdt(2) */
	shm_nattch: i64,  /* No. of current attaches */
}

```
We use `xopen` to get the pid, gid and uid of the caller instead of `open`.

The size of the segment should get rounded up to a multiple of the system memory page size. Which is
`syscall::PAGE_SIZE` for now. Then we have:
```rust
pub struct IpcPerm {
	key: KeyT,
	permission: u16,
	uid: u32, // TODO: these must be system defined types for uid, ...
	gid: u32,
	cuid: u32,
	cgid: u32,
	shm: File // Return value of open("/scheme/shm/")
}
```

A dictionary is used to keep the links between assigned shm ids and allocated segments `HashMap<u64, usize>`.

`shm` keeps housekeeping data and implements the core functionalities which are:
1. Creating a mapping between shm keys and reserved memory segments and managing them
2. Managing segments permissions
3. Manageing segments life cycle

relibc exposes the mentioned functions to application and also mmap()s the segments to process memory.

General workflow is as follows:
- A process calls `shmget()` to get a shm id.
- The process then calls `shmat()` with that id to get the shared memory segment mapped to its memory space.
- A call to `shmdt()` by the process reverses the previous step.
- The process can query info about the shm segments or change its permissions by calling `shmctl()`.


## API
### int shmget(key_t key, size_t size, int shmflg)
Get the id of the new or already existing shared memory segment.

If `key` is the value of IPC_PRIVATE it means the segment is not to be shared and IPC_CREAT is implied.

The behavior of the function depends on the `shmflg` as follows:

IPC_CREAT: Create a new segment. key argument is ignored.

IPC_EXCL: Is valid only when combined with IPC_CREAT. If the key exists, call fails.

SHM_HUGETLB: Ignored.

SHM_HUGE_2MB, SHM_HUGE_1GB: See SHM_HUGETLB.

SHM_NORESERVE: Ignored.

`shmflg` also contains Linux like file permissions ('rwxrwxrwx') to be set on the created segment.


`shmget` return '-1' on error and a shm id otherwise.

#### Errors
EACESS: Calling process does not have the required permission according to file permission bits.

EEXIST: When both IPC_EXCL and IPC_CREAT are specified but the shm segment already exists.

EINVAL: `size` is greater than the size of the requested segment.

ENOENT: No segment exists for the given key and IPC_CREAT was not specified.

ENOMEM: No memory could be allocated for segment overhead.

ENOSPC: No more ids available. Maximum number of ids have been assigned.

#### Limits
TODO: Can be set in daemon config.

### void* shmat(int shmid, cost void *shmaddr, int shmflg)
Brings a shared memory segment into calling process address space.

Returns the start address of the mapped memory or '-1' if it fails which means `errno` should be read.

If possible provided `shmaddr` will be used as the start address (depends on `ipcd` and `mmap`). 

A successful call updates `shm_atime`, `shm_lpid` and `shm_nattch` in the `ShmIdDs`. So the relibc
should inform the daemon of the memory map. **This is a bad design because daemon state gets depended on outside
code.**

The behavior of the function depends on the `shmflg` as follows:

SHM_RND: Round down the start address to the nearest multiple of `SHMLBA`. Should look if there is a
Redox counterpart for that constant.

SHM_EXEC: Allow the content of the segment to be executed (How does it relate to x permission? Guess x
is ignored)

SHM_RDONLY: Process should have read permission. Without it r/w is assumed and attachment fails if the
process lacks proper access.

SHM_REMAP: Replace current mappings in the address range (Applicable?)

### int shmdt(cost void *shmaddr)
Detaches the shared memory segment located at the address specified by `shmaddr`. In other words
reverses the actions of `shmat()`.

Returns '0' on success and '-1' on failure. In the latter case `errno` should be read to find the cause.

A successful call updates `shm_atime`, `shm_lpid` and `shm_nattch` in the `ShmIdDs`. 

### int shmctl(int shid, int op, struct shmid_ds *buf)

Has these functions:
- IPC_STAT, SHM_STAT, SHM_STAT_ANY: Get a copy of `ShmIdDs` in `buf`.
- IPC_SET: Change some fields of `ShmIdDs`.
- IPC_RMID: Mark the segment for destruction.
- IPC_INFO, SHM_INFO: Get global shm limits.

#### Errors
TODO

## Control Channel Protocol
TODO

# Drawbacks
[drawbacks]: #drawbacks

There is no drawback.

It may worth noting that there is another POSIX ipc/shm [real_time extension](https://pubs.opengroup.org/onlinepubs/007908799/xsh/shm_open.html) interface.

# Alternatives
[alternatives]: #alternatives

Alternative approach is to extend tempfs to keep required housekeeping data and have the apps mmap the
files. This is not a good idea because:
- tempfs is designed to handle apps' file system usage. Apps wanting to share memory
segments has nothing to do with file system. We want nothing to do with inodes here.
- ipcd already has the mechanisms of mmap implemented.
 
# Unresolved questions
[unresolved]: #unresolved-questions
Is extending the ipcd the best solution? Maybe tempfs is more suited as it has permissions checking
already implemented. Maybe all functionalities could merge into one daemon.
