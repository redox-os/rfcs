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
fn setattr<T>(fd: usize, inbuf: AttrBuf<T>) -> isize;

/// Return the size of outbuf on success,
/// -errno on failure
fn getattr<Tin, Tout>(fd: usize,
    inbuf: AttrBuf<Tin>,
    outbuf: mut AttrBuf<Tout>) -> isize;

/// Aligned to an 8 byte boundary
struct AttrBuf<T> {
    /// The size of the struct (maximum 4k)
    size: u16,
    /// Target can be the resource, scheme, or kernel
    target: [u8; 2],
    /// The protocol and version of the request (typically a string)
    protocol: [u8; 12],
    /// The settings or action message, aligned to an 8 byte boundary
    data: T,
}
```

`fd` is an open file descriptor.

The `data` will vary. It can be binary or a text slice,
and must be contiguous with the data structure ([u8; n], not String).
It is recommended that a serialization format be used,
e.g. `toml`, which will allow fields to be left blank.
However, for well-defined protocols such as `termios`,
it is reasonable to use existing data structure definitions.

The `target` describes who should interpret the set/get message.
A proposed set of identifiers is below.
- `fd` - the specfic resource that the `fd` represents.
- `sm` - the scheme/daemon, for example enabling a feature
or requesting if a capability exists.
- `ke` - the kernel (used by the client), to change how this `fd` is handled,
such as responding with
success for write operations immediately and not waiting
for the scheme to complete, or requesting to intercept fpath
operations.
- `ks` - the kernel (used by a scheme), to change how this scheme is handled,
such as increasing the maximum number of pending messages to the scheme.
- `re` - used internally by `relibc` to map to be processed before
making a system call.
- `ns` - the namespace in which the file descriptor was opened,
purpose TBD.

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

Use the Linux system calls.

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
