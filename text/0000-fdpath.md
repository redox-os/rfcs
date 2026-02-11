- Feature Name: fdpath
- Start Date: 2025-12-19
- RFC PR: https://gitlab.redox-os.org/redox-os/rfcs/-/merge_requests/34
- Redox Issue: (leave this empty)

# Summary
[summary]: #summary

This RFC proposes several interfaces intended to provide capability-limited access to hierarchical filesystems, as well as the ability to look up paths based on already opened fds.

# Motivation
[motivation]: #motivation

With capability-based security being introduced to Redox, our current `fpath` syscall is no longer sufficient for obtaining the original path an fd was opened from, as it requires schemes to hardcode the name they are bound to.
This requires a hack in `nsmgr`, and currently forbids schemes from being renamed, at least those providing a filesystem structure.

# Detailed design
[design]: #detailed-design

The following interfaces will mostly not be enforceable on all schemes, and there are indeed schemes that do not provide a filesystem structure.
That said, when configuring sandboxed systems, all involved schemes will need to comply with this convention in order to be sandboxable.

## openat

The openat call will take an additional argument `MAX_UPWARD_DEPTH`, to be stored in file descriptions and which limits the maximum amount of `../` allowed from subsequent openat calls on that fd.
A max upward depth of 0 is logically equivalent with `RESOLVE_BENEATH` on other OSes.
Assuming directory hardlinks are forbidden so that the filesystem follows a tree structure except possibly in the presence of mount points, so that dirfds only need to store the underlying filesystem ID of its directory node, schemes are for files only expected to keep track of the directory entry used to resolve a file (which is uniquely referenced), as well as the file node itself (which multiple hardlinks can point to).
It should always be allowed without privileges to derive a dirfd with shallower max upwards depth, resulting in a subtree.
As fscall (most likely a variant of dup with some flag or standardized buffer names), called `DERIVE(fd, MAX_UPWARD_DEPTH)` from now on, will be added that allows opening the maximum possible "`../../..`" from any given file fd or dirfd (then with depth 0 relative to that fd, obviously).

## tokens

A concept called _token dirfds_ will be introduced. This is a dirfd with no allowed calls, i.e. no openat from it, getdents, fstat etc, as the name implies. Token fds will be opened using `DERIVE(fd, TOKEN)`, and the returned fd should not be a newly created fd, but a forwarded instance of a unique fd corresponding to that dir inode (which means the scheme will have a token dirfd opened by itself).

TODO: The implementation can choose between the following two. It would be good to keep unnecessary interfaces out of the kernel, but at the same time avoid requiring a centralized scheme for all namespaces. If namespaces are intended to be decentralized, the token handling could be moved out to a separate daemon, but probably out-of-scope now.

- An external scheme can provide the token fds, preferably `nsmgr`, or a different daemon. When handling `DERIVE(dirfd, TOKEN)`, it would open a new token and store it in the directory entry, or if there already exists one, `DUP` and return that.
- (A kernel interface `fdmap` will be added that consists of a map from fds to some arbitrary user-defined integer, where the keys are opaque to userspace (even though this could be changed in the future depending on the capability system later adopted). The fds would be served by the scheme, and the filesystem daemon would need to keep track of the token fds that exist for a given directory entry, since the returned fd from `DERIVE` must be reference-identical between calls.)

## mounting

Although it's not an explicit goal to support nested mount points like other Unix-like systems, there needs to be a way to `a := openat(redoxfs_root, "scheme")` followed by `fd := openat(a, "some-scheme")`; support for `openat` makes it insufficient for `/scheme/...` to be merely syntactical. For that reason, schemes will have `DERIVE(dirfd, MOUNT)` that first checks for emptiness of `dirfd`, and then returns a "RAII" fd to a lock. These handles will be owned by `nsmgr`.

## fd-to-path

