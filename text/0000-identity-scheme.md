- Feature Name: identity\_scheme
- Start Date: 2025-09-17
- RFC PR: 
- Redox Issue: 


# Summary

This RFC proposes the introduction of a kernel-level `identity_scheme` to represent user and group identities (UIDs/GIDs) as file descriptors, referred to as `identity_fd`s. Instead of passing integer UID/GID values, system call requests will include an `identity_fd` representing the caller's identity. This `fd` will be held directly within the process's kernel `Context`, inaccessible from its userspace file table, ensuring that a process cannot tamper with its own identity. The kernel will be responsible for automatically attaching this `identity_fd` to all outgoing requests to other schemes, creating a secure and unforgeable method of identity propagation.


# Motivation

The current model for representing user and group identity in Redox OS follows the traditional UNIX approach, utilizing integer-based User IDs (UIDs) and Group IDs (GIDs). This identity information is maintained within the kernel's Context for each process and is propagated to schemes via the CallerCtx during a system call. Schemes rely on this integer representation as the basis for all permission and access control checks.

This traditional model presents two significant architectural challenges. First, integer IDs represent a form of ambient authority; they are data rather than unforgeable capabilities, making them susceptible to misuse or misinterpretation in complex scenarios involving untrusted schemes. Second, and more critically, this model breaks down in the presence of intermediary services like the planned namespace manager (nsmgr). When nsmgr forwards a request on behalf of a user process, the target scheme perceives the caller as nsmgr itself (typically with uid=0), completely obscuring the identity of the original user. This fundamentally undermines the security model, as the target scheme has no reliable way to perform access control checks against the true initiator of the request.

To address these shortcomings, this RFC proposes the introduction of a kernel-level identity_scheme. This scheme's purpose is to manage identity as a kernel resource, replacing integer UIDs/GIDs with unforgeable, capability-based file descriptors known as identity_fds. An identity_fd serves as a verifiable handle to a user's complete security context. This capability-based approach ensures that a user's identity can be securely and reliably propagated through any number of intermediaries, allowing the final target scheme to perform accurate, spoof-proof permission checks on the original caller.


# Detailed design

### 1\. The identity scheme and identity fd

A new scheme, identity scheme will be implemented within the kernel. This scheme is solely responsible for creating, managing, and interpreting identity resources. An `identity_fd` is a file descriptor that represents a resource managed by this scheme. This resource contains information equivalent to traditional UIDs, GIDs, and supplementary groups.

### 2\. Kernel `Context` Integration

The in-kernel `Context` struct for each process will be modified to hold an `identity_fd` directly, replacing the current `euid` and `egid` integer fields.

```rust
// A conceptual representation
pub struct Context {
    // ... existing fields
    pub identity_fd: Arc<RwLock<FileDescription>>,
    // ...
}
```

This `identity_fd` is not present in the process's file descriptor table, making it impossible for userspace to directly `close()`, `dup()`, or otherwise manipulate it. 

### 3\. Automatic Syscall Propagation

When a process executes any system call that results in a request to a scheme, the kernel will automatically:

1. Retrieve the `identity_fd` from the calling process's `Context`.
2. Attach a duplicate of this `identity_fd` to the request packet sent to the target scheme. 
    The kernel will insert the FD into the scheme's file table at UserScheme::read. The scheme can then read from this FD to determine the caller's UID/GID and perform permission checks.

### 4\. `procmgr` and Setting Identity FD
The identity_fd for a new process will be set up by procmgr during fork.
procmgr will dup() the caller's identity_fd (which is enclosed in the fork request) and use a privileged operation (sendfd to a special new_ctxt_fd) to install it into the child's Context.
`procmgr` will `dup()` a caller's `identity_fd` that enclosed to `fork request` and use a privileged operation (`sendfd` to a special `new_ctxt_fd`) to install it into the child's `Context`.

### 5\. Handling Intermediaries like `nsmgr`

This design explicitly handles the authority delegation problem.

1. A client process calls `openat` on a namespace fd, which sends a request to `nsmgr`. The kernel automatically attaches the client's `identity_fd`.
2.  `nsmgr` receives the request and the client's `identity_fd`. It resolves the path and prepares to call `openat` on the target scheme.
3. `nsmgr` makes a modified `openat` syscall, specifying the client's `identity_fd` as the identity to use for the request, rather than its own.
4. The kernel intercepts this. It inspects `nsmgr`'s own `identity_fd` (from its `Context`) to check for a special **"permission delegation" capability**.
5. If this capability is present, the kernel allows the operation and forwards the client's `identity_fd` to the target scheme. If not, the request is denied.
    This ensures that only trusted system services can act on behalf of other users.

### 6\. Privilege Management (`procmgr`, `setresugid`, and `sudo`)

- `setresugid`: This operation will be managed by the identity scheme. To change a process's identity, a process executes setresugid on its own `proc_fd` and sends the request to procmgr, just as it does with the current method. procmgr will then call setresugid on the `identity_fd` corresponding to the process handle.
  The identity_scheme will validate this request by checking if procmgr's own identity_fd possesses the setresugid execution capability. This highly privileged capability is granted to procmgr at boot time.
- `sudo`: The `sudo` scheme could function by having its own `identity_fd` with the `setresugid` capability. However, granting this capability is sensitive. An alternative is for the `sudo` scheme to continue interacting with `procmgr`, which remains the sole gatekeeper for identity changes.


# Drawbacks

- Performance Overhead: This design will significantly increase the number of system calls. Each request to a scheme now involves the kernel duplicating an `fd`, and the receiving scheme must accept it, read from it, and close it. This adds considerable overhead compared to passing simple integers. While optimizations may be possible later, the initial performance impact could be substantial.
- Migration Effort: This is a fundamental change to the OS's security architecture. Every scheme in the system must be updated to stop using integer UIDs/GIDs from `CallerCtx` and instead handle the incoming `identity_fd`.
- Complexity: The overall system complexity increases. Developers writing schemes need to understand this new mechanism.


# Alternatives

- Maintain the Status Quo: The issue with intermediaries like nsmgr can be addressed by extending `openat` with a callerctx mask.

# Unresolved questions

- User/Group Management: Where should the logic for creating, deleting, and modifying users/groups reside? Should `procmgr` be given capability `fd`s to manage this, or should a new dedicated userspace scheme be created?
- `fstat` and Inode Ownership: How can `fstat` return the owner of a file as an `identity_fd`? Associating a persistent `identity_fd` with every inode on a filesystem seems infeasible, as it would quickly lead to file descriptor exhaustion (`EMFILE`) for the filesystem scheme.
- `chmod` and `chown` Implementation: The precise mechanism for how `chmod` and `chown` would operate in this new model is still undetermined.
