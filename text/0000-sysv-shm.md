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

(ipcd)<->(sysvshmd)<->(relibc)<->(application)

sysvshmd keeps housekeeping data and implements the core functionalities which are:
1. Create a mapping between shm keys and reserved memory segments and manage them
2. Get shared memory segments from ipcd
3. Manage segment permissions
4. Manage segment lifecyles

relibc exposes functions to application and also mmap()s the segments to process memory space.


Overview of the workflow is that a process should call `shmget()` to get a shm id. Then the process calls `shmat()` with that id to get the shared memory mapped to its memory space.

One way to implement is to have a daemon track shms. So calling shmget open()s a control channel to our daemon, and shmat() dup()s another fd, which can be mapped to process's memory by relibc. Other functions like shmctl() can be communicated by writing special commnds through the control channel.

One point to consider is that in this design relibc MUST inform sysvshmd of some actions so the daemon can update its internal data structures. The ideal was to have the daemon handle all core functionalities.


# Drawbacks
[drawbacks]: #drawbacks

SysV interface is superseded by POSIX ipc/shm mechanisms. Therefore newer software should not rely on it.

# Alternatives
[alternatives]: #alternatives

The interface MUST be exposed by relibc to make dependent projects buildable. The alternative is to change those apps' source aka understand their logic and implement shared memory in some other way.
 
# Unresolved questions
[unresolved]: #unresolved-questions

Is that the way FDs are used in this design aligned with Redox design philosophy?
