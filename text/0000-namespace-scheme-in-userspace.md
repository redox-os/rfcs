- Feature Name: namespace\_scheme\_in\_user\_space
- Start Date: 2025-09-17
- RFC PR: (leave this empty)
- Redox Issue: (leave this empty)

# Summary
[summary]: #summary

This RFC proposes creating a user-space namespace scheme, or nsmgr, at the path `/scheme/namespace` to manage all schemes and their open capability fds within a namespace. This change will make all resource access mediated by nsmgr, shifting namespace management from the kernel to user space.

# Motivation
[motivation]: #motivation

Currently, namespace management resides in the kernel, which adds overhead due to constant context switching between user space and the kernel. By moving nsmgr to user space, we can reduce this overhead. At the same time, this transition provides an opportunity to introduce a capability-based security mechanism for managing schemes within the namespace.

# Detailed design
[design]: #detailed-design

## namespace scheme(nsmgr)
A new scheme, nsmgr, will be created in user space at the path `/scheme/namespace`. This scheme will manage all schemes and their corresponding open capability fds within a namespace. nsmgr will be one of the first processes started by init. All open syscalls will be mediated by nsmgr.

## Creating a new namespace
Currently, new namespaces are created via the `mkns` syscall. After this change, the same operation can be performed by calling `dup(namespace_fd, names)`.

## Opening a scheme resources
When a process needs to open a resource, it will call openat with the namespace fd and path instead of calling open. For example, to open `/scheme/file/path/to/file`, the process would call `openat(namespace_fd, "/scheme/file/path/to/file", ...)`.

When nsmgr receives the `openat` syscall, it will check if the target scheme is registered in the namespace. If it is, nsmgr will forward the `openat` syscall to the scheme by calling openat with the corresponding open capability fd and the reference path. If the scheme is not registered, nsmgr will return an error.

The necessary change from `open` to `openat` would be implemented at the lowest level of libredox, so it would automatically apply to all programs in Redox.

## Collecting scheme name and open capability fd
There are three types of schemes that must provide their FDs to nsmgr.

- Kernel Schemes: Schemes like event, pipe, sys, etc., are started by the kernel. Their open capability fds will be passed from the kernel to the user-mode bootstrap. The bootstrap will then pass them and their names to nsmgr via the init process.
- Early Userspace Schemes: These schemes, such as procmgr and initfs, are started by the bootstrap process before nsmgr. Their open capability fds will be passed to the bootstrap, and the bootstrap will pass them and their names to nsmgr via the init process.
- Regular Userspace Schemes: These schemes are started by the init process after nsmgr is already running. When a scheme is started, it will send its own open capability fd to nsmgr using fd passing. nsmgr will then read the scheme name from the scheme's open capability fd and register the scheme in the namespace with the corresponding name and fd.

# Drawbacks
[drawbacks]: #drawbacks

The number of context switches may increase because all open operations are mediated by nsmgr.

# Alternatives
[alternatives]: #alternatives

- Maintain the Status Quo: We could keep namespace management in the kernel. This avoids the overhead of context switches but limits the adoption of user-space management and capability-based security.

# Unresolved questions
[unresolved]: #unresolved-questions

- Identity Propagation: How can the identity of the original requester be propagated to the scheme being accessed through nsmgr? This might be resolved in the short term by adding `filter_uid_gid` to the openat syscall.
- Permission Handling: How should we handle permissions for creating and deleting schemes and namespaces, both locally and globally?
- User Namespace Configuration: How should the namespace for each user be configured?
