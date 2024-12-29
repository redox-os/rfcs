- Feature Name: base-system-repo
- Start Date: 2024-12-29
- RFC PR: (leave this empty)
- Redox Issue: (leave this empty)

# Summary
[summary]: #summary

Merge the repos forming the base system into a single repo.

# Motivation
[motivation]: #motivation

While we will likely be able to stabilize the userspace ABI relatively soon through dynamic linking of relibc and libredox, the syscall interface as well as the interface of system services between each other and with relibc is likely to take much longer to stabilize if it ever gets stabilized. Many internal improvements currently require merge requests across multiple repos that need to be merged at the same time which makes such changes harder to do. For this reason some repos contain programs that don't actually quite fit in the repo but are only there because it makes changes easier. For example the driver repo contains inputd, fbbootlogd and fbcond, none of which are actually drivers, but all of them are somewhat coupled with the graphics drivers. Merging all programs in the base system into a single repo will make it easier to make atomic changes. It is also currently not safe for users to update individual packages that are part of the base system.

# Detailed design
[design]: #detailed-design

This is the bulk of the RFC. Explain the design in enough detail for somebody familiar
with the language to understand, and for somebody familiar with the compiler to implement.
This should get into specifics and corner-cases, and include examples of how the feature is used.

The following repos will be merged into a single "base" repo:

* audiod
* contain
* drivers
* escalated
* init
* initfs
* ipcd
* logd
* netstack
* ptyd
* ramfs
* randd
* zerod

The following repos will **not** be merged into the "base" repo:

* binutils
* bootloader (the bootloader interface rarely changes)
* coreutils
* dash
* findutils
* installer
* ion
* pkgar
* pkgutils
* redoxerd
* uutils
* pretty much everything outside of the core category in the cookbook

For the actual merge, I propose to make for each repo a commit which moves the entire content to a subdirectory and then git merge all the repos together into a new repo. This preserves the full history of all repos as well as git blame. And afterwards all issues will have to be transfered to the new repo.

# Drawbacks
[drawbacks]: #drawbacks

It is a non-trivial amount of work.

# Alternatives
[alternatives]: #alternatives

What other designs have been considered? What is the impact of not doing this?

# Unresolved questions
[unresolved]: #unresolved-questions

Should the following repos be merged into the "base" repo? I think they should, but it might be less disruptive to keep them in separate repos at least for the time being.

* bootstrap
* kernel
* relibc
* redoxfs
* orbital
