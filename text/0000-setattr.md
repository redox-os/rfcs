- Feature Name: setattr
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Redox Issue: (leave this empty)

# Summary
[summary]: #summary

For many different kinds of named resources
(files, pseudo-devices, file systems, etc.)
there are reasons to set or get certain attributes.
The setattr/getattr functionality is intended to provide
a consistent protocol for all resources for communicating settings
and performing actions that are not part of the normal
open/read/write operations.

setattr/getattr are two new system calls that will allow messaging
about named resources and file descriptors.
A message format is proposed that will cover
all cases and allow future expansion.

# Motivation
[motivation]: #motivation

There are many situations where a named resource must be treated specially,
for example,
- a physical device, pseudo-device or filesystem daemon
might have both standard and specialized settings and attributes
- the mounting of a device might require
special interaction between the device and the kernel
- an intermediary scheme like `contain` may want to intercept certain file operations like fpath,
and may need to register to do so.
- ioctl is the UNIX general purpose mechanism for setting device and pseudo-device parameters
- fcntl is used to set a variety of flags for files
- user-level open calls may include flags that change the behavior of files or resources
- there may be situations where the interaction between the scheme and the kernel needs to be modified,
e.g. short-circuit writes (the kernel reports success to the client before informing the scheme of the operation)

UNIX and Linux have several mechanisms that have evolved to handle
communication of settings,
but there is no uniformity in how they are defined,
and there are many system calls to support them.

The proposal is to unify all these mechanisms under two system calls.

# Detailed design
[design]: #detailed-design

## System Calls

Two system calls are proposed, `setattr` and `getattr`.

- `getattr` sends a message to query the state of settings.
`getattr` should not cause any significant side effects,
allowing `getattr` to be read-only-safe.
A result is provided
through a mutable buffer argument.
- `setattr` sends a message with settings to be applied,
or an action to be invoked.
`setattr` can return data through a mutable buffer.

The (non-public) system call signatures are
```
/// Return 0 on success,
/// -errno on failure
fn setattr<T>(
    fd: usize, 
    request: &[u8, 16],
    in_buf: &TIn,
    in_size: usize,
    out_buf: Option<&mut TOut>,
    out_size: usize
) -> isize;

/// Return the size of the out_buf content on success,
/// -errno on failure
fn getattr<Tin, Tout>(
    fd: usize,
    request: &[u8; 16],
    out_buf: &mut TOut,
    out_size: usize
) -> isize;
```

## libredox API

The public libredox API signatures are:

```
/// Return 0 on success,
/// -errno on failure
fn setattr<T>(
    fd: usize, 
    request: &[u8, 16],
    in_buf: &TIn,
    out_buf: Option<&mut TOut>
) -> Result<usize>;

/// Return the size of the out_buf content on success,
/// -errno on failure
fn getattr<Tin, Tout>(
    fd: usize,
    request: &[u8; 16],
    out_buf: &mut TOut
) -> Result<usize>;
```

## Details

`fd` is an open file descriptor.
The `fd` determines who should interpret the request.
In most cases, the file descriptor will be specifically created
for handling the requests (e.g. via `open_at` or named `dup`).

The `in_buf` and `out_buf` will vary. They can be binary or a text slice.
It is recommended that a serialization format be used,
e.g. `RON` with `#[skip_serializing_none]`,
which will allow optional fields to be left out.
In this case, `T` will be `str`.

However, for well-defined struct formats such as `termios`,
it is reasonable to use existing data structure definitions.

The `request` is a byte str in the form `b"TargetActionVersion"`
It has a fixed size of 16 bytes.
Unused bytes should be filled with 0 or whitespace.
`Target` is a prefix used as a uniqueness qualifier,
similar to the way different IOCTL numbers have a prefix
related to the device they apply to.
`Action` is the set of attributes to get or set,
or the action being requested,
such as seek on a physical device.
It also implies a particular format for the message,
for example, a message in RON format will have a
different `Action` identifier than a message in
binary format.

