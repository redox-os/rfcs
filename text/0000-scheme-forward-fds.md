- Feature Name: scheme_forward_fds
- Start Date: 2021-06-12
- RFC PR: https://gitlab.redox-os.org/redox-os/rfcs/-/merge_requests/17
- Redox Issue: (leave this empty)

# Summary
[summary]: #summary

This feature allows schemes to forward file descriptors local _to the process
that is responsible for handling that scheme_. In other words, rather than
giving the kernel a word-size integer representing a scheme-local file
descriptor number, it can instead give the kernel a fully valid file descriptor
in its _process file descriptor namespace_, that might have originated from a
completely different scheme in the first place, thus _forwarding_ file
descriptors.

# Motivation
[motivation]: #motivation

Microkernels are intrinsically meant to be as minimal as possible, and Redox
schemes serve as a useful IPC primitive. The ability to compose schemes, on the
other hand, is limited by the latency delay of chaining scheme calls, if scheme
A needs to communicate with scheme B, and in turn, scheme C, etc. Sometimes,
the schemes in such chains need to access the inner data, but in many cases,
filtering e.g. directory entries and enforcing access checks is sufficient.

A good example is the `irq:` scheme. IRQ handling must be be _fast_, and as
low-latency as possible. Ideally the only thing before the actual code, should
be to mask the interrupt and directly switch to the handler. However, the
current IRQ scheme also handles IRQ allocation for drivers, which adds
unnecessary kernel code, simply because adding a wrapper scheme would slow down
IRQ handling. This would also apply for hypothetical I/O port and MSR
scheme-based interfaces.

Additionally, The `proc:` scheme can probably at some point in the future, be
divided into a userspace and kernel part, providing convenience APIs such as
`proc:PID/memory` while forwarding performance-critical APIs such as ptrace
pipes.

With the feature described by this RFC, resource allocation can be moved to a
userspace scheme, and giving that wrapper scheme handler near-full access to
the kernel scheme. Once opened by a client, it will have a file descriptor that
directly points to the kernel IRQ scheme.

The same holds logic holds for userspace. Disk partitions are currently managed
by the disk drivers, mainly to maintain flexibility and performance, as a
middleman scheme would add latency to every read or write call. And while it
may be reasonable to add this particular duplicate functionality to multiple
drivers, this feature would enable disk drivers to simply add track allowed
ranges in each file descriptor, and then have partition daemons provide
forwarding schemes.

Most importantly, __a chroot tool implemented using schemes and namespaces,
would also become zero-cost for data, even if it may have to filter metadata
access, i.e. directory structures.__ Scheme file forwarding is not
_sufficient_, as some other interfaces such as `openat` may be required first,
but it is _necessary_ for allowing fast chroots.

# Detailed design
[design]: #detailed-design

On Redox, each file description is associated with a scheme ID and a
scheme-provided identifier, and is refcounted. Hence, schemes respond to
`SYS_OPEN` and `SYS_DUP` calls by returning that identifier, causing the kernel
to insert a new file descriptor, pointing to the newly-created file
description. A new file descriptor is thus _created_ in the process, with the
scheme being responsible for handling all subsequent system calls operating on
that file descriptor.

Alternatively, this new feature additionally allows schemes to _forward_ an
existing file description, as opposed to creating a new file description.

Scemes normally respond to kernel calls by reusing the same packet it received
from the kernel earlier, keeping all fields (even though every field except
`id` is ignored), and setting `a` to the return value of that scheme message.
The only exception is when triggerings events, where it instead sets the `id`
field to zero, and sets `a` to `SYS_FEVENT`, `b` to the scheme-provided number,
and `c` to the event flags.

It would be possible to add a new syscall number exclusively used as a scheme
response code, similar to `SYS_FEVENT` messages. However, since the response
needs to be associated with `packet.id`, which as a `u64` is not guaranteed to
fit within a `usize`, the current implementation introduces a new error code,
`ESKMSG`. Paired with `packet.b` = `SKMSG_FRETURNFD` and `packet.c` =
scheme-owned fd, the kernel will move that file descriptor out of the scheme's
file table, and then transparently store that file description into the
caller's file table, regardless of whether the file descriptor was created or
forwarded.

# Drawbacks
[drawbacks]: #drawbacks

The trivial drawback is that it adds complexity to the kernel, although in this
case it is relatively minor. The question is rather whether we actually do want
the functionality of being able to forward file descriptors that previously
originated from different schemes.

# Alternatives
[alternatives]: #alternatives

The obvious alternative would be to simply allow schemes to communicate with
the caller requesting a file descriptor from `SYS_OPEN` or `SYS_DUP`, via
previously suggested "fd channels", such as `cable:` or `sendfd:`. On the other
hand, this would require the client to implement different logic for this, thus
becoming less flexible and most importantly no longer transparent.

That said, scheme file forwarding is not incompatible with such file descriptor
channels, and when sending or receiving large sets of file descriptors, scheme
file forwarding may incur unnecessary performance overhead, in comparison.

# Unresolved questions
[unresolved]: #unresolved-questions

The primary unresolved question, is what syscall interface schemes should use
when forwarding. The alternative to the current implementation's "scheme-kernel
messages" (`ESKMSG`, `SKMSG_*`, and the rest of the packet), would be to use an
interface similar to `SYS_FEVENT`.

A minor question is whether forwarded files should be moved or cloned from the
scheme's file table. The current implementation moves them, but the scheme can
dup the file descriptor to keep it, if moved by the kernel, or respectively,
close it later, if cloned by the kernel.
