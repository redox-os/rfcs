- Feature Name: scheme-path
- Start Date: 2024-01-17
- RFC PR: (leave this empty)
- Redox Issue: (leave this empty)

# Summary
[summary]: #summary

As discussed with jeremy_soller, bjorn3, jcake, rw_van, 4lDO2, et al.

To alleviate the difficulties created by having `scheme_name:` as the root of a scheme path, the proposed new format is `/scheme/scheme_name` where `scheme` is the literal word "scheme" and `scheme_name` is the name of the scheme.

There will be a transition period where `scheme_name:` will continue to be accepted as a scheme path.

# Motivation
[motivation]: #motivation

The current `scheme_name:` format creates multiple problems.

- Linux programs cannot detect `scheme_name:` as an absolute path and instead treat it as a relative path.

- Paths containing colon (`:`) are not properly parsed when they are part of e.g. `$PATH` or other colon-separated lists.

- Rust's `std::path` library does not easily integrate new `Prefix` variants. Adding a `Scheme` variant to `Prefix` causes some existing Rust programs and libraries to not compile due to missing `match` arms, and may create other problems. An OS-aware implementation of `std::path` is under consideration by the Rust team but it is not imminent.

# Detailed design
[design]: #detailed-design

## Behavior

1. The new format for schemes will be `/scheme/scheme_name` where `scheme` is the literal string "scheme" and `scheme_name` is the name of the scheme (resource or service).

2. Slash (`/`) will be the only recognized path separator (after the transition is complete). Once the transition to the new format is entirely complete, `:` will be an allowed character in filenames.

3. The `file` scheme will now **always** be the default scheme. If a path does not start with `/scheme` but it does start with `/`, it will have `/scheme/file` prefixed during canonicalization.

4. Relative paths are now allowed to back up out of a scheme. Previously, `scheme_name:/..` would resolve to `scheme_name:`. Now, `/scheme/scheme_name/..` resolves to `/scheme` and `/scheme/scheme_name/../..` resolves to `/` which is equivalent to `/scheme/file`. `/..` resolves to `/` as on Unix.

5. Scheme providers currently receive paths for `open` requests with the scheme component stripped off by the kernel, with the path argument to the `open` call assumed as an absolute path within the receiving scheme. No change is required here.

**NOTE:** The creation/mounting of new schemes is documented in a separate RFC.

6. To ease the transition to the new format, functionality that parses `scheme_name:`
should be guarded with two feature flags, `scheme_fmt_warn` and `scheme_fmt_compat`.

    - If `scheme_fmt_warn` is enabled, an error message is printed to stderr or logged to the console when the `scheme_name:` format is used.

    - If `scheme_fmt_compat` is enabled, the `scheme_name:` format continues to work as previously.

    - Disabling `scheme_fmt_compat` will cause the `scheme_name:` format to be treated as a regular file name or as a path relative to the current working directory.

    - Initially, `scheme_fmt_compat` will be enabled by default and `scheme_fmt_warn` will be disabled.

In the descriptions below,

- items marked **DEPENDENCY** are required to be implemented before some other change can be completed.

- items marked **USER VISIBLE** are higher priority as the user will see filenames using the old format.

## Disk Partitions

`RedoxFS` is the scheme provider for the `file:` scheme, which will be renamed `/scheme/file`. RedoxFS handles one disk partition per scheme. In a scenario where multiple disk partitions are to be mounted,

1. There will be a "root partition" managed by RedoxFS and named `/scheme/file`. Its content will normally be referred to starting from `/`, i.e. `/scheme/file/bin` will be referred to as `/bin`.

2. Additional partitions will be managed by other instances of RedoxFS, (or a single instance managing multiple schemes) with their own scheme names, e.g. a scheme name of "file.home" would appear as `/scheme/file.home` and identify the RedoxFS instance that manages the "home" partition.

3. There will be a symbolic link in the root partition that places `/scheme/file.home` at its "mount point". e.g. `/home` -> `/scheme/file.home`. This will allow a `user` folder on the `home` partition to have an apparent name of `/home/user` and a full name of `/scheme/file.home/user`.

