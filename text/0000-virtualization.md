- Feature Name: Virtualization Environment
- Start Date: 2018-11-11
- RFC PR: 
- Redox Issue:

# Summary
[summary]: #summary

Outline of a possible approach for Hypervisor based Virtualization

# Motivation
[motivation]: #motivation

Redox-OS is not particularly suited for Virtualization purposes. To enable virtualization from embedded deployments to large-scale datacenter deployments, we need to figure out how to approach this.

# Detailed design
[design]: #detailed-design

While Microkernels in general prove to be awesome for virtualization, the redox kernel isn't particularly suited for it:
Kernels need to be optimized for virtualization. That's usually either done with a kernel module on top of a general purpose kernel (KVM/Bhyve) or targeted development (ESXi, Xen, Hyper-V, L4Re ...). This involves IO optimizations, scheduling and slimming down the kernel to a featureset based on virtualization tasks. While implementing virtualization in userspace is a possiblility, IO Optimizations and scheduling will either remain untouched or the kernel gets bloated to accomodate both general purpose aspects and virtualization.

A solution to this is, to split up kernel development into different projects:
1. A common subset
2. The general purpose kernel (subset kernel + modifications for server and desktop workloads)
3. A Hypervisor (subset kernel + virtualization)
4. Possibly a Para-Virtualized kernel that tapps into the hypervisor for less overhead (if following the Xen/Hyper-V design)

Potential benefits will be expertise in scheduling leading to efficiency and design improvements across all kernel related projects targeting to offer a solution for everything from embedded systems to large scale deployments thanks to a modular and integrated design.

# Drawbacks
[drawbacks]: #drawbacks

Since this approach requires an extraction of a base kernel, and a development process involving multiple kernel-projects, this involves a higher amount of organization and work due to adding dependencies. 

# Alternatives
[alternatives]: #alternatives

Xen would be the go-to alternative as a Hypervisor. This still involves extracting a base kernel though for Para-Virtualization/Domain0. 

# Unresolved questions
[unresolved]: #unresolved-questions

UFF too many to list them at this point in time.
