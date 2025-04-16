- Feature Name: fscall
- Start Date: 2025-04-16
- RFC PR: (leave this empty)
- Redox Issue: (leave this empty)

# Summary
[summary]: #summary

There are currently 8 miscellaneous filesystem syscalls that can trivially be converted to `SYS_CALL` with a standardized `metadata` format, and another 5 syscalls once we transition to fd-based directory operations.
This would reduce the number of syscalls from (as of procmgr) 36 to 23, or 21 when `MKNS` and `OPEN` are removed.

# Motivation
[motivation]: #motivation

When both the client and server behind a syscall is in userspace, there is little reason to define dedicated syscalls for those things.
Anything that does not need to be in the kernel, should not.

# Detailed design
[design]: #detailed-design

## Old

The 8 syscalls

- `SYS_FCHMOD`
- `SYS_FCHOWN`
- `SYS_GETDENTS`
- `SYS_FSTAT`
- `SYS_FSTATVFS`
- `SYS_FSYNC`
- `SYS_FTRUNCATE`
- `SYS_FUTIMENS`

will henceforth be denoted _legacy fs-metadata syscalls_, whereas

- `SYS_LINK`
- `SYS_RMDIR`
- `SYS_UNLINK`
- `SYS_FPATH`
- ~~`SYS_FRENAME`~~ (TODO: how to replace this?)

shall be called _legacy dirfd-based syscalls_.
The legacy syscalls

- `SYS_OPEN` and
- `SYS_MKNS`

have been intentionally omitted from this RFC, even though they are in the same category, as there is a separate RFC just for openat.

## New

There will be a new flag `CallFlag::STD_FS`, which will be conveyed to `redox-scheme` through the `Opcode::StdFsCall`.
The `metadata` buffer (guaranteed to have space for at least 24 bytes) will be of the format

```
#[repr(C, packed)]
struct StdFsCallMeta {
    kind: u8, // enum StdFsCallKind
    _rsvd: [u8; 7],
    arg1: u64,
    arg2: u64,
}
```

according to the following table

| kind | name       | arg1                   | arg2    | payload   | direction |
|------|------------|------------------------|---------|-----------|-----------|
| 1    | fchmod     | new_mode               | N/A     | N/A       | N/A       |
| 2    | fchown     | new_uid lo, new_gid hi | new_gid | N/A       | N/A       |
| 3    | getdents   | opaque_offset          | N/A     | buffer    | to client |
| 4    | fstat      | TODO: buf size/version | N/A     | buffer    | to client |
| 5    | fstatvfs   | TODO: buf size/version | N/A     | buffer    | to client |
| 6    | fsync      | flags (TODO: datasync?)| N/A     | N/A       | N/A       |
| 7    | ftruncate  | new size               | N/A     | N/A       | N/A       |
| 8    | futimens   | N/A (TODO?)            | N/A     | N/A       | N/A       |
|------|------------|------------------------|---------|-----------|-----------|
| 9    | link       | TODO (sendfd?)         | TODO?   | path?     | to server |
| 10   | unlinkat   | flags (e.g., is_dir)   | N/A     | utf8 path | to server |
| 11   | realpathat | N/A                    | N/A     | utf8 path | to client |
| 12   | renameat   | TODO (sendfd?)         | TODO?   | path?     | to server |

It should be relatively straightforward for all schemes supporting path calls, to switch over to these dirfd-based ones.

# Drawbacks
[drawbacks]: #drawbacks

It makes `SYS_CALL` more complex (although mostly insignificant).

# Alternatives
[alternatives]: #alternatives

It would be possible to reserve `Opcode` ID space (say 128..=255) and allow arbitrary scheme calls, if the goal is to keep as much complexity as possible out of `SYS_CALL`.
That would however require information to be passed other ways, such as whether buffers are read or write.

# Unresolved questions
[unresolved]: #unresolved-questions

How should `FRENAME` (and unimplemented `LINK`) be handled?
Should they take one fd and a relative path, or two fds to send files?
In the latter case, they would use `SYS_SENDFD` as backend, with some kernel logic to determine whether the sent fd is from the same scheme.
With this restriction lifted, it would be possible to link across schemes, but it might be difficult to implement this persistent e.g. for filesystems.

Should this RFC be split into non-path syscalls that can be replaced now, and another RFC after openat is completed?
