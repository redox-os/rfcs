- Feature Name: c_char
- Start Date: 2025-03-10
- RFC PR: (leave this empty)
- Redox Issue: (leave this empty)

# Summary
[summary]: #summary

Due to recent changes to Rust, c_char is not consistently defined and is causing compilation errors on Aarch64 and RISC-V.

In order to ensure consistent behavior across platforms, and to avoid possible logic errors due to signed vs. unsigned char types,
Redox will define the C type `char` as `signed char`, and the Rust type `c_char` as `i8`.

# Motivation
[motivation]: #motivation

Due to recent changes to Rust to align with the definition of `char` on various architectures and operating systems,
Redox compilation was broken.
It has been patched for x86 and x86_64 but not for Aarch64 and RISC-V.

In order to resolve this problem, and to ensure consistency and avoid compilation problems and logic problems in the future,
Redox will define `char` as `signed char` for the C data type and `c_char` as `i8` for the Rust data type.

Since these definitions are consistent with Windows and MacOS, they should require minimal change and should not affect most code.

# Detailed design
[design]: #detailed-design

Changes are required as follows.

### C data types

- The Redox fork of gcc must be changed for all relevant platforms
- All libraries must be compiled and tested on Redox (to the extent possible) for each architecture
- Changes to gcc should be upstreamed when they are tested (I'm not sure of the process for this)

### Rust data types

- The Redox fork of the Rust compiler must be changed
- The upstream libc crate must be changed
- relibc must be changed
- All code compiled and tested (to the extent possible)
- Changes to the Rust compiler should be upstreamed

# Drawbacks
[drawbacks]: #drawbacks

Some course of action is required to support Aarch64 and RISC-V.

# Alternatives
[alternatives]: #alternatives

The alternative is to use the Linux definitions for `char` and `c_char`, which vary by CPU architecture.
This is more likely to cause problems for Redox than it is for Linux,
as Redox needs to switch between C types and Rust native types frequently.
Although very little OS and relibc code is expected to do `c_char` comparisons where the high bit is relevant,
it is a possible source of bugs.

# Unresolved questions
[unresolved]: #unresolved-questions

The details of what files need to be changed need to be determined.
Perhaps we can track that in an issue.
