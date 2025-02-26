- Feature Name: namespace-scheme
- Start Date: 2024-01-17
- RFC PR: (leave this empty)
- Redox Issue: (leave this empty)

# Summary
[summary]: #summary

As discussed between jeremy_soller and rw_van.

Adopting the new scheme naming format `/scheme/scheme_name` creates an ambiguity when mounting a scheme in the effective namespace. Mounting a scheme will change from `open(":scheme_name", O_CREAT)` to

```
open("/scheme/namespace/scheme_name", O_CREAT | O_EXCL)
```

(See [Unresolved questions](#unresolved-questions) for naming options and issues.)

# Motivation
[motivation]: #motivation

Schemes exist within a namespace. Currently, a scheme is referred to as `scheme_name:`. The namespace manager is `RootScheme` in the kernel, and it is addressed using a scheme name of `:`, which is effectively an empty name `""` followed by a colon `:` separator. Creating/mounting a scheme is done using the format `:scheme_name`, i.e. an empty name followed by a separator and the scheme name as a path. This allows path parsing to naturally detect references to the root scheme, and to pass the scheme name to the root scheme for mounting.

The change to a naming format of `/scheme/scheme_name` make it so a request to the root scheme to mount `scheme_name` cannot be naturally parsed. There is no separator that indicates an empty scheme name.

# Detailed design
[design]: #detailed-design

## Behavior

1. Currently, the effective namespace is referred to using `":"`, i.e. an empty name `""` followed by a colon separator `":"`. The proposed new name for the effective namespace is `/scheme/namespace`. 

2. Currently, mounting scheme `scheme_name` is done by `open(":scheme_name", O_CREAT)`. The new open call will be

```
open("/scheme/namespace/scheme_name", O_CREAT | O_EXCL)
```

This will result in the new scheme being mounted as `/scheme/scheme_name`.

3. If the scheme has already been mounted and is healthy, a second create call will fail (`EEXIST`).

4. `open("/scheme/namespace/scheme_name")` **without** `(O_CREAT | O_EXCL)` present will provide an fd that can be used to query or set **TBD** information about the namespace's view of the scheme. (See [Unresolved questions](#unresolved-questions).)

## Changes

### Kernel dispatch

1. [kernel::syscall::fs::open](https://gitlab.redox-os.org/redox-os/kernel/-/blob/master/src/syscall/fs.rs?ref_type=heads#L66) needs to translate the empty scheme name `""` to the new namespace scheme name `"namespace"`.

2. The function [scheme::SchemeList::new_ns](https://gitlab.redox-os.org/redox-os/kernel/-/blob/master/src/scheme/mod.rs?ref_type=heads#L169) inserts an empty string `""` into the list of schemes as a key, so that parsing a path with an empty scheme is naturally forwarded to the RootScheme. Changing this string to `"namespace"`, in combination with the changes to `open` above, will enable the new format.

### redox-scheme

[redox_scheme::Socket::create_inner](https://gitlab.redox-os.org/redox-os/redox-scheme/-/blob/master/src/lib.rs?ref_type=heads#L129) should be updated as soon as possible to use the new format.

### Scheme providers

`redox-scheme` is the best practice interface for schemes. Wherever possible, schemes should be updated to use redox-scheme rather than implementing the scheme protocol themselves.

For those schemes that cannot not use `redox-scheme`, the `open` call to create the scheme will need to be modified.

### Contain

Contain should already be using redox-scheme and should therefore update automatically when redox-scheme is updated. However, this should be verified.

# Drawbacks
[drawbacks]: #drawbacks

There is no compelling reason to not do this.

# Alternatives
[alternatives]: #alternatives

## Status Quo

Due to the change in scheme naming described in the "Scheme Path" RFC, there is an ambiguity in referencing the root scheme. Does `/scheme/s1` mean "open the null path on scheme "s1" or does it mean open "s1" on the root scheme? This is unnecessarily confusing and would require coding of `open` flags to resolve the ambiguity.

## Special Files and Mount Points

Unix uses "special files", e.g. block-special and character-special files, to refer to physical devices and pseudo-devices. Special files are indicated by a filetype in their status flags, and the "major" and "minor" numbers are interpreted to determine the driver and specific resource (e.g. partition) for the special file. For filesystem providers, the special device can be "mounted" at a named point in the filesystem, e.g. the block-special device `/dev/nvme0n1p2` can be mounted at `/home`. This masks the file named `/home` and connects the filesystem to that point.

Conceivably, Redox could use a status bit to indicate a path that represents a "service provider" (scheme) that could be mounted, and then the "mount" system call could make the scheme accessible at some location in the filesystem. There would be a (mostly) one-to-one correspondence between what special files exist, and what schemes are in the namespace. This ultimately is very similar to the proposed mapping.

# Unresolved questions
[unresolved]: #unresolved-questions

1. What should the name of the effective namespace scheme (RootScheme) be?

    - `/scheme/ns`
    - `/scheme/namespace` (see note below)
    - `/namespace` (treats namespace as a thing that is not a scheme)
    - `/scheme/ens` (`/scheme/rns` for "real namespace")

The recommended name in this RFC, `/scheme/namespace`, will be the choice once this RFC is approved.

2. We need a naming format for distinct namespaces, e.g. `/scheme/namespaces/n` for namespace `n`. It should be separated from the naming of the effective namespace.

    - Having names for distinct namespaces would allow us to have a capability-based security mechanism where schemes can be inserted or removed from namespaces. It would also enable user-managed namespaces and schemes. Details to be discussed in another RFC.
    
    - Using a name that is distinct from the normal namespace, e.g. `/scheme/namespaces` (plural), would allow us to have arbitrary names for new namespaces. If we choose `/scheme/namespace/scheme_name` for mounting a scheme, and `/scheme/namespace/n` to refer to namespace `n`, then we are forced to use numbers rather than names for namespaces, and we will have schemes under the namespace folder at two different levels, creating confusion.

This is out of scope for this RFC.

3. We need better management of namespaces, but it should be addressed in a separate RFC.

This is out of scope for this RFC.

4. Should we change for `open` to `mount` when creating a new scheme? It includes additional flags and options.

This is out of scope for this RFC.

5. Assume that when a scheme is restarted, it will call `open("/scheme/namespace/scheme_name", O_CREAT | O_EXCL)`. If those flags are not provided, then the open is to query the namespace about the state of the scheme. What actions can be performed on the resulting fd, and what information is provided?

This is out of scope for this RFC.
