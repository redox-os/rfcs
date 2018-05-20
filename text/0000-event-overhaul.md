- Feature Name: event-overhaul
- Start Date: 2018-05-19
- RFC PR: (leave this empty)
- Redox Issue: (leave this empty)

# Summary
[summary]: #summary

Overhaul the kernel event system to support mio.

# Motivation
[motivation]: #motivation

The current kernel event system has one event queue per context, which is not
flexible enough to be used by mio.

# Detailed design
[design]: #detailed-design



# Drawbacks
[drawbacks]: #drawbacks

A number of programs will need to be updated to the new event system. The old
system will by necessity need to be removed.

# Alternatives
[alternatives]: #alternatives

The alternatives are to attempt to implement different event handling in a
userspace daemon.

# Unresolved questions
[unresolved]: #unresolved-questions

What parts of the design are still TBD?
