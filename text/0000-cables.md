- Feature Name: cables
- Start Date: 01-12-2017
- RFC PR: (leave this empty)
- Redox Issue: (leave this empty)

# Summary
[summary]: #summary

A new scheme, comparable to pipes, but dedicated to sending/receiving file
descriptors across processes.

# Motivation
[motivation]: #motivation

Being able to send arbitrary file descriptors across processes (in particular,
but not limited to pipes), is important both as a first step towards high-level
IPC, and as a mechanism for sandboxing.

The objective of _cables_ is to provide a primitive for this purpose that
is both minimal and secure.

# Detailed design
[design]: #detailed-design


We introduce a new scheme, `cable:`. Files of scheme `cable`
behave essentially as regular pipes, with one major difference: each message
sent across a `cable` consist in exactly:
- a file descriptor; and
- a key, used to (de)multiplex messages.

The file descriptor is transparently duplicated during the communication. A
sending process _P_ passes as argument to `write` a file descriptor valid in
_P_. A receiving process _Q_ receives from `read` a file descriptor valid in
_Q_.

As instances of `cable` are represented in userspace as file descriptors, they
can themselves be sent between processes. This allows developers to create
cables (and anonymous pipes, etc.) that can be used to communicate between two
arbitrary processes _P_ and _Q_, provided there is a chain of `cable`s between
_P_ and _Q_ willing to initiate the communication.

## Example

The following is pseudo code, slightly simplified from the usual file API:

```rust
fn main() {
  let cable = File::open("cable:some/optional/path").unwrap();
  let pid = Process::fork().unwrap();
  if pid == 0 {
    // Child process.
    // For this simplified example, we assume that this process is
    // running unprivileged and has somehow dropped rights to open
    // files.

    // Receive a `File` object from `cable`.
    let mut fd : usize = 0;
    let mut key: [u8, 32] = [...];
    cable.read(&mut fd, &mut key).unwrap();
    assert_eq!(key, b"private key"); // The key is typically used to demultiplex messages≥
  } else {
    // Parent process.
    // For this example, it behaves as a sandbox.

    // Open a file on behalf of the other process.
    let file = File::open("file:user/potus/mail/privkey.txt").unwrap();

    // Now send the file to the other process.
    cable.write(fd, "private key").unwrap();
  }
}
```

## Technical details

If successful, a `write` on a `cable` will cause the file descriptor to be
duplicated. If the file descriptor cannot be duplicated, the call to `write`
fails. The call to `read` does not cause a second duplication.

When a `cable` is closed, any file descriptor that it may be keeping alive
(i.e. a `write` has happened but no `read`) is closed asynchronously.

# Drawbacks
[drawbacks]: #drawbacks

This proposal introduces a semi-magic scheme that must be implemented
in the kernel.

# Alternatives
[alternatives]: #alternatives

## POSIX sockets

POSIX offers a similar feature using sockets to send file descriptors.

This RFC aims to be much simpler than full sockets, while offering a sound base
to implement sockets on top if necessary.

## `dup_from`/`dup_export`

Previous experiments on the feature involved two syscalls `dup_from`/`dup_export`
that permitted importing/exporting a file descriptor from/to a process.

While these syscalls made clearer that the feature required kernel support,
their reliance on process ids was prone to race conditions both inside the
kernel (what should the kernel do if a process with this pid doesn't exist
anymore? when should cleanup take place?) and in userspace (where, depending
on the allocation strategy for pids, an attacker with the ability to use
`kill` and `fork` could impersonate as another process, intercepting or forging
file descriptors).

## Generic mechanism

Instead of hardcoding the format of message to one file descriptor + one buffer,
we could aim for something more powerful (and eventually extensible), with an
intended high-level API along the lines of:

```rust
// Send a file descriptor.
cable.send(key, &[File(fd)]).unwrap();
// Send some binary.
cable.send(key, &[Binary(buf)]).unwrap();
// Send several items at once.
cable.send(key, &[File(fd1), Binary(buf), Binary(buf2), File(fd2)]).unwrap();

// Possibly other cases. I can't think of any, though.

// Receive
if let Some(key, data) = cable.receive() {
  // `data` is `Box<[File|Binary]>`.
}
```

This would serve as a good base for a high-level (typed) IPC mechanism.

## Performing key matching in the scheme

An obvious variant would be to let the scheme implementation perform the
matching on key, instead of expecting just about every single client to
do it.

This choice was made because 1/ matching shouldn't be very hard; 2/ keeping
it out of the kernel is one less thing to worry about. This doesn't mean that
it's written in stone.

# Unresolved questions
[unresolved]: #unresolved-questions

There may be a way to minimize the implementation of `cable` into
a few syscalls. Whether this is a good idea is an open question.