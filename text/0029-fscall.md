- Feature Name: fscall
- Start Date: 2025-04-16
- RFC PR: https://gitlab.redox-os.org/redox-os/rfcs/-/merge_requests/29
- Redox Issue: (leave this empty)

# Summary
[summary]: #summary

There are currently 8 miscellaneous filesystem syscalls that can trivially be converted to `SYS_CALL` with a standardized `metadata` format, and another 5 syscalls once we transition to fd-based directory operations.
This would reduce the number of syscalls from (as of procmgr) 36 to 25, or 21 when `MKNS`, `OPEN`, `FRENAME`, and `LINK`, are removed.

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

- `SYS_RMDIR`
- `SYS_UNLINK`
- `SYS_FPATH`

shall be called _legacy dirfd-based syscalls_.
The legacy syscalls

- `SYS_OPEN`,
- `SYS_LINK`,
- `SYS_FRENAME`, and
- `SYS_MKNS`

will be omitted from this RFC, as it is less obvious what the replacements will look like, and since openat has a separate RFC.

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
| 1    | fchmod     | new_mode               | RSVDZ   | N/A       | N/A       |
| 2    | fchown     | new_uid lo, new_gid hi | RSVDZ   | N/A       | N/A       |
| 3    | getdents   | opaque_offset          | RSVDZ   | buffer    | to client |
| 4    | fstat      | RSVDZ: buf size/version| RSVDZ   | buffer    | to client |
| 5    | fstatvfs   | RSVDZ: buf size/version| RSVDZ   | buffer    | to client |
| 6    | fsync      | flags (like fdatasync) | RSVDZ   | N/A       | N/A       |
| 7    | ftruncate  | new size               | RSVDZ   | N/A       | N/A       |
| 8    | futimens   | RSVDZ                  | RSVDZ   | times     | to server |
|------|------------|------------------------|---------|-----------|-----------|
| 10   | unlinkat   | flags (e.g., is_dir)   | RSVDZ   | utf8 path | to server |
| 11   | realpathat | RSVDZ                  | RSVDZ   | utf8 path | to client |

(Where RSVDZ means "reserved - must be zero".)

It should be relatively straightforward for all schemes supporting path calls, to switch over to these dirfd-based ones.

# Drawbacks
[drawbacks]: #drawbacks

- It makes `SYS_CALL` more complex (although mostly insignificant).
- The kernel will not be able to enforce utf-8 for paths sent via this mechanism, but whether there should exist a separate SYS_CALL just for that, is highly debatable.

# Alternatives
[alternatives]: #alternatives

It would be possible to reserve `Opcode` ID space (say 128..=255) and allow arbitrary scheme calls, if the goal is to keep as much complexity as possible out of `SYS_CALL`.
That would however require information to be passed other ways, such as whether buffers are read or write.

# Unresolved questions
[unresolved]: #unresolved-questions

Are there any unresolved questions?
