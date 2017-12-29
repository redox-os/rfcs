- Feature Name: channels
- Start Date: 2017-12-29
- RFC PR: (leave this empty)
- Redox Issue: (leave this empty)

# Summary
[summary]: #summary

Design for a fast, bidirectional IPC mechanism in Redox.

# Motivation
[motivation]: #motivation

An easy to use, performant mechanism for IPC would be extremely useful. At the moment, the primary way of communicating between processes that are not parent-child is through creating schemes.

# Detailed design
[design]: #detailed-design

The current design consists of a single scheme, `chan:`, that provides an interface for creating, interfacing with, and closing channels. Each channel has one server process and one or more client processes.

In this documentation, `<name>` is used to represent a channel name. This can be anything, but must be known by all the participating processes.

Usage:
1. The server process opens `chan:/<name>` with the `O_CREAT` flag.
2. It listens for connections and duplicates the file descriptor to accept (similarly to the way that tcp is implemented in Redox)
3. The connection can be written to by both the client and server.

# Drawbacks
[drawbacks]: #drawbacks

There aren't any real drawbacks to this. It's a solid, easily extendible IPC mechanism.

# Alternatives
[alternatives]: #alternatives

- Named Pipes
- Actually implement UNIX Domain Sockets
- Do nothing

# Unresolved questions
[unresolved]: #unresolved-questions

- Should it implement sending file descriptors? If so, how?
- Should it use shared memory?
- Should processes be allowed to say that only processes with certain permissions can connect?