The filesystem call `relpath` will be added, taking two fds and an output buffer. The first fd is logically a _reference_ dirfd, whereas the second fd can be either a dirfd or file fd. The kernel will be responsible for checking whether the fds are provided by the same scheme, and if so, unwrap both fds into scheme tags and sent as a regular scheme call.
It will then be the responsibility of the scheme to walk upwards from the file entry/directory entry and append to the output buffer the resulting relative path (backwards), until the max upward depth is reached, or the root dirfd of the filesystem is reached and the reference fd was not a parent of the original fd. 

If the max upward depth is reached or if the kernel returns EXDEV (the two fds passed originated from different schemes), the client should instead request `top_fd := DERIVE(orig_fd, MAX_UPWARD_DEPTH)` and then call `nsmgr` with a `SYS_CALL` request similar to `relpath`, taking a namespace fd, external `top_fd` and output buffer to be filled and later read by the client.
When mapping scheme root fds to namespace bind points during namespace construction, `nsmgr` will obtain the (unique) token fds corresponding to those root fds, and build an `fdmap` with those as keys, and internal ids etc. as values, where it has an internal map where those ids can be converted to names.
Later, `nsmgr` will be able to use this `fdmap` to lookup the `/scheme/...` prefix, where all paths are assumed to be partitionable into at most two components -- the `/scheme/<scheme>` part and the `provided/by/scheme` part, so called _flat mount points_.

While it is out-of-scope for this RFC to support non-flat mount points (such as "`/boot/efi`", if `/`, `/boot/` and `/boot/efi/` are separate filesystems), the fd-to-path method could probably be extended to that by letting `nsmgr` map the token dirfds to the dirfd and name it is mounted to, repeating the process to get the full path. (Of course, the filesystem would need to keep track of when an empty dirfd is mounted, in which case it would indicate to the caller that it should ask `nsmgr` to get the root fd of the mounted filesystem instead).

While out-of-scope for this RFC as well, the same method of using token fds could maybe also be leveraged for handling POSIX uid/gid permissions. The filesystem provider can then keep an `fdmap` of `uidfd => fs uid`, and analogously for gids, allowing system uids to be mapped to filesystem uids dynamically. It may benefit from additional protections on whether such token fds are allowed to be sent further by the scheme and so on, or the scheme would need to be trusted.

# Drawbacks
[drawbacks]: #drawbacks

This obviously adds complexity both to userspace, in the form of many new additional fs calls, and to the kernel, in the form of the double-fd scheme call and `fdmap`.
The latter interfaces can probably be generalized in the future though.

# Alternatives
[alternatives]: #alternatives

The obvious alternative would be for `nsmgr` to simply man-in-the-middle all fs calls, but that would add considerable latency overhead. Fd-to-path could also be implemented by simply sending the original fd to `nsmgr`, and let it maintain scheme fds it can use to figure out where it should filter the scheme's path, and what prefix (`/scheme/...`) it should prepend.
However, if the goal is to be able to provide filesystems without requiring that all fs providers be trusted, it would need some form of fd-based lookup either way.

# Unresolved questions
[unresolved]: #unresolved-questions

It is slightly unclear if `fdmap` should be built into the handler for certain syscalls (so the scheme gets its mapped value directly in the SQE for example, or if it should be based on sending an fd and then using a separate syscall for looking that up).

Man-in-the-middle schemes could filter only the directory structure but fd-forward all files (for perf), in which case they would either need to keep the names as is for fd-to-path to work (meaning name resolution is done by the _filtered_ underlying scheme), or open the leaf fds with a max upward depth of zero.
With upward depth zero, the man-in-the-middle filter would also need to be the namespace manager, since the client uses a single reference to a global namespace fd to lookup between mount points.
An more scalable alternative could be to attach a "parent dirfd", i.e. the nearest fd in the parent chain provided by a different scheme, to all of a subset of fds.
This could be done either in the kernel, or if the POSIX filetable is made indirect, in redox-rt by storing both the capability itself and an additional "parent" capability.
