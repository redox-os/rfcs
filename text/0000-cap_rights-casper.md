- Feature Name: cap_rights-casper
- Start Date: 2026-09-15
- RFC PR: (leave this empty)
- Redox Issue: (leave this empty)

# Summary
[summary]: #summary

This RFC proposes a Capsicum-inspired capability-based security framework and a Casper-like privilege separation model for Redox OS. The capability model introduces 64-bit fine-grained permission masks bound to file descriptors (FDs), divided into 16 generic bits verified by the kernel and 48 scheme-specific bits delegated to scheme providers via chained `Sqe`/`Cqe` (Submission/Completion Queue Entries). To support sandboxed processes that drop their namespace and CWD FDs, we propose an anonymous-scheme-based Casper architecture featuring a privileged broker daemon and isolated worker services (such as `fileargs`), enabling transactional and bulk resource delegation without requiring global namespace access.

# Motivation
[motivation]: #motivation

Currently in Redox OS, process sandboxing is primarily achieved via scheme namespaces. By dropping or isolating a namespace (e.g., closing the `namespace fd`), a process loses access to global scheme resolution, effectively mitigating ambient authority. 

The `contain` utility uses this mechanism by creating a dedicated namespace with a restricted set of visible schemes, mediating path resolution through an anonymous `contain` scheme that filters access according to a configuration file. However, `contain` only restricts which schemes and path prefixes a sandboxed application can open; once a descriptor is acquired, POSIX access flags cannot prevent fine-grained descriptor-level mutations (such as executing `std_fs_call` operations like `unlinkat` or `frename`, or changing file flags via `fcntl`). Furthermore, once an application completely drops its namespace to achieve full isolation, it loses the ability to dynamically open any additional resources.

FreeBSD addresses this architecture using Capsicum and the Casper daemon. Porting this security paradigm natively to Redox OS requires:
1. Granular capability restriction: Restricting standard descriptor rights (read, write, mmap, etc.) and domain-specific calls (such as filesystem link/unlink operations or custom scheme calls).
2. Efficient capability transfer: Seamlessly passing rights without breaking `schemev2` backward compatibility.
3. Privilege separation service (Casper): Providing a broker-worker IPC architecture that leverages Redox's scheme-based design, anonymous schemes, and native FD passing mechanisms, avoiding the complexity of Unix-domain socket passing (`SCM_RIGHTS`).

# Detailed design
[design]: #detailed-design

## 1. 64-Bit Capability Model

64-bit capability rights are added to every FD entry:

```

+-----------------------------------------------+---------------------------+
|         Scheme-Specific Rights (48 bits)      | Generic Rights (16 bits)  |
+-----------------------------------------------+---------------------------+
63                                            16 15                         0

```

### Generic Rights (16 bits)
- `CAP_MMAP`
- `CAP_MMAP_X`
- `CAP_READ`
- `CAP_WRITE`
- `CAP_SEEK`
- `CAP_CREATE`
- `CAP_LOOKUP`
- `CAP_FCNTL`
- `CAP_EVENT`
- `CAP_SYS_CALL` (Redox-specific: permission to issue `SYS_CALL`)
- `CAP_NAMED_DUP` (Redox-specific: permission to execute duplicate operations with non-empty buffers)

The kernel directly validates generic rights during corresponding system call dispatches.

### Scheme-Specific Rights (48 bits) and Capability Type
The upper 48 bits are reserved for scheme-defined semantics, such as `std_fs_call` operations (renaming, unlinking) or socket/device-specific commands.

Because the interpretation of these 48 bits depends on the underlying resource, each descriptor is also associated with a FD Type (e.g., file, process, net). This allows userland runtime libraries (`relibc`) to apply the correct bitmask when performing operations like `cap_rights_limit()`.

### Enforcement Flow via Sqe/Cqe Chaining
To remain compatible with `schemev2` without modifying the core memory layout of `Sqe` and `Cqe` packets, we utilize chaining flags:

```rust
// Flag indicating an adjacent Sqe/Cqe contains extended capability metadata
bitflags! {
    #[derive(Clone, Copy, Debug, Default)]
    pub struct SqeFlags: u8 {
        // ...
        // If this flag is set, the next SQE serves as extra information for this SQE.
        const NEXT_EXTRA = 1 << 2;
    }
}
bitflags! {
    #[derive(Clone, Copy, Debug, Default, Eq, PartialEq)]
    pub struct CqeFlags: u8 {

        // If this flag is set, the next CQE serves as extra information for this CQE.
        const NEXT_EXTRA = 1 << 7;
    }
}
```