Further details on the topic of disk partitions, mount points, `realpath` and `fpath` are outside the scope of this RFC.

## Code Changes

### redox-path

A new crate for handling of new and legacy paths is now available. `redox-path` includes `canonicalize_with_cwd`, which canonicalizes paths that use the new format. Legacy-format paths are not currently canonicalized as some schemes use paths that must be formatted very precisely.

### PATH variable

In future, the PATH environment variable will be converted to colon-separated format. In the short term, it is recommended that the PATH be limited to `/usr/bin` and all commands be copied or linked in that directory.

### Kernel Scheme Dispatch

The kernel "Scheme Dispatch" functionality must be changed first, including compatibility feature guards. Once that is done, all other changes can be done as time permits. See also the RFC regarding the "Namespace" scheme.

#### Current implementation

[syscall::fs::open](https://gitlab.redox-os.org/redox-os/kernel/-/blob/master/src/syscall/fs.rs?ref_type=heads#L44) is responsible for dispatching `open` calls to the appropriate scheme. It assumes paths are already in canonicalized form. 

It currently obtains the scheme name by splitting the name at the first `:`. It dispatches to the appropriate scheme by looking up the scheme in the current namespace. `:scheme` is naturally parsed into an empty scheme name `""` with a path of `scheme`. This is interpreted as a request to the `RootScheme` with a path argument of `scheme`.

This functionality will continue to be provided until we are ready to delete it, but with feature guards described below.

The path format for dispatch to the `RootScheme` is discussed in another RFC.

#### Changes required

1. `syscall::fs::open` will now also need to parse the new format by stripping the `/scheme/` literal and taking the string up to the next `/` as the scheme name. A path that does not begin with `/scheme/scheme_name` (or is not in the previous format, when `scheme_fmt_compat` is enabled) is considered an error.

2. The `scheme_fmt_warn` and `scheme_fmt_compat` feature guards should be implemented. Logging to the console will be done for old format scheme references.

**DEPENDENCY** This work must be completed before any other conversion work can proceed.

### Rust's std::path

Rust's `std::path` has some junk code in it due to *partial* rejection of past Redox pull requests. All the Redox-specific code should now be removed from `std::path` as Redox will now work with Linux-format paths. It is proposed that this work should be done once we are confident in the new `/scheme/scheme_name` format, but as soon as possible after that.

  - [here](https://github.com/rust-lang/rust/blob/master/library/std/src/path.rs#L303)

  - [here](https://github.com/rust-lang/rust/blob/master/library/std/src/path.rs#L2180)

  - [here](https://github.com/rust-lang/rust/blob/master/library/std/src/path.rs#L2673)

  - Possibly others

### Redox's std::path

All Redox-specific code in Redox's fork of Rust's `std::path` should be removed. Similar to the above changes, plus removal of the `Scheme` variant of Prefix.

### Camino

There is a [PR pending](https://github.com/camino-rs/camino/pull/88) for Camino, to adopt the `Scheme` Prefix variant. It should be closed.

### relibc Canonicalize

#### Current functionality

Currently, paths that use libc `open` are canonicalized to create a full path. The canonicalization uses the current working directory `CWD` as an additional information source during canonicalization.

- If the path given to `open` starts with `scheme_name:`, it is taken as an absolute path and is used unchanged.

- If the path starts with `/`, it is taken as absolute within the scheme of `CWD` (typically `file:`), and the scheme portion of `CWD` is prepended.

- If the path does not start with `scheme_name:` or `/`, it is prefixed with `CWD`.

#### Changes

1. The [canonicalize_with_cwd](https://gitlab.redox-os.org/redox-os/relibc/-/blob/master/src/platform/redox/path.rs?ref_type=heads#L16) function in `relibc` needs to be modified to accept `/scheme/scheme_name/path` as a new format, for specifying both the path and the CWD. This function will convert all paths to the new format prior to `syscall::open`.

2. This function will need to be changed to use `/scheme/file` as the scheme for an absolute path that does not contain a scheme.

3. The `scheme_fmt_warn` and `scheme_fmt_compat` feature guards should be implemented, with old format scheme references being warned on `stderr`.

4. An `assert` that the path must not follow the old format should be included when `scheme_fmt_compat` is disabled, at least for some initial period. This will trigger an abort, and when used with `RUST_BACKTRACE=full` can help determine the source of the problem.

5. After all old-format code has been removed, the `assert` should be removed, and paths starting with `name:` will be treated as allowed relative paths and handled normally.

### realpath

`realpath` is a `libc` function that on Linux takes a file descriptor and returns an absolute path that can be used to open the same file. This path will have all symbolic links resolved.

See [Unresolved questions](#unresolved-questions) regarding `realpath` issues.

On Redox, `realpath` uses the scheme service `fpath` to determine the path. `fpath` is expected to return a full pathname including the scheme. Current `fpath` implementations return paths using `scheme_name:path` format.

`realpath` should be modified as follows:

1. On return from `fpath`, `realpath` will strip `/scheme/file` from paths that contain it, so a path such as `/scheme/file/home` will be reported as `/home`.

2. The `scheme_fmt_warn` feature guard will enable `realpath` to check if the scheme format is `scheme_name:` and report to `stderr` if it is.

3. The `scheme_fmt_compat` feature guard will enable `realpath` to replace from `scheme_name:` with `/scheme/scheme_name/` (with appropriate bounds checking and overlapping copy). `file:` will be stripped from paths that contain it, replacing it with a leading `/` if needed.

### fpath implementations

Several scheme providers implement `fpath`, where the scheme-relative path is calculated for a given file descriptor, and the scheme prefix is inserted by the scheme provider.

1. `RedoxFS` is the main provider of `fpath` and should be updated as soon as possible to return a path in the new format.

2. We will need to do a survey of schemes to determine which ones provide `fpath`.

3. We will need to work through a prioritized list of `fpath` implementations to update them.

### redox-scheme

`redox-scheme` is the current best practice for creating user-space schemes. It should be updated to use the new scheme format. Scheme creation is discussed in the "Namespace Scheme" RFC.

**DEPENDENCY** - Is `redox-scheme` ready for all drivers to be migrated?

### redox-event

`redox-event` is the current best practice for using an event-based interface.

1. It should be updated to use the new scheme format.

2. A timer service should be added to `redox-event`, as many event subscribers use timers/timeouts directly, and in fact the use of timers is often the motivation for using the event scheme.

3. `epoll` requires the ability to gather all events that are available. This implies a need for a non-blocking check for events, e.g. `is_ready` or `maybe_next`.

**DEPENDENCY** - A timer service should be implemented as soon as possible. Is `redox-event` ready for all (most) event users to be migrated?

**DEPENDENCY** - A non-blocking check for events is needed by `epoll`.

### Scheme Clients

`event:`, `time:` and `file:` schemes are referenced in many places throughout the Redox code. They will all need to be modified.
1. Where `event:` and `time:` are referenced, consider using the `redox-event` crate (depends on the [timer service](#redox-event) being added to `redox-event`).

2. Where `file:` is used, should we simply delete the scheme reference (since `/scheme/file` is now always the default), or should we convert it to `/scheme/file/`?

### epoll

[epoll](https://gitlab.redox-os.org/redox-os/relibc/-/blob/master/src/platform/redox/epoll.rs?ref_type=heads#L56) in `relibc` uses the `event:` and `time:` schemes directly. It should be converted to use `redox-event` if possible. A non-blocking check for events may be required. If this is not feasible, `epoll` should be modified to use the new scheme format.

### libc: Scheme

The `libc:` scheme is implemented in `relibc` as way to resolve paths that depend on the process context, e.g. `/dev/tty`. `/dev/tty` is a symbolic link in the `file:` scheme that refers to `libc:tty`. The `libc:` scheme parses the full path used for `open` (`"libc:tty"` in this case). It needs to be updated to use the new format. The primary use of the `libc` scheme is via symbolic links in the [filesystem configs](#filesystem-configs).

On first glance, it looks like the `libc:` scheme can be updated with a [single change](https://gitlab.redox-os.org/redox-os/relibc/-/blob/master/src/platform/redox/libcscheme.rs?ref_type=heads#L6) to a constant. However, further investigation should be done.

**DEPENDENCY** This change should be done at the same time as updating the filesystem configs. Not many Redox applications use this functionality, so it is not urgent unless we are porting Linux TUI apps.

### RedoxFS

1. RedoxFS has its own [canonicalize](https://gitlab.redox-os.org/redox-os/redoxfs/-/blob/master/src/mount/redox/scheme.rs?ref_type=heads#L164) functionality that needs to be updated.

2. Old format symbolic links are supported. Because old format paths are not canonicalized, RedoxFS needs to convert old format links to the new format when resolving the links.

3. [fpath](https://gitlab.redox-os.org/redox-os/redoxfs/-/blob/master/src/mount/redox/scheme.rs?ref_type=heads#L634) will need to be revised.

### Contain

Contain makes extensive use of filename parsing, canonicalization and scheme references. The work to be done is not listed here as it is extensive.

**DEPENDENCY** The `desktop-contain` filesystem config should be updated at the same time as the updates to Contain.

### Ion

Ion has some code that strips or adds the `file:` prefix.

**User Visible** Although `file:` is removed, other scheme-prefixed paths are displayed to the user. This should be minimized by updates to `realpath`.

### Bash, Dash, Nushell, etc.

Shells that have Redox forks likely have `file:` prefix stripping to enable `glob` pattern expansion. This will need to be removed.

### Other libraries and crates

Any library or crate that has a Redox fork likely has some scheme-related code. They will need to be examined.

### Filesystem Configs

All filesystem configs (e.g. `desktop.toml`) need to be updated to the new format.

1. Symbolic links e.g. `/dev/tty` will need to be in the new format.

2. `desktop-contain.toml` will need to be revised to the new format.

## Documentation Changes

### Book

All documentation about schemes will need to be updated. There are several pages that discuss schemes.

### README files

A scan of the README files for each repo will need to be done, with updates as needed.

# Drawbacks
[drawbacks]: #drawbacks

1. Amount of Rework
2. Moving away from URI format
3. History of pushing for URI format to be accepted by the Rust community

# Alternatives
[alternatives]: #alternatives

## URI (Status Quo)

The status quo of URI-style `scheme_name:` paths seems to have reached its limit. Although it is a recognized path format in POSIX, it is not used for filesystem references. In practice URIs are converted by services into simple Unix paths by web applications that process URIs. 

Our use of this format is now impeding the porting of Linux applications and FOSS Rust applications.

## Plan 9 Paths

Plan 9 paths are similar to the proposed Redox scheme format, except that Plan 9 services are mostly present at the root of the filesystem, e.g. `/tcp`. Redox will have all services (schemes) logically under the `/scheme/` folder. This will help keep the filesystem root clean and allow for more *nix-like filenames.

## Other Path Formats

Using some Windows-compatible path format could alleviate some of the problems of having scheme-based names. [UNC](https://en.wikipedia.org/wiki/Path_(computing)#Universal_Naming_Convention) and the DeviceNS formats are supported by Rust `std::path`. However, there doesn't seem to be a benefit over using `/scheme/`.

POSIX allows that paths starting with `//` are "implementation defined" but other points in the POSIX specification state that a prefix of more than one slash is to be treated as a single slash.

# Unresolved questions
[unresolved]: #unresolved-questions

1. On Redox, the `fpath` scheme service is used as the mechanism to obtain a path for a file descriptor. However, it produces results that are not guaranteed to be correct. A future implementation of an `fpath`-like service will address the problems. Resolving this issue is outside the scope of this RFC.