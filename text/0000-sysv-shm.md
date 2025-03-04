- Feature Name: sysv-shm
- Start Date: 2025-01-29
- RFC PR: (leave this empty)
- Redox Issue: (leave this empty)

# Summary
[summary]: #summary

Implement SysV [shm](https://pubs.opengroup.org/onlinepubs/9799919799/functions/V2_chap02.html#tag_16_07)
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

(ipcd)<->(sysvshmd)<->(relibc)<->(application)

sysvshmd keeps housekeeping data and implements the core functionalities which are:
1. Create a mapping between shm keys and reserved memory segments and manage them
2. Get shared memory segments from `ipcd`
3. Manage segment permissions
4. Manage segments life cycle

relibc exposes functions to application and also mmap()s the segments to process memory space.

General workflow is as follows:
- A process calls `shmget()` to get a shm id.
- The process calls `shmat()` with that id to get the shared memory mapped to its memory space.
- A call to 

One way to implement is to have a daemon track shms. So calling shmget open()s a control channel to our daemon, and shmat() dup()s another fd, which can be mapped to process's memory by relibc. Other functions like shmctl() can be communicated by writing special commands through the control channel.

One point to consider is that in this design relibc MUST inform sysvshmd of some actions so the daemon can update its internal data structures. The ideal was to have the daemon handle all core functionalities.

## API
### int shmget(key_t key, size_t size, int shmflg)
Get the id of the new or already existing shared memory segment.

If key is the value of IPC_PRIVATE it means the segment is not to be shared and IPC_CREAT is implied.

A dictionary is used to keep the links between assigned shm ids and allocated segments `HashMap<u64, usize>`.
The behavior of the function depends on the `shmflg` as follows:

IPC_CREAT: Create new segment. key argument is ignored.

IPC_EXCL: Is valid only when combined with IPC_CREAT. If the key exists, call fails.

SHM_HUGETLB: Ignored.

SHM_HUGE_2MB, SHM_HUGE_1GB: See SHM_HUGETLB.

SHM_NORESERVE: Ignored.

`shmflg` also contains Linux like file permissions ('rwxrwxrwx') to be set on the created segment.
For every shared memory segment we have a structure as follows:
```rust
pub struct SysvShmIdDs {
	shm_perm: SysvIpcPerm,    /* Ownership and permissions */
	shm_segsz: usize,   /* Size of segment (bytes) */
    shm_atime: Option<SystemTime>,/* Last attach time */
    shm_dtime: Option<SystemTime>, /* Last detach time */
    shm_ctime: Option<SystemTime>, /* Creation time/time of last modification via shmctl() */
	shm_cpid: usize   ,/* PID of creator */
	shm_lpid: usize,   /* PID of last shmat(2)/shmdt(2) */
	shm_nattch: i64,  /* No. of current attaches */
}

```
Here we need to know the pid of the creator. It would be best if we could get the pid and gid of
the process which has opened the fd. Otherwise relibc should get and relay it which is breaking our
promise to keep the daemon independent.

Size of the segment should get rounded up to a multiple of page size. Which is
`syscall::PAGE_SIZE` for now. Then we have:
```rust
pub struct SysvIpcPerm {
	key: KeyT,
	permission: u16,
	uid: u32, // TODO: these must be system defined types for uid, ...
	gid: u32,
	cuid: u32,
	cgid: u32,
	shm: File // Return value of open("/scheme/shm/")
}
```
User and group ids plus permission are set according to the `shmflg`. So the uid and gid of the calling
process should be known from the fd to the daemon.

`shmget` return '-1' on error and a shm id otherwise.

#### Errors
EACESS: Calling process does not have the required permission according to file permission bits.

EEXIST: When both IPC_EXCL and IPC_CREAT are specified but the shm segment already exists.

EINVAL: `size` is greater than the size of requested segment.

ENOENT: No segment exists for the given key and IPC_CREAT was not specified.

ENOMEM: No memory could be allocated for segment overhead.

ENOSPC: No more ids available. Maximum number of ids have been assigned.

#### Limits
TODO: Can be set in daemon config.

### void* shmat(int shmid, cost void *shmaddr, int shmflg)
Brings a shared memory segment into calling process address space.

Returns the start address of the mapped memory or '-1' if it fails which means `errno` should be read.

If possible provided `shmaddr` will be used as the start address (depends on `ipcd` and `mmap`). 

A successful call updates `shm_atime`, `shm_lpid` and `shm_nattch` in the `SysvShmIdDs`. So the relibc
should inform the daemon of the memory map. **This is a bad design because daemon state gets depended on outside
code.**

The behavior of the function depends on the `shmflg` as follows:

SHM_RND: Round down the start address to the nearest multiple of `SHMLBA`. Should look if there is a
Redox counterpart for that constant.

SHM_EXEC: Allows the content of the segment to be executed (How does it relate to x permission? Guess x
is ignored)

SHM_RDONLY: Process should have read permission. Without it r/w is assumed and attachment fails if the
process lacks proper access.

SHM_REMAP: Replace current mappings in the address range (Applicable?)

### int shmdt(cost void *shmaddr)
Detaches the shared memory segment located at the address specified by `shmaddr`. In other words
reverses the actions of `shmat()`.

Return '0' on success and '-1' on failure. In the latter case `errno` should be read to find the cause.

A successful call updates `shm_atime`, `shm_lpid` and `shm_nattch` in the `SysvShmIdDs`. 

### int shmctl(int shid, int op, struct shmid_ds *buf)

Has these functions:
- IPC_STAT, SHM_STAT, SHM_STAT_ANY: Get a copy of `SysvShmIdDs` in `buf`.
- IPC_SET: Change some fields of `SysvShmIdDs`.
- IPC_RMID: Mark the segment for destruction.
- IPC_INFO, SHM_INFO: Get global shm limits

#### Errors
TODO

## Control Channel Protocol
TODO

# Drawbacks
[drawbacks]: #drawbacks

There is no technical drawback just that we are reimplementing an obsolete interface as 
SysV interface is superseded by POSIX ipc/shm [mechanisms](https://pubs.opengroup.org/onlinepubs/007908799/xsh/shm_open.html).
Therefore newer software should not rely on it. Although it may still be useful.

# Alternatives
[alternatives]: #alternatives

The interface MUST be exposed by relibc to make dependent projects buildable. The alternative
is to change those apps' source aka understand their logic and implement shared memory in some
other way.
 
# Unresolved questions
[unresolved]: #unresolved-questions

Is that the way FDs are used in this design aligned with Redox design philosophy?
We share data structures between relibc and the daemon which defines the way commands are
conveyed.

In the most simple form a command id following its arguments (defined by the command).
