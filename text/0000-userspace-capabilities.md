- Feature Name: Userspace capabilities
- Start Date: 2016-11-07
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

This proposal introduces _capabilities_, a new mechanism for dealing with
permissions in Redox OS. This proposal is designed to minimize the required
kernel-level support, rather expecting that most of the feature is implemented
in userspace.

# Motivation
[motivation]: #motivation

Redox aims to be a highly safe and secure operating system. One of the key design
principles behind this safety and security is to keep the kernel small, by
delegating as much as possible to independent and clearly-separated schemes,
generally executed in userland. This design also has the benefit of giving more
freedom to scheme developers, who can adopt design and coding conventions that
match the expected feature set of individual schemes, rather than a uniform
design, as is the case in monolithic kernels.

This proposal proceeds further along in this direction, by delegating scheme
access control, which is a large subset of the security features expected from
an OS, to schemes themselves. We expect this change to have the following
implications:

- further reduce the role and size of the kernel;
- give further freedom to scheme developers, who may tailor security policies
  of each scheme independently, rather than relying upon a system-wide notion
  of what is permitted and what isn't;
- provide finer-grained security policies, which in turn will let application
  developers enforce better separation of concerns between components of their
  applications;
- clearer separation of concerns between the kernel and schemes.

# Detailed design
[design]: #detailed-design

## High-level overview.

This proposal introduces _capabilities_, a new mechanism for dealing with
permissions in Redox OS. This proposal is designed to minimize the required
kernel-level support, rather expecting that most of the feature is implemented
in userspace.

In this proposal, capabilities regulate access to urls. For instance,
a process _granted_ capability `file:tmp/*:rwx` has _read, write and execute_ (`rwx`)
access to all files in directory `tmp` (`tmp/*`). With this capability,
modifying `file:tmp/foo` will succeed. On the other hand, this capability
does not allow the process to read file `file:users/potus/mail/confidential.txt`.

By opposition to file-system centric permission mechanisms (such as typical
implementations of DAC or MAC), capabilities are purely dynamic and are attached
to contexts (processes and threads). Just as importantly, capabilities are neither
limited to `file:`,
nor hard-coded to a specific set of permissions with as `r`, `w` and `x`.
Indeed, any scheme can define and enforce its own grammar of capabilities.
For instance, a userspace `music:` scheme could expose a capability
`music:volume/max/90%+` that grants the ability to increase the sound volume past
some limit.


### Obtaining, dropping and granting capabilities.

As mentioned, capabilities are attached to contexts (processes and threads).

A context is created with the same capabilities as its
parent. To decrease the exposed security perimeter, contexts are encouraged to
drop capabilities as soon as the capability is not needed anymore.

Capabilities are _propagated_ between contexts sharing some knowledge of each
other. At any time, a context may request a capability from any other context.
If the requestee has itself been granted the capability, it can decide to
_grant_ the capability to the requesting context.

```rust
fn spy() {
  {
    // By default, we do not have access to this file. Attempting to open it
    // fails.
    let investigate = File::open("file:users/potus/mail/confidential.txt", ...)
      .unwrap_err();

    // Request a capability from a specific process. Note that capabilities are
    // exposed as regular files.
    let cap = File::dup_from(pid, "file:users/potus/mail/*.txt:r")
      .unwrap();

    // Since we have the permission to read the file, the following call will
    // succeed.
    let jackpot = File::open_at(cap, "file:users/potus/mail/confidential.txt", ...)
      .unwrap();
    let mut data = String::new();
    jackpot.read_to_string(&mut data)
      .unwrap();
  }

  // From this point, the capability has been dropped.
}
```

As shown in this snippet, capabilities are just files. Indeed, calls
`File::dup_from` and `File::open_at` work not only with capabilities
but with any file.

Let us consider this first call: `File::dup_from(pid, "file:users/potus/mail/*.txt:r")`
is a variant of `dup` that requests a file object from another process `pid`.
The full path of the file is `"file:users/potus/mail/*.txt:r"`. We will
see below how `pid` can provide this file.

The second call, `File::open_at(cap, "file:users/potus/mail/confidential.txt", ...)`
is requests the opening of `"file:users/potus/mail/confidential.txt"` and provides
a proof that the context has been granted capability `cap`.

The following snippet demonstrates a process granting a capability to another process

```rust
fn honeypot() {
  // First, let's obtain the capability from somewhere else.
  let cap = File::dup_from(pid_0, "file:users/potus/mail/*.txt:r")
    .unwrap();

  // We may now grant the capability to process `pid_2`.
  File::dup_to(pid_2, cap)
    .unwrap();

  // At this stage, `pid_2` can call `File::dup_from(pid, "file:users/potus/mail/confidential.txt:r")` and succeed.
}
```

Again, `File::dup_to` is not limited to capabilities. The OS keeps track of
open files (including capabilities) and how they are duped across processes.

