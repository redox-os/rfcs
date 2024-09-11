- Feature Name: coprocess
- Start Date: 2024-09-10
- RFC PR: https://gitlab.redox-os.org/redox-os/rfcs/-/merge_requests/23
- Redox Issue: (leave this empty)

# Summary
[summary]: #summary

This RFC suggests defining the _coprocess_ concept, between that of threads and processes in terms of isolation, allowing significantly faster context switching that can bypass the kernel.
Coprocesses will on x86 be based on using User Protection Keys, theoretically allowing up to 16 coprocesses in the same process set.
Crucially, this will impose requirements on the coprocesses, namely that they are position-independent executables, and that program code is not allowed to use any instruction that can modify the active protection key.
Hence, any program loading code will need airtight checking, potentially by guarding executable mmap rights behind capabilities.

# Motivation
[motivation]: #motivation

Microkernel-based systems are typically expected to isolate system components including drivers, network, and filesystem, into separate address space and set of capabilities, etc.
However, this is a tradeoff between performance and isolatability, and it is not entirely obvious where the "walls" should be, and how strong.
Currently, the Redox model is generally based on the conventional division of components into daemons following the Unix philosophy.
For example, when accessing a mounted redoxfs-formatted USB drive, this actively involves the client itself, `redoxfs`, `usbscsid`, `xhcid`, and passively `pcid`.

The fact that this model introduces dependency chains, of usually at least 3 process boundaries, has latency implications, even though throughput can be decoupled from latency by using asynchronous queues.
The maximum achievable context switch (one-way) latency last time this was checked on a 5950x, was 711 cycles, using a custom kernel with a simple direct switch syscalls (in reality this will be slightly higher, as it made certain assumptions).
This latency will obviously stack the more switches are required to complete any given request.
Thus, it would be useful if similar processes could be combined into groups of coprocesses.

Modern x86 cpus introduce a feature called user protection keys, allowing 16 possible tags that can be assigned to individual pages, where userspace can modify the "accessible" and "writable" bits for each indivdual tag, using the RDPKRU and WRPKRU instructions.
A result of this is that multiple programs, assuming they are position-independent, can be embedded into the same address space, where these can be switched to by modifying the PKRU.

# Detailed design
[design]: #detailed-design

Henceforth, we continue to use the term _process_ for referring to a set of kernel scheduling units with a fixed address space.
Using this definition, a _process_ consists of a set of coprocesses.
A _simple process_ is a process containing only one coprocess.

The memory-related syscalls will be extended to allow assigning protection key at page granularity, using the same range-based syscalls.
Protection key 0, called the master key, will be reserved for redox-rt, where PKRU == ALL := 0xffff_ffff.
Redox-rt runs in _master mode_ when PKRU == ALL, and unlike the coprocesses, is allowed to contain code that sets the PKRU.
Key 0 will be used for intra-process context switching, and will always be access-disabled inside any coprocess.

Any memory syscall that adds new execute permission, will need to be forbidden outside PKRU == ALL.
This can either be checked explicitly by the kernel by reading PKRU, or if [virtual memory capabilities](https://gitlab.redox-os.org/redox-os/rfcs/-/merge_requests/22), by assigning a user protection key to capability arrays as well.
Operations like execv will thus need to occur via redox-rt, which internally needs to switch to master mode.

Redox-rt will obviously need to be dynamically linked for this to work.
Additionally, all coprocesses will need to be PIEs, as they will be embedded in the same address space.

TODO: How will scheduling work?
The kernel always tracks the current context, which will somehow need to be set as part of the intra-process switch.
Even if this requires entering the kernel, this would avoid the page table switch cost, so it would still be an overall improvement.
It could be possible to store the pointer to the current context somewhere in the address space (where the page containing this pointer would be master-tagged), if this does not incur any unnecessary performance cost in the kernel.

If alternatively the kernel design is restructured so there can only be one context per address space and hardware thread, it would be possible to implement intra-process scheduling in redox-rt.
This is however probably a change the size of a research project, and will certainly need deep tradeoff analysis.

The fact that PKRU is a bitset, will allow for flexible intra-process memory sharing.
For example, if redoxfs and nvmed reside in the same process, they can allocate IO buffers with a key available in both coprocesses.
Additionally, a tracer may be granted read-only access to the traced coprocess.
This would however require the traced coprocess to uphold the same guarantees, which might not be true in most practical cases.

# Drawbacks
[drawbacks]: #drawbacks

The obvious drawback is that this adds complexity, and it should be possible to achieve reasonable performance even without the coprocess concept.


# Alternatives
[alternatives]: #alternatives

The alternative would be to stick to the currect "process = same address space" model.

Process-context identifiers would improve indirect latency by reducing TLB stalls after address space switches, but empirically it appears PCIDs does not directly affect the time it takes to switch page tables itself.

# Unresolved questions
[unresolved]: #unresolved-questions

This may need to be analyzed in terms of CPU side channel resistance.
It [appears that](https://www.intel.com/content/www/us/en/developer/articles/technical/software-security-guidance/best-practices/related-intel-security-features-technologies.html) protection keys are equivalently protected on Intel CPUs compared to user/kernel or different-PCID pages.
In other words, any Meltdown-immune CPU will, according to this advisory, similarly be immune to Meltdown.

Generally speaking, WRPKRU will be serializing, so v1 should be implicitly mitigated.
However, there are µ-architectures where instructions can sometimes execute even if they are not _architectural_, i.e. the CPU can sometimes make guesses about which instructions it will run, which may not necessarily always be correct.
This does however seem unlikely, as WRPKRU is serializing, and any Meltdown-immune CPU will not even be able to fetch cache lines from different protection keys until _after_ the WRPKRU.
V2 might require RSB filling or IBPB depending on hardware.
This question can possibly be generalized into other types of side channel, cf. [1].

So far, this RFC has (implicitly) only discussed non-SMP systems, but the PKRU register is separate for each hardware thread.
Thus, there are multiple possible ways to extend this to SMP.
For example, the tags assigned to coprocesses could be allocated symmetrically, where each coprocesses could consist of multiple threads, as would be expected for simple processes.
Alternatively, if all coprocesses are singlethreaded, it would be possible to assign one CPU-tag pair to each coprocess, increasing the set size.

Some kernels, including Linux, supports _core scheduling_ where sibling hyperthreads never run in different address spaces.
This can have performance advantages due to the internal cache and queues often shared between hyperthreads.
If keys are assigned nonsymmetrically to hardware threads, this should be taken into account.

There is a similar less commonly available Kernel Memory Protection Key feature, using the IA32_PKRS MSR.
This could similarly allow e.g. lightweight kernel modules, to for example allow dymamically loading schedulers, or other parts of the kernel where it would make sense to dynamically replace implementations.

# References

Shapiro, Jonathan S (2003). Vulnerabilities in Synchronous IPC Designs, _IEEE Symposium on Security and Privacy_. https://srl.cs.jhu.edu/courses/600.439/shap03vulnerabilities.pdf
