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
- a key, typically used to (de)multiplex messages; and
- a set of file descriptor; and
- a binary value.


The file descriptors are transparently duplicated during the communication. A
sending process _P_ passes as argument to `write` files descriptors valid in
_P_. A receiving process _Q_ receives from `read` files descriptors valid in
_Q_.

As instances of `cable` are represented in userspace as file descriptors, they
can themselves be sent between processes. This allows developers to create
cables (and anonymous pipes, etc.) that can be used to communicate between two
arbitrary processes _P_ and _Q_, provided there is a chain of `cable`s between
_P_ and _Q_ willing to initiate the communication.

## Example

```rust
fn main() {
  let cable = Cable::open("cable:").unwrap(); // Empty path.
  let pid = Process::fork().unwrap();
  if pid == 0 {
    // Child process.
    // For this simplified example, we assume that this process is
    // running unprivileged and has somehow dropped rights to open
    // files.

    // Receive data from `cable`.
    if let Ok(message) = cable.read() {
      assert_eq!(message.files.len(), 1);
      assert_eq!(message.data.len(), 0);
      assert_eq!(message.key, b"private key"); // The key is typically used to demultiplex messages.
      // ...
    }
  } else {
    // Parent process.
    // For this example, it behaves as a sandbox.

    // Open a file on behalf of the other process.
    let file = File::open("file:user/potus/mail/privkey.txt").unwrap();

    // Now send the file to the other process.
    cable.write(CableMessage {
      files: [fd],
      key: b"private key",
      data: []
    }).unwrap();
  }
}
```

## Technical details

If successful, a `write` on a `cable` will cause the file descriptors to be
duplicated. If any of the file descriptors cannot be duplicated, the call to
`write` fails. The call to `read` does not cause a second duplication.

When a `cable` is closed, any file descriptor that it may be keeping alive
(i.e. a `write` has happened but no `read`) is closed asynchronously.

In the current document, cables MUST have an empty path. Followup RFCs may
introduce a semantics for non-empty path, but this is beyond the scope of this
document.

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

## Cables-based IPC

Cables could be used as a full-blown IPC mechanism. This is probably a bad idea
for performance, as it means that any IPC call would need to perfom syscalls.
A better IPC design would use shared memory, futex-style mostly-userspace
locking and only fallback to cables if file descriptors need to be sent.

For this reason, we do not plan to extend Cables further towards implementing
IPC.


## Performing key matching in the scheme

An obvious variant would be to let the scheme implementation perform the
matching on key, instead of expecting just about every single client to
do it.

We decide to NOT head in this direction because:

- matching in userspace isn't very hard, so matching in the kernel is probably
  not very useful;
- we believe that matching in kernel encourages placing numerous expensive
  syscalls, which could easily be avoided by keeping the matching in userspace.

# Unresolved questions
[unresolved]: #unresolved-questions

There may be a way to minimize the implementation of `cable` into
a few syscalls. Whether this is a good idea is an open question.