`Version` is a numeric version number,
allowing message formats to be refined or expanded
over time.

For `setattr`, if an `out_buf` is provided,
then on return,
it will typically contain the previous value,
prior to the new value being set,
and the copying of the previous value and
setting of the new value will typically have been done atomically.
Not all schemes and actions will support this behavior,
so it should be documented for each scheme and action.
A return value of 0 indicates that no data was copied
to `out_buf`.
If `out_buf` is None or `out_size` is 0, no data will be returned.

`setattr` and `getattr` must be interruptable,
such as by `SIGALRM` or `SIGINT`.
On interrupt, the return value will be `-EINTR`.

Note that serialization/deserialization is the responsibility of
the client and the target.
`relibc` or `libredox` may provide an API that handles
serialization/deserialization transparently,
but that is outside the scope of this RFC.

## Scheme support

The various Scheme traits in `redox-scheme`
should provide a default implementation
of `setattr` and `getattr` that returns EOPNOTSUPP.

Assuming the file descriptor will be obtained by
`openat` or through named `dup` (as "termios" is today),
the scheme daemon must implement a handler for the
path used for `setattr`/`getattr`,
either as its own handler or as part of the
regular handler.

Each supported action and its `request` byte string
must be documented,
including caveats and limitations.

Unsupported `request` values will cause a return of EINVAL.

## relibc interface

Other than as an implementation for POSIX and Linux services,
`setattr` and `getattr` will not be public interfaces
through relibc.

## ioctl/termios

This mechanism is intended as an alternative to `ioctl` and `termios`.
If desired, ioctl/termios messages can be supported, indicating the
IOCTL message type as the `action`.

The implementation of `ioctl` and `termios` functions in `relibc`
can be updated to the new mechanism,
when the appropriate schemes are implemented.

# Drawbacks
[drawbacks]: #drawbacks

- These system calls will be used by our `systemd` equivalent
to monitor and control daemons.
Therefore they must not block indefinitely if a daemon fails to respond.
The current solution assumes `getattr` and `setattr` can be run in a thread
specifically for that purpose,
and another thread can send a signal to interrupt a blocked `getattr` call
if too much time elapses.
This is not ideal.
A preferred solution for non-blocking is an `io_uring` style implementation TBD.

- This approach allows for future expansion of message formats,
but this can lead to compatibility issues if proper care
is not taken when defining formats and request codes.

- UNIX/Linux already has many system calls to support setting/getting
configuration for various types of resources.
We are diverging from the standard they set.

- relibc will likely need to convert between protocols to provide
POSIX/Linux compatible calls.

# Alternatives
[alternatives]: #alternatives

1. Use the Linux system calls.

2. Use Plan 9 style paths, which treat configurable
entities as a directory with a collection of read/write files
that represent settings and control.
We could further refine `open` or `openat` using 
`O_GETATTR` and and `O_SETATTR` flags,
to unambiguously indicate that this descriptor
is for getting and setting attributes only.
This would help with a refined permission structure
for capability-based security as well.

    a. There are potentially multiple sets of attributes to
    set/get per resource.
    Using a different path for each attribute set would allow a
    `read` or `write` of attributes to be very simple,
    as with the current `termios` implementation.
    However, implementing a non-blocking `read` would be
    require some further analysis.

    b. Using a single path to identify all attributes for the resource would
    require a two-step process when getting attributes,
    first to write a message indicating what attributes are desired,
    then reading them back when they are ready.
    This could easily support non-blocking `read` of attributes.

3. To provide a non-blocking paradigm,
we could add another system call, `readattr`.
The caller sends a request,
then monitors a file descriptor waiting for a response.
For example `getattr` would be a non-blocking send of a request,
and `readattr` would read the response when ready.

# Unresolved questions
[unresolved]: #unresolved-questions

1. Is it reasonable for our `systemd` alternative to use
threads to encapsulate `getattr` calls so they can be interrupted?