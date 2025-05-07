- Feature Name: openat-setattr
- Start Date: 2024-11-22
- RFC PR: (leave this empty)
- Redox Issue: (leave this empty)

# Summary
[summary]: #summary

To get a file descriptor for `setattr` and `getattr`, the following call is proposed.
```
openat(resource_fd: usize, attr_group: &str, open_flags: usize, _creat_mode: usize);
```

- `resource_fd` is the file descriptor the device or service to be controlled by `setattr/getattr`
- `attr_group` is the UTF-8 name of a group of attributes that can be either read or set
- `open_flags` is either `O_GETATTR` or `O_GETATTR | O_SETATTR`
- `_creat_mode` is 0 and ignored for the purposes of this RFC

This requires a new system call, `openat`, which will initially just be for `setattr/getattr`,
but will eventually be expanded to provide functionality similar to POSIX `openat`.

`openat` will also replaced "named dup":
```
dup(resource_fd: usize, attr_group: &str)
```

# Motivation
[motivation]: #motivation

For terminology/clarity purposes:
- "fd" refers to a file descriptor, which is a user space handle for an open file.
An fd normally remains open after a `fork` or `exec` unless the flag `O_CLOEXEC` is set.
- "file description" refers to the kernel's view of the open file,
including the (singular, shared) file offset for read/write.
There may be one or more fd's, possibly in different processes, that refer to the same file description.
- "named dup" refers to the Redox system call `dup` with a non-empty string argument.

Currently, Redox uses "named dup" to open a file descriptor used for `ioctl` and other attribute-related operations,
with the `dup` call including a string that names the attribute set.

This approach implies that permissions to get or set attributes should be determined by the ability to open
the main resource file descriptor.

"named dup" is also used as a mechanism to provide named flags to `dup`,
to invoke various special actions or modify behavior to be performed during `dup`.

According to POSIX, old and new file descriptors from `dup` are interchangeable, other than the `FD_CLOEXEC` flag.
This is not always the case with Redox's "named dup".
It is particularly problematic given that `read` and `write` operations are used to get and set attributes
on a "named dup" fd, as the kernel will need to know when the new fd from "named dup" represents a shared file description
and when it represents a different target and therefore a different file description.
Using `openat` provides a new file descriptor that serves a different purpose from the original,
and is more in line with what would be expected by a Linux/POSIX programmer.

Adding `openat` for `setattr/getattr` allows permission to be granular and specific to individual attribute groupings,
and is potentially independent of permission to open the original resource.

`openat` is expected to serve a different purpose in future,
allowing sandboxing of programs within a directory.
However, it seems like the proposed use of `openat` is appropriate and a good fit,
and implementing it for this purpose can be leveraged later in RedoxFS and Contain for sandboxing.

# Detailed design
[design]: #detailed-design

A new system call, `openat` is to be added.
- A kernel implementation of the system call is required (`src/syscall/mod.rs`, `src/syscall/fs.rs`).
- A stub implementation of the handler is required (add `kopenat` to `src/scheme/mod.rs`),
returning `EOPNOTSUPP`.

- The API is to be implemented in `libredox`, `redox-syscall` and `relibc`.
    - Add `Fd::openat` and `call::openat` and `extern "C" redox_openat_v1` to `libredox`
    - Add `openat` to `src/call.rs` in `redox-syscall`
    - Add `redox_openat_v1` to `src/platform/redox/libredox.rs` in `relibc`
    - (Future) Provide a C API for `openat` in `src/header/fcntl.rs` in `relibc`

The default implementation in `redox-scheme` will initially be
to return `ENOENT` (for each scheme type).

Schemes that support `openat` and setting or getting of attributes will
provide an implementation of `openat` that checks the name of the attribute group
and determines if it is available for the resource identified by the 

(Future) All uses of "named dup" that are for setting or getting attributes are to be converted to `openat`.
This will require changes to several schemes to provide an implementation for `openat`
that will be very similar to their implementation of "named dup".
And the code that invokes "named dup" will need to be converted to `openat`.
This can be done in conjunction with implementing `setattr/getattr` for the attributes,
rather than reading or writing the attributes to the descriptor created by "named dup".
One specific example is the operation of `ioctl`.

There are some uses of "named dup" that are for purposes other than `setattr/getattr`,
so further analysis must be done before eliminating "named dup".
It may be more appropriate to use flags rather than a string to represent `dup` options.

# Drawbacks
[drawbacks]: #drawbacks

The current implementation of "named dup" works and can be used effectively.
The consequences of future growth in the use of "named dup" are hard to determine.
It may be unnecessary to make this change if the current mechanism can handle
the requirements:
- Ensure that permissions for "named dup" can be applied in the way that they would be for "openat",
or potentially apply the permissions during read/write operations on the resulting fd.
- Ensure that the kernel support for sharing or not sharing file descriptions,
based on the use of "named dup", is easy to maintain as the use of "named dup" grows.

# Alternatives
[alternatives]: #alternatives

1. `fcntl` with flags indicating that it should provide a file descriptor for `setattr/getattr`
would be a good alternative, and would not require a new system call,
but would further overload `fcntl`.
However, since the purpose of `fcntl` is to provide an overloaded "file control" function,
that's probably ok.
2. It is not a hard requirement that a new fd is needed for `setattr/getattr`.
They could be used on an already open file descriptor.
`setattr` and `getattr` are descriptive enough that they can identify the attribute group that is needed.
However, this poses a problem of permission granularity and limiting access to attributes.
3. The system call name for opening a `setattr/getattr` fd can be something other than `openat`.
`open_attr` or `openattr` would be a reasonable choice (`openattr` is maybe too close to `openat`).
As `openat` is primarily intended for sandboxing, it may be better to not overload it.

# Unresolved questions
[unresolved]: #unresolved-questions

1. Analysis of each use of "named dup" to determine what the best strategy is for
replacement, or whether status quo is best for that case.
2. For attributes that are common to all (or at least most) schemes,
what is the best way to share the common implementation?
    - In `redox-scheme`
    - A new crate `scheme-stats`, with every scheme needing to implement `openat` for the scheme
