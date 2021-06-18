- Feature Name: scheme_forward_fds
- Start Date: 2021-06-12
- RFC PR: https://gitlab.redox-os.org/redox-os/rfcs/-/merge_requests/17
- Redox Issue: (leave this empty)

# Summary
[summary]: #summary

This feature allows schemes to forward file descriptors local to the process
that is responsible for handling that scheme. In other words, rather than
giving the kernel a word-size integer representing a scheme-local file
descriptor number, it can instead give the kernel a fully valid file descriptor
in its _process file descriptor namespace_, that might have come from a
completely different scheme in the first place. Therefore, it allows
_forwarding_ file descriptors.

# Motivation
[motivation]: #motivation

Microkernels are intrinsically meant to be as minimal as possible, but it
quickly gets inconvenient and not as flexible as it could be, when the scheme
handler has to be responsible both for opening file descriptors, but also for
handling those file descriptors.

Take the `irq:` scheme as an example. We need IRQ handling to be _fast_, and as
low-latency as possible. Ideally the only thing before the actual code, should
be to mask the interrupt and directly switch to the handler, almost
signal-like. However, we still need the kernel to protect userspace processes
from each other, and for the IRQ scheme, this means that the kernel will have
to keep track of which process owns which IRQ. I/O ports are very similar, and
if the kernel would do this, then it would in addition have to track ranges,
almost like grants. However, with the feature described by this RFC, we could
designate resource allocation to a userspace scheme, and give that scheme
handler near-full access to the kernel scheme. Once opened by a client, it will
have a file descriptor that directly points to the kernel IRQ scheme.

The same goes for userspace. Now, imagine the task of handling disk partitions.
Currently, this is done in the disk drivers, mainly to maintain flexibility and
performance, as a middleman scheme would add latency to every single byte read
or written. And while it may be reasonable to add this duplicate functionality
to multiple drivers, the same problem applies for e.g. firewalls. __A chroot
tool implemented via schemes and namespaces, would also become zero-cost for
data, even if it may have to filter metadata access, i.e. directory
structures.__

# Detailed design
[design]: #detailed-design

Normally, when a scheme wants to respond, it takes a packet it received from
the kernel earlier. It keeps all fields (even though every field but the `id`
is optional for this), and sets `a` to the return value of that scheme message.
However, when triggerings events it sets the `id` field to zero, and sets `a`
to `SYS_FEVENT`.

To support our particular use case, we can do the same sort of scheme-to-kernel
message, but instead of having `a` be `SYS_FEVENT`, we could set it to
`SYS_OPEN` or some other new "syscall number". We let it take the actual
syscall ID as a regular parameter, together with the file descriptor number,
local to the _context_ handling the scheme, as opposed to _the scheme "fd
namespace" itself_.

Although an implementation detail, the kernel ought to be able to use some
sentinel value, either as a success opcode (probably `-1` as that file
descriptor number would otherwise not survive well in the POSIX userland world,
which reserves `-1` to indicate failure), or, probably better, as a
userspace-forbidden `Err(ESENTINEL)` error code. We might however have to
extend the scheme trait with a pseudo-syscall like `kopen`, since the
pre-io_uring kernel relies on dynamic dispatch for all system calls.

# Drawbacks
[drawbacks]: #drawbacks

The trivial drawback is that it adds complexity to the kernel, although in this
case it is relatively minor. The question is rather whether we actually do want
the functionality of being able to forward file descriptors that previously
originated from different schemes.

# Alternatives
[alternatives]: #alternatives

The obvious alternative would be to simply allow schemes to communicate with
the client that it wants to send it a file descriptor via previously suggested
"fd channels", like `cable:` or `sendfd:`. However, this would require the
client to implement different logic for this, and thus would not be as
flexible. Nothing really forbids both of two such interfaces to be used in
parallel.

# Unresolved questions
[unresolved]: #unresolved-questions

A minor question is whether we want to generalize this to every single syscall
flagged with `SYS_RET_FILE`, which would also include other similar syscalls
like `SYS_DUP`. The most notable use-case for this is probably `ipcd`, which
could benefit from redirecting everything to the kernel pipe scheme. Or vice
versa, to in the extreme case let the kernel redirect the pipe scheme to some
userspace daemon like `ipcd`.