Note that `dup_from`/`dup_to` do _not_ constitute a rendez-vous mechanism nor
do they provide primitives for discovering contexts or pids. Indeed,
if our snippet `spy` executes the call `File::dup_from` before our snippet
`honeypot` has executed its call to `File::dup_to`, the call to `File::dup_from`
will simply fail.

This is by design, to keep the specifications and implementation simple. Should
rendez-vous or discovery be needed, applications will need to use other features
of the system or libraries.

### Enforcing capabilities.

Before a capability can be propagated across contexts, it must first be created.
As a capability is simply a file handle, it can be created just as any other
file, by the scheme handler, when responding to a call to `Scheme::open`.
_Interpreting and enforcing a capability is the responsibility of
the scheme handler_.

Initially, the scheme handler is in charge of deciding whether to trust the process
requesting the capability – in the case of `file:`, for instance, very few
processes should be entrusted with general access to `file:*` but the
well-known `user:` process can be trusted with general access to `file:user/*:rwx`.

```rust
impl Scheme for MyScheme {
  /// Argument `source` is the file descriptor passed to `open_at`. It serves
  /// as a proof that the calling process has access to a given file.
  fn open(&self, path: &[u8], source: Fd) -> Result<Fd> {
    let should_accept = // ... Is `source` a proof that the caller has access to `path`?
    if should_accept {
      let result = self.new_fd(path, Some(source)); // Store security information on `result`
      return Ok(result)
    } else {
      Err(Error::new(EACCES))
    }
  }
}
```

As is already the case, further calls upon a file descriptor can then take advantage of the
security information recorded during `open`:

```rust
impl Scheme for MyScheme {
  fn read(&self, file: Fd, buf: &mut [u8]) -> Result<Fd> {
    if self.get_security(file)
      .unwrap()
      .permits_read() {
      // Actually read file descriptor.
      return Ok(...)
    }

    // No capability grants the ability to perform operation, that's a EACCES
    return Err(Error::new(EACCES));
  }

  // ...
}
```

Note that the tracking of `capabilities` involves no kernel magic. Indeed,
since the distribution of capabilities requires solely the use of `dup_from`,
`dup_to` and `close`, which are themselves implemented as part of the scheme,
this tracking runs entirely in the scheme process.

Also note that `open_at` can serve not only to access concrete files but also
to request weaker versions of a capability. Again, this is handled entirely
in the scheme process.

### Anchoring the chain of capabilities

In this model, creating a file (capability or otherwise) on a scheme already
requires access to a file descriptor of the same scheme. Consequently, we need
a first file. This first file is obtained during initialization, by the root
protocol, on behalf of the initializer process.

```rust
impl Scheme for MyScheme {
  /// Called during initialization by the root protocol on behalf of the
  /// initializer process. We guarantee that this method can only be called
  /// once and only before the first call to `open`.
  fn init(&self) -> Result<Fd> {
    /// Create and store a file descriptor with universal access to this scheme.
    let fd = self.new_fd(b"", None);
    return Ok(fd)
  }
}
```

The scheme guarantees that this file descriptor has universal access to the
scheme. In turn, the initializer process is in charge of requesting and
distributing weaker capabilities as needed to the processes it spawns.


### Revoking capabilities

As we have already seen, a process can close a capability exactly as it
can close any other file. Sometimes, however, this is not sufficient.

For instance, consider a file residing on a usb key, or a capability granting
read access to that file. Usb keys share a property with floppy disks of old:
they can be removed from the system. Therefore, any file descriptor pointing
towards the disk must be revoked from the entire system – whether the file
descriptor points to an actual file or is simply a capability granting access
to a subset of the files on the usb key.

In some cases, though, we do not want to revoke all the descriptors for
a given file. Consider, for instance, the case of a malicious application.
Once malice has been detected, the process must be killed immediately. Moreover,
any capability that has been granted to the process must be immediately revoked
not only from the (now dead) process but also from any other process to which
the attacker may have leaked this capability.


```rust
fn rat_trap(pid, cap) {
  // This function is called because we have somehow determined that `pid` (with
  // whom we have shared `cap`) has been compromised.
  File::undup_to(pid, cap).unwrap();

  // This call revokes `cap` from `pid`, but also from any context which has
  // been granted `cap` from `pid`, recursively. However, if a context has
  // received `cap` from other, non-compromised, sources, it is not affected.
}
```

## Detailed changes covered by this proposal.

While the main goal of this proposal is to provide capabilities, the only
changes needed are the ability to dup (and undup) file descriptors across contexts and
to track down these operations.

### Syscalls

Implementing the feature requires the following changes to syscalls, as follows:

1. Replace `SYS_OPEN(path, path_len, flags)` with `SYS_OPEN_AT(path, path_len, fd)`,
  where `fd` is a file descriptor valid for the calling context. This
  file descriptor is substituted with a file-descriptor-at-origin during the dispatch
  to `Scheme::open`, as it currently is with most file-related syscalls.
