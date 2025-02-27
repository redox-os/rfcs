- Feature Name: sysv-shm
- Start Date: 2025-01-29
- RFC PR: (leave this empty)
- Redox Issue: (leave this empty)

# Summary
[summary]: #summary

Implement SysV shm function family.
- [shmget()](https://man7.org/linux/man-pages/man2/shmget.2.html)
- [shmat()](https://man7.org/linux/man-pages/man2/shmat.2.html)
- [shmdt()](https://man7.org/linux/man-pages/man2/shmdt.2.html)
- [shmctl()](https://man7.org/linux/man-pages/man2/shmctl.2.html)

# Motivation
[motivation]: #motivation

Enables building of the projects that depend on it (like PostgreSQL).

# Detailed design
[design]: #detailed-design
## Overview

(ipcd)<->(sysvshmd)<->(relibc)<->(application)

sysvshmd keeps housekeeping data and implements the core functionalities which are:
1. Create a mapping between shm keys and reserved memory segments and manage them
2. Get shared memory segments from ipcd
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
### shmget(key_t key, size_t size, int shmflag)
Get the id of the new or already existing shared memory segment.

A dictionary is used to keep the links between assigned shm ids and allocated segments `HashMap<u64, usize>`.
The behavior of the function depends on the `shmflag` as follows:
IPC_CREAT: Create new segment. key argument is ignored.
IPC_EXCL: Is valid only when combined with IPC_CREAT. If the key exists, call fails.
SHM_HUGETLB: Ignored for now. Do we have huge pages concept in Redox?
SHM_HUGE_2MB, SHM_HUGE_1GB: See SHM_HUGETLB.
SHM_NORESERVE: TODO: Check if upstream shared memory service has this option if so it is relayed, otherwise ignored.

`shmflag` also contains Linux like file permissions ('rwxrwxrwx') to be set on the created segment.
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
Here we need to know the pid of the creator. It would be best if we could get the pid and gid of the process which has
opened the fd. Otherwise relibc should get and relay it which is breaking our promise to keep the daemon independent.

Size of the segment should get rounded up to a multiple of page size. Where can we get the memory page size in Redox?

Then we have:
```rust
pub struct SysvIpcPerm {
	key: KeyT,
	permission: u16,
	uid: u32, // TODO: these must be system defined types for uid, ...
	gid: u32,
	cuid: u32,
	cgid: u32,
	shm: File // Return value of open("shm://")
}
```
User and group ids plus permission are set according to the `shmflag`. So the uid and gid of the calling process should
be known from the fd to the daemon.

`shmget` return -1 on error and a shm id otherwise.

TODO: Errors (In Redox a special method is called to know the error code)
TODO: Limits, can be set in daemon config.
## Control Channel Protocol


# Drawbacks
[drawbacks]: #drawbacks

SysV interface is superseded by POSIX ipc/shm mechanisms. Therefore newer software should not rely on it.

# Alternatives
[alternatives]: #alternatives

The interface MUST be exposed by relibc to make dependent projects buildable. The alternative is to change those apps' source aka understand their logic and implement shared memory in some other way.
 
# Unresolved questions
[unresolved]: #unresolved-questions

Is that the way FDs are used in this design aligned with Redox design philosophy?
We share data structures between relibc and the daemon which defines the way commands are conveyed.
In the most simple form a command id following its arguments (defined by the command).
