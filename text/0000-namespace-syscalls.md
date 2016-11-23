- Feature Name: namespace-syscalls
- Start Date: 2016-11-23
- RFC PR: (leave this empty)
- Redox Issue: (leave this empty)

# Summary
[summary]: #summary

A namespace is designed to implement the following with one abstraction:
- `cap_enter`, by default
- `chroot`, by allowing a filter on `file:`
- [OS-level virtualization](https://en.wikipedia.org/wiki/Operating-system-level_virtualization) such as [FreeBSD-style Jails](https://en.wikipedia.org/wiki/FreeBSD_jail) or [Illumos-style Zones](https://en.wikipedia.org/wiki/Solaris_Containers), with more complex filtering of scheme access

It achieves this with the addition of three syscalls:
- `getns`, which gets the current namespace
- `mkns`, which creates a new namespace
- `setns`, which switches namespaces

# Motivation
[motivation]: #motivation

Why are we doing this? What use cases does it support? What is the expected outcome?

# Detailed design
[design]: #detailed-design

```rust
// Get the current namespace
let old_ns = getns();
// Create a new empty namespace
let new_ns = mkns(&[]);
// Switch to the new namespace
// This is only possible because this process created new_ns
setns(new_ns);

// Create a child fork
let child = clone(0);
if child == 0 {
    // Execute a process in the new namespace
    // This will reset the original namespace, preventing setns(old_ns)
    exec("process-to-contain");
}else{
    // Create a new `file:` in the new namespace
    let file_scheme = open(":file", O_CREAT | O_RDWR);

    // Switch back to the original namespace
    // This is only possible because this process was once inside of old_ns
    setns(old_ns);

    // For every file event in the new `file:`
    for event in file_scheme.events() {
        // Translate it if required and forward it to the original `file:`
        handle_event(event);
    }
}
```

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

- Potential rooting by placing a setuid program in a specially designed container

# Alternatives
[alternatives]: #alternatives

What other designs have been considered? What is the impact of not doing this?

# Unresolved questions
[unresolved]: #unresolved-questions

What parts of the design are still TBD?

- How to prevent rooting by placing a setuid program in a specially designed container