1. Kernel to Scheme (`Sqe` Chaining): When an application invokes `SYS_CALL`, the kernel verifies `CAP_SYS_CALL`. It then formats the primary `Sqe` with `SqeFlags::NEXT_EXTRA` and places the full 64-bit mask in a subsequent chained `Sqe`. The scheme retrieves this entry and enforces its local capability policy.
2. Scheme to Kernel (`Cqe` Chaining): When a scheme returns a new resource reference or modifies rights upon `openat/dup`, it sets `CqeFlags::NEXT_EXTRA`. The adjacent `Cqe` carries the truncated capability rights and the fd type to the kernel. If the response is a scheme-local resource it is limited with the dir fd's full capabilities; otherwise, it is limited with the dir fd's generic capabilities.

### Setting and Getting Capability rights by syscall
`fcntl` with setting and getting capability rights commands allows a process to limit FD rights.
It takes a pointer to capability rights:
```rust
fcntl(fd, F_SETCAP, *const cap_rights);
fcntl(fd, F_GETCAP, *mut cap_rights);
```

---

## 2. Casper Architecture on Redox OS

The Casper framework is composed of three actors:

```
[Client Process]
   │
   │ 1. Spawns Casper Broker, acquires casper_fd via initial pipe
   │ 2. openat(casper_fd, "fileargs")
   v
[Casper Broker] (Privileged Namespace Daemon)
   │
   │ 3. Spawns Worker with base dirfd inherited
   v
[Worker (e.g., fileargs)] (Sandboxed Worker) <--- 4. Direct IPC via worker_fd
   │
   ├ 5. Receives limits & closes its own namespace fd
   └ 6. Validates path, openat(dirfd), truncates capabilities, returns FDs

```

### Components and Responsibilities

- Client: The application to be secured. It connects to the Casper Broker, receives service channels, sets policies, and drops its `namespace fd` and `CWD fd`.
- Casper Broker: A privileged daemon maintaining full namespace access (`namespace fd`). It operates as an anonymous scheme (not registered to namespaces). It spawns isolated workers on demand and passes direct communication channels back to the client.
- Worker Process (e.g., `fileargs`): An isolated instance of an anonymous scheme spawned by the Broker:
    - Receives a pre-opened base `dirfd` from the Broker.
    - Receives the client's whitelist (allowed paths, flag masks).
    - Handles client requests by resolving paths strictly relative to `dirfd`.



---

## 3. Worker Lifecycle and Operational Flow

### Step 1: Bootstrapping and Sandboxing

1. Connection: The Client creates a pipe, spawns the Casper Broker, and receives `casper_fd` over the pipe.
2. Worker Creation: The Client issues `openat(casper_fd, "fileargs")`. The Broker prepares a base `dirfd`, spawns the `fileargs` worker, passes the `dirfd` to it, and yields the worker's scheme handle (`worker_fd`) to the Client.
3. Policy Locking: The Client transmits its whitelist policies (allowed filenames, access flags) to the Worker.
4. Worker Self-Isolation: The Worker closes `namespace_fd`. The worker can no longer spawn processes.
5. Client Self-Isolation: Upon receiving policy confirmation from the Worker, the Client closes namespace fd and CWD fd. The Client is now fully sandboxed.

### Step 2: Resource Acquisition

#### Single Resource Request

1. The Client invokes `openat(worker_fd, "test.txt", O_RDONLY)`.
2. The Worker verifies the path against its whitelist and asserts that the flags conform to the policy.
3. The Worker executes `openat(dirfd, "test.txt", O_RDONLY)`.
4. The Worker truncates the 64-bit capabilities and returns the FD.

#### Bulk Resource Acquisition via `dup`
`dup`ing `worker_fd` with paths and access flags creates an isolated session within the Worker scheme.
The Client calls bulk fd receive on that fd. The Worker scheme prepares all fds and capability rights, then returns them.

### Step 3: Teardown

When the Client closes `worker_fd`, the Worker scheme event loop receives a close event. The Worker exits.


# Drawbacks

* Sqe/Cqe Complexity: Chaining submission and completion queue entries increases the state management logic in scheme event loops and kernel dispatchers.

# Alternatives

# Unresolved questions

1. Worker-per-Handle vs. Worker-per-Process: Should Casper Broker spawn a dedicated `fileargs` worker for each client session, or should it create a handle for it?
2. Place to store FD type: Should the in-kernel `FileDescriptor` struct store the FD type?
