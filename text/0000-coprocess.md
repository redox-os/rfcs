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

# Prior art

Intel added the PKU feature for a reason, and although relatively recent, various possible configurations have been explored in academia [1-3].
The performance results are overwhelmingly positive, for example [2]

> Cross-domain switches are 16–116x faster than regular process context switches

or [3]

> Notably, the unikernel with our isolation exhibits only 0.6% slowdown on a set of macro-benchmarks.

The closest existing system appears to be [2], which uses the a modified protection key hardware implementation for isolating possibly untrusted components in a process.
This idea does however appear to be mostly unexplored in a µkernel setting, on conventional hardware.

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
A _master page_ is a page for which the PKEY=0.
Key 0 will be used for intra-process context switching, and will always be access-disabled inside any coprocess.

Redox-rt will thus be possible to call from any coprocess. It will need to implement access checks when context switches are requested.
It will also need to shadow the current PKRU inside a master page, so it can check directly after the PKRU instruction if the switch was allowed or not.
This is because any coprocess can jump to the PKRU instruction.

Any memory syscall that adds new execute permission, will need to be forbidden outside PKRU == ALL.
This can either be checked explicitly by the kernel by reading PKRU, or if [virtual memory capabilities](https://gitlab.redox-os.org/redox-os/rfcs/-/merge_requests/22), by assigning a user protection key to capability arrays as well.
Protection keys for usermode pages similarly apply in the kernel, even if it is not yet decided if virtual memory capabilities will reside in user or kernel memory.
In the latter case, it is obviously not possible for userspace to set IA32_PKRS, so it would need to somehow store that information elsewhere.
It could for example store the protection key as a pointer tag in each internal file descriptor pointer, or possibly a full bitset, or alternatively at capability page granularity.
Another perhaps better option would be to statically partition the capability address space by protection key, in which case the kernel can simply compare the pkey bits of the capability address, with the current PKRU register.
Operations like execv will thus need to occur via redox-rt, which internally needs to switch to master mode.

Redox-rt will obviously need to be dynamically linked for this to work.
Additionally, all coprocesses will need to be PIEs, as they will be embedded in the same address space.

If alternatively the kernel design is restructured so there can only be one context per address space and hardware thread, it would be possible to implement intra-process scheduling in redox-rt.
This is however probably a change the size of a research project, and will certainly need deep tradeoff analysis.

The fact that PKRU is a bitset, will allow for flexible intra-process memory sharing.
For example, if redoxfs and nvmed reside in the same process, they can allocate IO buffers with a key available in both coprocesses.
Additionally, a tracer may be granted read-only access to the traced coprocess.
This would however require the traced coprocess to uphold the same guarantees, which might not be true in most practical cases.

# Drawbacks
[drawbacks]: #drawbacks

The obvious drawback is that this adds complexity, and it should be possible to achieve reasonable performance even without the coprocess concept.

The coprocess concept is not generalizable to arbitrary programs.
It imposes several restrictions on programs, such as position independence and the absence of unverified program loading
Furthermore, protection keys control only data accesses, not instruction fetches.
Coprocesses would on ther other hand have their own key-associated address subspace, making any instruction page effectively execute-only.
Additionally, if that code results in any attempt at accessing data not pertaining to the caller's address substace, such as global variables, it would immediately result in a segmentation fault handled by redox-rt.
Such a fault handler may cross-check the page protection key of the instruction address, with the current PKRU, and detect an intentional or unintentional attempt at executing another coprocess' text.
Hence, although other coprocess' text may be leaked, there is likely no significant significant security issue with this, assuming .text secrecy is not part of the threat model.

A more significant issue with this however, is that coprocesses can arbitrarily jump to code inside redox-rt, including the WRPKRU instruction itself.
Thus, redox-rt in master mode will need to shadow the current PKRU in a master page only it can access, and implement checks at the required places to detect if this happens.
As a side-channel countermeasure (it will have entered another coprocess's address subspace!), this must always result in the termination of the caller.

As for reliability, the possibility of executing other coprocess' text can presumably be limited by separating and/or randomizing programs' locations.
Since there are only 16 possible keys, the amount of remaining address space (4-level: 8 TiB, or 5-level: 4 EiB) is still huge.

# Alternatives
[alternatives]: #alternatives

The alternative would be to stick to the currect "process = same address space" model.

Process-context identifiers would improve indirect latency by reducing TLB stalls after address space switches, but empirically it appears PCIDs does not directly affect the time it takes to switch page tables itself.
These two optimizations are orthogonal though, and indeed the TLB can be represented as an M×N matrix of address subspaces, with M PCIDs and N PKEYs.

# Unresolved questions
[unresolved]: #unresolved-questions

TODO: How will scheduling work?
The kernel always tracks the current context, which will somehow need to be set as part of the intra-process switch.
Even if this requires entering the kernel, this would avoid the page table switch cost, so it would still be an overall improvement.
It could be possible to store the pointer to the current context somewhere in the address space (where the page containing this pointer would be master-tagged), if this does not incur any unnecessary performance cost in the kernel.

This may need to be analyzed in terms of CPU side channel resistance.
It [appears that](https://www.intel.com/content/www/us/en/developer/articles/technical/software-security-guidance/best-practices/related-intel-security-features-technologies.html) protection keys are equivalently protected on Intel CPUs compared to user/kernel or different-PCID pages.
In other words, any Meltdown-immune CPU will, according to this advisory, similarly be immune to Meltdown.

Generally speaking, WRPKRU will be serializing, so v1 should be implicitly mitigated.
However, there are µ-architectures where instructions can sometimes execute even if they are not _architectural_, i.e. the CPU can sometimes make guesses about which instructions it will run, which may not necessarily always be correct.
This does however seem unlikely, as WRPKRU is serializing, and any Meltdown-immune CPU will not even be able to fetch cache lines from different protection keys until _after_ the WRPKRU.
V2 might require RSB filling or IBPB depending on hardware.
This question can possibly be generalized into other types of side channels, cf. [4].

So far, this RFC has (implicitly) only discussed non-SMP systems, but the PKRU register is separate for each hardware thread.
Thus, there are multiple possible ways to extend this to SMP.
For example, the tags assigned to coprocesses could be allocated symmetrically, where each coprocesses could consist of multiple threads, as would be expected for simple processes.
Alternatively, if all coprocesses are singlethreaded, it would be possible to assign one CPU-tag pair to each coprocess, increasing the set size.

Initially it would be far simpler if coprocess sets were allocated statically, either predefined or by inheriting the coprocess set after fork/execv until the number of PKEYs are exhausted. Allocating those dynamically would also be an option, perhaps by statically partitioning the address space (43 or 52 bits on x86) by dividing the 512-entry PML4/PML5 array into 32 PML4s/PML5s per coprocess.
This would allow coprocesses to jump between coprocess sets, possibly dynamically.
Unfortunately, this would vastly increase the complexity of the TLB shootdown logic, even though it would be possible to lazily allow an old coprocess to remain mapped even after it has moved.

Some kernels, including Linux, supports _core scheduling_ where sibling hyperthreads never run in different address spaces.
This can have performance advantages due to the internal cache and queues often shared between hyperthreads.
If keys are assigned nonsymmetrically to hardware threads, this should be taken into account.

There is a similar less commonly available Kernel Memory Protection Key feature, using the IA32_PKRS MSR.
This could similarly allow e.g. lightweight kernel modules, to for example allow dymamically loading schedulers, or other parts of the kernel where it would make sense to dynamically replace implementations.

# References

1. Vahldiek-Oberwagner A, et al. (2019). ERIM: Secure, Efficient In-process Isolation with Protection Keys (MPK), _Proceedings of the 28th USENIX Security Symposium_. https://www.usenix.org/conference/usenixsecurity19/presentation/vahldiek-oberwagner

2. Schrammel D, et al. (2020). Secure, Efficient In-process Isolation with Protection Keys (MPK), _Proceedings of the 29th USENIX Security Symposium._ https://www.usenix.org/conference/usenixsecurity20/presentation/schrammel

3. Sung M (2020). Intra-unikernel isolation with Intel memory protection keys, _Proceedings of the 16th ACM SIGPLAN/SIGOPS International Conference on Virtual Execution Environment_. https://doi.org/10.1145/3381052.3381326

4. Shapiro J S (2003). Vulnerabilities in Synchronous IPC Designs, _IEEE Symposium on Security and Privacy_. https://srl.cs.jhu.edu/courses/600.439/shap03vulnerabilities.pdf
