- Feature Name: setattr
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Redox Issue: (leave this empty)

# Summary
[summary]: #summary

For many different kinds of named resources (files, pseudo-devices, file systems, etc.) there are reasons to set certain capabilities.
The setattr/getattr functionality is intended to provide a consistent protocol for all resources for communicating settings and
performing actions that are not part of the normal data open/read/write protocol.

setattr/getattr are two new system calls that will allow messaging
about named resources and file descriptors.
A message format is proposed that will cover all cases and allow future expansion.

# Motivation
[motivation]: #motivation

There are many situations where a named resource must be treated specially,
for example,
- a physical device or pseudo-device might have settings that need to be changed
- the mounting of a device might require special interaction between the device and the kernel
- an intermediary scheme like `contain` may want to intercept certain file operations like fpath,
and may need to register to do so.
- ioctl is the UNIX general purpose mechanism for setting device and pseudo-device parameters
- fcntl is used to set a variety of flags for files
- open may also include flags that change the behavior of files or resources
- there may be situations where the interaction between the scheme and the kernel needs to be modified, e.g. short-circuit writes (the kernel reports success to the client before informing the scheme of the operation)

UNIX and Linux have several mechanisms that have evolved to handle
communication of settings,
but there is no uniformity in how they are defined,
and there are many system calls to support them.

The proposal is to unify all these mechanisms under two system calls (it could also be done with one system call - see [Unresolved questions](#unresolved-questions)).

# Detailed design
[design]: #detailed-design

## System Calls

Two system calls are proposed, `setattr` and `getattr`.

- `setattr` sends a message with settings to be applied,
or an action to be invoked.
This is a one-way message and should not normally block.
- `getattr` sends a message to query the state of settings.
It can also be used to invoke an action and wait for a response.
This is a round-trip message and may block.
- In the case that an action is to be invoked (e.g. power off a device),
with the result read at a future time,
a `setattr` message can trigger the action,
but will include a `message identifier` field,
and `getattr` can be used to read the outcome of the action by indicating
the message identifier.

The signatures are
```
/// Return the size of inbuf on success,
/// -errno on failure
fn setattr<T>(fd: usize, protocol: &str, inbuf: T) -> isize;

/// Return the size of the outbuf content on success,
/// -errno on failure
fn getattr<Tin, Tout>(fd: usize,
    message_type: &[u8; 32],
    inbuf: Tin,
    outbuf: Tout) -> isize;
```

`fd` is an open file descriptor.

The `data` will vary. It can be binary or a text slice.
It will be read/written as a [u8; n] slice.
It is recommended that a serialization format be used,
e.g. `ron` with `#[skip_serializing_none]`,
which will allow optional fields to be left out.
In this case, `T` will be `&str`.

However, for well-defined struct formats such as `termios`,
it is reasonable to use existing data structure definitions.

The `message_type` is a str in the form `"target/protocol_format[.V.v]\0"`
It has a fixed size of 32 bytes and all unused bytes must be zero.

The `target` describes who should interpret the set/get message.
A proposed set of `target` identifiers is below.
- `fd` - the specfic resource that the `fd` represents.
- `scheme` - the scheme/daemon, for example enabling a feature
or requesting if a capability exists.
- `kernel` - the kernel (used by the client), to change how this `fd` is handled,
such as responding with
success for write operations immediately and not waiting
for the scheme to complete, or requesting to intercept fpath
operations.
- `kscheme` - the kernel (used by a scheme), to change how this scheme is handled,
such as increasing the maximum number of pending messages to the scheme.
- `client` - used internally by `relibc` to map to be processed before
making a system call.
- `namespace` - the namespace in which the file descriptor was opened,
purpose TBD.

`protocol` is a short (e.g. one word) identifier of the kind of
content of this message.
The name should include or imply the class of resource (pseudo-terminal,
graphics window, device type, etc.)
Possible examples could be `"termios"`, "`window`", `"filelock"`, `"pcicfg"`, etc.

`format` is a short identifier for the format of the data,
normally one of `"data"` for binary data
or `"ron"` for serialized data,
although other formats may be used if needed.

The version numbering `"V.v"` is an optional major/minor number pair
to specify the version of the data definition used.
It is not needed if there is only one data definition for
the specified protocol.

Note that the actual system call will include the size of
`inbuf` and the allocated size of `outbuf`.

Note that serialization/deserialization is the responsibility of
the client and the target.
`relibc` or `libredox` may provide an API that handles
serialization/deserialization transparently,
but that is outside the scope of this RFC.

## Scheme support

The various Scheme traits should provide a default implementation
that returns EINVAL.

Each scheme must implement handlers for its supported protocols.
Unsupported protocols will result in a return of EINVAL.

## Common protocols

Several protocols are common to all schemes.
- A `formats` protocol to request what protocols, formats and versions are supported.
The proposed format for this is that the client will send a list of
protocol/format/version it supports, in preferred order,
and the scheme will respond with the first match.

- An `fcntl` alternative TBD.

- The `kernel` protocol used to modify how the kernel treats
a particular file descriptor.
This protocol should be recognized by all schemes so the kernel
can inform them when the client has made a request.

## Scheme-specific protocols

Each scheme category can establish its own protocol for set/get operations.
The protocol name should uniquely identify the category (`fs`, `disk`, `tty`, etc.).
It should also identify the format and version of the protocol.
A major/minor numbering scheme is recommended,
with minor number bumps indicating new fields but no removals or
renaming, and major number bumps for removing or renaming fields,
or changing the size of a struct.

## ioctl/termios

This mechanism is intended as an alternative to `ioctl` and `termios`.
If desired, ioctl/termios messages can be supported, indicating the
IOCTL message type as the protocol.

The implmentation of `ioctl` and `termios` functions in `relibc`
will need to be updated to the new mechanism,
when the appropriate schemes are implemented.

# Drawbacks
[drawbacks]: #drawbacks

- This approach allows for future expansion of protocols and formats,
but this can lead to a mess of compatibility issues if proper care
is not taken when defining protocols.

- UNIX/Linux already has many system calls to support setting/getting
configuration for various types of resources.
We are diverging from the standard they set.

- Relibc will likely need to convert between protocols to provide
Linux compatible calls.

# Alternatives
[alternatives]: #alternatives

1. Use the Linux system calls.

2. Use Plan 9 style paths, which treat configurable
entities as a directory with a collection of read/write files
that represent settings and control.

# Unresolved questions
[unresolved]: #unresolved-questions

1. Should this be combined into a single `setgetattr` system call?
The messaging protocol could be adjusted to include a flag indicating
if it is a `set`, `get` or `action` message.

2. Should this operation be performed on the original `fd` or should
there be a `dup` or similar that provides an `fd` specifically for
set/get operations?

3. Should we include an optional `AttrBuf` as an argument to open,
to remove the need to perform an `fcntl` when special features are
desired? Do we still keep the flags argument as well, and to what extent do we move things into the AttrBuf?