2. New syscall `SYS_DUP_TO(pid, fd)`, dispatched to `Scheme::dup_to`. Informs the
  scheme that `pid` may now call `SYS_DUP_FROM` and obtain this `fd`.
3. New syscall `SYS_DUP_FROM(pid, path) -> Result<fd>`, dispatched to `Scheme::dup_from`.
  Requests a new file descriptor from the scheme, provided a file descriptor has
  already been registered with `dup_to` by context `pid` and for the current
  context id.
4. New syscall `SYS_UNDUP_TO(pid, fd)` (could possibly, dispatched to `Scheme::undup_to`.
  Request revocation of all capabilities represented by `fd` from `pid` and any process
  to which `pid` has granted the capability, recursively. Could possibly
  be merged with a powered-up `SYS_CLOSE`.
5. Optionally, remove syscalls `SYS)GET_GID`, `SYS_GET_EGID`, `SYS_GET_UID`, `SYS_GET_EUID`.

Note that none of these syscalls require any intelligence from the kernel.
After the usual checks, the call is simply routed towards the scheme implementation.

Also note that `SYS_DUP_TO` and `SYS_DUP_FROM`, taken together, are essentially
to `SYS_DUP`, so it might be possible 

### Contexts

Optionally, remove `uid` and `gid` from the definition of a context.

### Scheme-level (generally userland)

#### New methods

The definition of trait `Scheme` is extended as follows:

1. A new method `fn init(&self) -> Result<Fd>`, designed to return an all-powerful file
  descriptor.
2. Change method `open` to `fn open(&self, path: &[u8], source: Fd) -> Result<Fd>`.
  We remove arguments `uid` and `gid` (which are replaced by finer-grained capability management)
  and `flags` (which is now part of `path`).
3. A new method `fn dup_to(&self, pid: Pid, fd: Fd) -> Result<()>`.
4. A new method `fn dup_from(&self, pid: Pid, path: &[u8]) -> Result<Fd>`.
5. A new method `fn undup_to(&self, pid: Pid, fd: Fd) -> Result<()>`.

Other methods are untouched.

### Libstd

We expose the new features as part of struct `File`, through the following operations:

```rust
impl File {
  pub fn open_at(path: Path, source: &File) -> Result<File>;
  pub fn dup_to(&self, pid: Pid) -> Result<()>;
  pub fn undup_to(&self, pid: Pid) -> Result<()>;
  pub fn dup_from(pid: Pid, path: Path) -> Result<File>;
}
```

### Specific cases

#### DoS

A malicious or buggy application may DoS a scheme by attempting to `File::dup_to`
too many file descriptors. We expect that this is no worse than the current
situation in which a malicious or buggy application can open countless file
descriptors. However, the number of active `dup_to` may need to be capped.

#### Receiving the same file descriptor from several places

A malicious or buggy application could attempt to trick the system into killing
legitimate file descriptors by calling `dup_to(pid, fd)` towards a process `pid`
that already has access to `fd`, counting on a watchdog to accidentally revoke
a legitimate use of `fd`.

To avoid this situation, we need to refcount the number of sources for a file
descriptor in a given context.

#### `fork`/`exec`

Calling `fork` creates a process that inherits all file descriptors, including
capabilities. The scheme receives `dup` calls from the parent process but no
`dup_from`/`dup_to`.

Calling `exec` closes all these file descriptors, including capabilities.

# Drawbacks
[drawbacks]: #drawbacks

## Compatibility

This proposal requires changes that will undoubtedly break most existing
schemes and applications.

Also, one of the secondary objectives of Redox is to simplify porting of Linux
applications. It is not clear that this proposal will support this secondary
objectives.

## Security of schemes

While decentralizing security means that we can better tailor security
properties of individual schemes or applications, it also means that security
audits of machines is more complex, as it requires examining each scheme and
each application.

# Alternatives
[alternatives]: #alternatives

## Capabilities as special objects

An early design of this proposal involved capabilities as a new form of objects
comparable to files, tracked by the kernel.

This proposal has not been submitted
as it made the kernel more complex, rather than simpler.

## Intersections of capabilities

An early design of this proposal involved capabilities as files that magically
gave abilities to a context. Attempts to open `file:users/potus/mail/confidential.txt`
would succeed if the context had in its table of file descriptors a descriptor to,
for instance, `file:users/potus/mail/confidential.txt:+r` and fail after this
file descriptor had been closed.

This proposal has not been submitted for two reasons:
- it made applications harder to check;
- it required lots of boilerplate in the implementation of `Scheme`.

# Unresolved questions
[unresolved]: #unresolved-questions

This proposal implies the existence of a controller process in charge of spawning
schemes and delegating capabilities. Specifications of the controller process are
not covered by this proposal.

At this stage, it is not clear whether file descriptors can easily be shared
between threads within a single process.