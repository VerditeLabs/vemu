Gamozo Labs Blog
About
Vectorized Emulation: Hardware accelerated taint tracking at 2 trillion instructions per second
Oct 14, 2018

This is the introduction of a multipart series. It is to give a high level overview without really deeply diving into any individual component.

Read the next post in the series: MMU Design
Vectorized emulation, why do I do this to myself?

Why

Changelog
Date	Info
2018-10-14	Initial
Tweeter
Follow me at @gamozolabs on Twitter if you want notifications when new blogs come up, or I think you can use RSS or something if you’re still one of those people.

Performance disclaimer
All benchmarks done here are on a single Xeon Phi 7210 with 96 GiB of RAM. This comes out to about $4k USD, but if you cheap out on RAM and buy used Phis you could probably get the same setup for $1k USD.

This machine has 64 cores and 256 hardware threads. Using AVX-512 I run 4096 32-bit VMs at a time ((512 / 32) * 256).

All performance numbers in this article refer to the machine running at 100% on all cores.

Terminology
Term	Inology
Lane	A single component in a larger vector (often 32-bit throughout this document)
VM	A single VM, in terms of vectorized emulation it refers to a single lane of a vector
Intro
In this blog I’m going to introduce you to a concept I’ve been working on for almost 2 years now. Vectorized emulation. The goal is to take standard applications and JIT them to their AVX-512 equivalent such that we can fuzz 16 VMs at a time per thread. The net result of this work allows for high performance fuzzing (approx 40 billion to 120 billion instructions per second [the 2 trillion clickbait number is theoretical maximum]) depending on the target, while gathering differential coverage on code, register, and memory state.

By gathering more than just code coverage we are able to track state of code deeper than just code coverage itself, allowing us to fuzz through things like memcmp() without any hooks or static analysis of the target at all.

Further since we’re running emulated code we are able to run a soft MMU implementation which has byte-level permissions. This gives us stronger-than-ASAN memory protections, making bugs fail faster and cleaner.

How it came to be an idea
My history with fuzzing tools starts off effectively with my hypervisor for fuzzing, falkervisor. falkervisor served me well for quite a long time, but my work rotated more towards non-x86 targets, which it did not support. With a demand for emulation I made modifications to QEMU for high-performance fuzzing, and ultimately swapped out their MMU implementation for my own which has byte-level permissions. This new byte-level permission model allowed me to catch even the smallest memory corruptions, leading to finding pretty fun bugs!

More and more after working with QEMU I got annoyed. It’s designed for whole systems yet I was using it for fuzzing targets that were running with unknown hardware and running from dynamically dumped memory snapshots. Due to the level of abstraction in QEMU I started to get concerned with the potential unknowns that would affect the instrumentation and fuzzing of targets.

I developed my first MIPS emulator. It was not designed for performance, but rather purely for simple usage and perfect single stepping. You step an instruction, registers and memory get updated. No JIT, no intermediate registers, no flushing or weird block level translation changes. I eventually made a JIT for this that maintained the flush-state-every-instruction model and successfully used it against multiple targets. I also developed an ARM emulator somewhere in this timeframe.

When early 2017 rolls around I’m bored and want to buy a Xeon Phi. Who doesn’t want a 64-core 256-thread single processor? I really had no need for the machine so I just made up some excuse in my head that the high bandwidth memory on die would make reverting snapshots faster. Yeah… like that really matters? Oh well, I bought it.

While the machine was on the way I had this idea… when fuzzing from a snapshot all VMs initially start off fuzzing with the exact same state, except for maybe an input buffer and length being changed. Thus they do identical operations until user-controlled data is processed. I’ve done some fun vectorization work before, but what got me thinking is why not just emit vpaddd instead of add when JITting, and now I can run 16 VMs at a time!

Alas… the idea was born

A primer on snapshot fuzzing
Snapshot fuzzing is fundamental to this work and almost all fuzzing work I have done from 2014 and beyond. It warrants its own blog entirely.

Snapshot fuzzing is a method of fuzzing where you start from a partially-executed system state. For example I can run an application under GDB, like a parser, put a breakpoint after the file/network data has been read, and then dump memory and register state to a core dump using gcore. At this point I have full memory and register state for the application. I can then load up this core dump into any emulator, set up memory contents and permissions, set up register state, and continue execution. While this is an example with core dumps on Linux, this methodology works the same whether the snapshot is a core dump from GDB, a minidump on Windows, or even an exotic memory dump taken from an exploit on a locked-down device like a phone.

All that matters is that I have memory state and register state. From this point I can inject/modify the file contents in memory and continue execution with a new input!

It can get a lot more complex when dealing with kernel state, like file handles, network packets buffered in the kernel, and really anything that syscalls. However in most targets you can make some custom rigging using strace to know which FDs line up, where they are currently seeked, etc. Further a full system snapshot can be used instead of a single application and then this kernel state is no longer a concern.

The benefits of snapshot fuzzing are performance (linear scaling), high levels of introspection (even without source or symbols), and most importantly… determinism. Unless the emulator has bugs snapshot fuzzing is typically deterministic (sometimes relaxed for performance). Find some super exotic race condition while snapshot fuzzing? Well, you can single step through with the same input and now you can look at the trace as a human, even if it’s a 1 in a billion chance of hitting.

A primer on vectorized instruction sets
Since the 90s many computer architectures have some form of SIMD (vectorized) instruction set. SIMD stands for single instruction multiple data. This means that a single instruction performs an operation (typically the same) on multiple different pieces of data. SIMD instruction sets fall under names like MMX, SSE, AVX, AVX512 for x86, NEON for ARM, and AltiVec for PPC. You’ve probably seen these instructions if you’ve ever looked at a memcpy() implementation on any 64-bit x86 system. They’re the ones with the gross 15 character mnemonics and registers you didn’t even know existed.

For a simple case lets talk about standard SSE on x86. Since x86_64 started with the Pentium 4 and the Pentium 4 had up to SSE3 implementations, almost any x86_64 compiler will generate SSE instructions as they’re always valid on 64-bit systems.

SSE provides 128-bit SIMD operations to x86. SSE introduced 16 128-bit registers named xmm0 through xmm15 (only 8 xmm registers on 32-bit x86). These 128-bit registers can be treated as groups of different sized smaller pieces of data which sum up to 128 bits.

4 single precision floats
2 double precision floats
2 64-bit integers
4 32-bit integers
8 16-bit integers
16 8-bit integers
Now with a single instruction it is possible to perform the same operation on multiple floats or integers. For example there is an instruction paddd, which stands for packed add dwords. This means that the 128-bit registers provided are treated as 4 32-bit integers, and an add operation is performed.

Here’s a real example, adding xmm0 and xmm1 together treating them as 4 individual 32-bit integer lanes and storing them back into xmm0

paddd xmm0, xmm1

Register	Dword 1	Dword 2	Dword 3	Dword 4
xmm0	5	6	7	8
xmm1	10	20	30	40
xmm0 (result)	15	26	37	48
Cool. Starting with AVX these registers were expanded to 256-bits thus allowing twice the throughput per instruction. These registers are named ymm0 through ymm15. Further AVX introduced three operand form instructions which allow storing a result to a different register than the ones being used in the operation. For example you can do vpaddd ymm0, ymm1, ymm2 which will add the 8 individual 32-bit integers in ymm1 and ymm2 and store the result into ymm0. This helps a lot with register scheduling and prevents many unnecessary movs just to save off registers before they are clobbered.

AVX-512
AVX-512 is a continuation of x86’s SIMD model by expanding from 16 256-bit registers to 32 512-bit registers. These registers are named zmm0 through zmm31. Further AVX-512 introduces 8 new kmask registers named k0 through k7 where k0 has a special meaning.

The kmask registers are used to perform masking on instructions, either by merging or zeroing. This makes it possible to loop through data and process it while having conditional masking to disable operations on a given lane of the vector.

The syntax for the common instructions using kmasks are the following:

vpaddd zmm0 {k1}, zmm1, zmm2

chart simplified to show 4 lanes instead of 16

Register	Dword 1	Dword 2	Dword 3	Dword 4
zmm0	9	9	9	9
zmm1	1	2	3	4
zmm2	10	20	30	40
k1	1	0	1	1
zmm0 (result)	11	9	33	44
or

vpaddd zmm0 {k1}{z}, zmm1, zmm2

chart simplified to show 4 lanes instead of 16

Register	Dword 1	Dword 2	Dword 3	Dword 4
zmm0	9	9	9	9
zmm1	1	2	3	4
zmm2	10	20	30	40
k1	1	0	1	1
zmm0 (result)	11	0	33	44
The first example uses k1 as the kmask for the add operation. In this case the k1 register is treated as a 16-bit number, where each bit corresponds to each of the 16 32-bit lanes in the 512-bit register. If the corresponding bit in k1 is zero, then the add operation is not performed and that lane is left unchanged in the resultant register.

In the second example there is a {z} suffix on the kmask register selection, this means that the operation is performed with zeroing rather than merging. If the corresponding bit in k1 is zero then the resultant lane is zeroed out rather than left unchanged. This gets rid of a dependency on the previous register state of the result and thus is faster, however it might not be suitable for all applications.

The k0 mask is implicit and does not need to be specified. The k0 register is hardwired to having all bits set, thus the operation is performed on all lanes unconditionally.

Prior to AVX-512 compare instructions in SIMD typically yielded all ones in a given lane if the comparision was true, or all zeroes if it was false. In AVX-512 comparison instructions are done using kmasks.

vpcmpgtd k2 {k1}, zmm10, zmm11

You may have seen this instruction in the picture at the start of the blog. What this instruction does is compare the 16 dwords in zmm10 with the 16 dwords in zmm11, and only performs the compare on lanes enabled by k1, and stores the result of the compare into k2. If the lane was disabled due to k1 then the corresponding bit in the k2 result will be zero. Meaning the only set bits in k2 will be from enabled lanes which were greater in zmm10 than in zmm11. Phew.

Vectorized emulation
Now that you’ve made it this far you might already have some gears turning in your head telling you where this might be going next.

Since with snapshot fuzzing we start executing the same code, we are doing the same operations. This means we can convert the x86 instructions to their vectorized counterparts and run 16 VMs at a time rather than just one.

Let’s make up a fake program:

mov eax, 5
mov ebx, 10
add eax, ebx
sub eax, 20
How can we vectorize this code?

; Register allocation:
; eax = zmm0
; ebx = zmm1

vpbroadcastd zmm0, dword ptr [memory containing constant 5]
vpbroadcastd zmm1, dword ptr [memory containing constant 10]
vpaddd       zmm0, zmm0, zmm1
vpsubd       zmm0, zmm0, dword ptr [memory containing constant 20] {1to16}
Well that was kind of easy. We’ve got a few new AVX concepts here. We’re using the vpbroadcastd instruction to broadcast a dword value to all lanes of a given ZMM register. Since the Xeon Phi is bottlenecked on the instruction decoder it’s actually faster to load from memory than it is to load an immediate into a GPR, move this into a XMM register, and then broadcast it out.

Further we introduce the {1to16} broadcasting that AVX-512 offers. This allows us to use a single dword constant value with in our example vpsubd. This broadcasts the memory pointed to to all 16 lanes and then performs the operation. This saves one instruction as we don’t need an explicit vpbroadcastd.

In this case if we executed this code with any VM state we will have no divergence (no VMs do anything different), thus this example is very easy. It’s pretty much a 1-to-1 translation of the non-vectorized x86 to vectorized x86.

Alright, let’s try one a bit more complex, this time let’s work with VMs in different states:

add eax, 10
becomes

; Register allocation:
; eax = zmm0

vpaddd zmm0, zmm0, dword ptr [memory containing constant 10] {1to16}
Let’s imagine that the value in eax prior to execution is different, let’s say it’s [1, 2, 3, 4] for 4 different VMs (simplified, in reality there are 16).

Register	Dword 1	Dword 2	Dword 3	Dword 4
zmm0	1	2	3	4
const	10	10	10	10
zmm0 (result)	11	12	13	14
Oh? This is exactly what AVX is supposed to do… so it’s easy?

Okay it’s not that easy
So you might have noticed we’ve dodged a few things here that are hard. First we’ve ignored memory operations, and second we’ve ignored branches.

Lets talk a bit about AVX memory
With AVX-512 we can load and store directly from/to memory, and ideally this memory is aligned as 512-bit registers are whole 64-byte cache lines. In AVX-512 we use the vmovdqa32 instruction. This will load an entire aligned 64-byte piece of memory into a ZMM register ala vmovdqa32 zmm0, [memory], and we can store with vmovdqa32 [memory], zmm0. Further when using kmasks with vmovdqa32 for loads the corresponding lane is left unmodified (merge masking) or zeroed (zero masking). For stores the value is simply not written if the corresponding mask bit is zero.

That’s pretty easy. But this doesn’t really work well when we have 16 unique VMs we’re running with unique address spaces.

… or does it?

VM memory interleaving
Since most VM memory operations are not affected by user input, and thus are the same in all VMs, we need a way to organize the 16 VMs memory such that we can access them all quickly. To do this we actually interleave all 16 VMs at the dword level (32-bit). This means we can perform a single vmovdqa32 to load or store to memory for all 16 VMs as long as they’re accessing the same address.

This is pretty simple, just interleave at the dword level:

chart simplified to show 4 lanes instead of 16

Guest Address	Host Address	Dword 1	Dword 2	Dword 3	…	Dword 16
0x0000	0x0000	1	2	3	…	33
0x0004	0x0040	32	74	55	…	45
0x0008	0x0080	24	24	24	…	24
All we need to do is take the guest address, multiply it by 16, and then vmovdqa32 from/to that address. It once again does not matter what the contents of the memory are for each VM and they can differ. The vmovdqa32 does not care about the memory contents.

In reality the host address is not just the guest address multiplied by 16 as we need some translation layer. But that will get it’s own entire blog. For now let’s just assume a flat, infinite memory model where we can just multiply by 16.

So what are the limitations of this model?

Well when reading bytes we must read the whole dword value and then shift and mask to extract the specific byte. When writing a byte we need to read the memory first, shift, mask, and or in the new byte, and write it out. And when doing non-aligned operations we need to perform multiple memory operations and combine the values via shifting and masking. Luckily compilers (and programmers) typically avoid these unaligned operations and they’re rare enough to not matter much.

Divergence
So far everything we have talked about does not care about the values it is operating on at all, thus everything has been easy so far. But in reality values do matter. There are 3 places where divergence matters in this entire system:

Loads/stores with different addresses
Branches
Exceptions/faults
Loads/stores with different addresses
Let’s knock out the first one real quick, loads and stores with different addresses. For all memory accesses we do a very quick horizontal comparison of all the lanes first. If they have the same address then we take a fast path and issue a single vmovdqa32. If their addresses differ than we simply perform 16 individual memory operations and emulate the behavior we desire. It technically can get a bit better as AVX-512 has scatter/gather instructions which allow the CPU to do this load/storing to different addresses for us. This is done with a base and an offset, with 32-bits it’s not possible to address the whole address space we need. However with 64-bit vectorization (8 64-bit VMs) we can leverage scatter/gather instructions to their fullest and all loads and stores just become a fast path with one vmovdqa32, or a slow (but fast) path where we use a single scatter/gather instruction.

Branches
We’ve avoided this until now for a reason. It’s the single hardest thing in vectorized emulation. How can we possibly run 16 VMs at a time if one branches to another location. Now we cannot run a AVX-512 instruction as it would be invalid for the VMs which have gone down a different path.

Well it turns out this isn’t a terribly hard problem on AVX-512. And when I say AVX-512 I mean specifically AVX-512. Feel free to ponder why this might be based on what you’ve learned is unique to AVX-512.

…

…

…

Okay it’s kmasks. Did you get it right? Well kmasks save our lives. Remember the merging kmasks we talked about which would disable updates to a given lane of a vector and ignore writes to a given lane if it is not enabled in the kmask?

Well by using a kmask register on all JITted AVX-512 instructions we can simply change the kmask to disable updates on a given VM.

What this allows us to do is start execution at the same location on all 16 VMs as they start with the same EIP. On all branches we will horizontally compare the branch targets and compute a new kmask value to use when we continue execution on the new branch.

AVX-512 doesn’t have a great way of extracting or broadcasting arbitrary elements of a vector. However it has a fast way to broadcast the 0th lane in a vector ala vpbroadcastd zmm0, xmm0. This takes the first lane from xmm0 and broadcasts it to all 16 lanes in zmm0. We actually never stop following VM #0. This means VM #0 is always executing, which is important for all of the horizontal compares that we talk about. When I say horizontal compare I mean a broadcast of the VM#0 and compare with all other VMs.

Let’s look in-detail at the entire JIT that I use for conditional indirect branches:

; IL operation is Beqz(val, true_target, false_target)
;
; val          - 16 32-bit values to conditionally branch by
; true_target  - 16 32-bit guest branch target addresses if val == 0
; false_target - 16 32-bit guest branch target addresses if val != 0
;
; IL pseudocode:
;
; if val == 0 {
;    goto true_target;
; } else {
;    goto false_target;
; }
;
; Register usage
; k1    - The execution kmask, this is the kmask used on all JITted instructions
; k2    - Temporary kmask, just used for scratch
; val   - Dynamically allocated zmm register containing val
; ttgt  - Dynamically allocated zmm register containing true_target
; ftgt  - Dynamically allocated zmm register containing false_target
; zmm0  - Scratch register
; zmm31 - Desired branch target for all lanes

; Compute a kmask `k2` which contains `1`s for the corresponding lanes
; for VMs which are enabled by `k1` and also have a non-zero value.
; TL;DR: k2 contains a mask of VMs which will be taking `ftgt`
vptestmd k2 {k1}, val, val

; Store the true branch target unconditionally, while not clobbering
; VMs which have been disabled
vmovdqa32 zmm31 {k1}, ttgt

; Store the false branch target for VMs not taking the branch
; Note the use of k2
vmovdqa32 zmm31 {k2}, ftgt

; At this point `zmm31` contains the targets for all VMs. Including ones
; that previously got disabled.

; Broadcast the target that VM #0 wants to take to all lanes in `zmm0`
vpbroadcastd zmm0, xmm31

; Compute a new kmask of which represents all VMs which are going to
; the same location as VM #0
vpcmpeqd k1, zmm0, zmm31

; ...
; Now just rip out the target for VM #0 and translate the guest address
; into the host JIT address and jump there.
; Or break out and generate the JIT if it hasn't been hit before
The above code is quite fast and isn’t a huge performance issue, especially as we’re running 16 VMs at a time and branches are “rare” with respect to expensive operations like memory loads and stores.

One thing that is important to note is that zmm31 always contains the last desired branch target for a given VM. Even after it has been disabled. This means that it is possible for a VM which has been disabled to come back online if VM #0 ends up going to the same location.

Lets go through a more thorough example:

; Register allocation:
; ebx - Pointer to some user controlled buffer
; ecx - Length of controlled buffer

; Validate buffer size
cmp ecx, 4
jne .end

; Fallthrough
.next:

; Check some magic from the buffer
cmp dword ptr [ebx], 0x13371337
jne .end

; Fallthrough
.next2:

; Conditionally jump to end, for clarity
jmp .end

.end:
And the theoretical vectorized output (not actual JIT output):

; Register allocation:
; zmm10 - ebx
; zmm11 - ecx
; k1    - The execution kmask, this is the kmask used on all JITted instructions
; k2    - Temporary kmask, just used for scratch
; zmm0  - Scratch register
; zmm8  - Scratch register
; zmm31 - Desired branch target for all lanes

; Compute kmask register for VMs which have `ecx` == 4
vpcmpeqd k2 {k1}, zmm11, dword ptr [memory containing 4] {1to16}

; Update zmm31 to reference the respective branch target
vmovdqa32 zmm31 {k1}, address of .end  ; By default we go to end
vmovdqa32 zmm31 {k2}, address of .next ; If `ecx` == 4, go to .next

; Broadcast the target that VM #0 wants to take to all lanes in `zmm0`
vpbroadcastd zmm0, xmm31

; Compute a new kmask of which represents all VMs which are going to
; the same location as VM #0
vpcmpeqd k1, zmm0, zmm31

; Branch to where VM #0 is going (simplified)
jmp where_vm0_wants_to_go

.next:

; Magicially load memory at ebx (zmm10) into zmm8
vmovdqa32 zmm8, complex_mmu_operation_and_stuff

; Compute kmask register for VMs which have packet contents 0x13371337
vpcmpeqd k2 {k1}, zmm8, dword ptr [memory containing 0x13371337] {1to16}

; Go to .next2 if memory is 0x13371337, else go to .end
vmovdqa32 zmm31 {k1}, address of .end   ; By default we go to end
vmovdqa32 zmm31 {k2}, address of .next2 ; If contents == 0x13371337 .next2

; Broadcast the target that VM #0 wants to take to all lanes in `zmm0`
vpbroadcastd zmm0, xmm31

; Compute a new kmask of which represents all VMs which are going to
; the same location as VM #0
vpcmpeqd k1, zmm0, zmm31

; Branch to where VM #0 is going (simplified)
jmp where_vm0_wants_to_go

.next2:

; Everyone still executing is unconditionally going to .end
vmovdqa32 zmm31 {k1}, address of .end

; Broadcast the target that VM #0 wants to take to all lanes in `zmm0`
vpbroadcastd zmm0, xmm31

; Compute a new kmask of which represents all VMs which are going to
; the same location as VM #0
vpcmpeqd k1, zmm0, zmm31

.end:
Okay so what does the VM state look like for a theoretical version (simplified to 4 VMs):

Starting state, all VMs enabled with different memory contents (pointed to by ebx) and different packet lengths:

Register	VM 0	VM 1	VM 2	VM 3
ecx	4	3	4	4
memory	0x13371337	0x13371337	3	0x13371337
K1	1	1	1	1
First branch, all VMs with ecx != 4 are disabled and are pending branches to .end, VM #1 falls off

Register	VM 0	VM 1	VM 2	VM 3
ecx	4	3	4	4
memory	0x13371337	0x13371337	3	0x13371337
K1	1	0	1	1
Zmm31	.next	.end	.next	.next
Second branch, VMs without 0x13371337 in memory are pending branches to .end, VM #2 falls off

Register	VM 0	VM 1	VM 2	VM 3
ecx	4	3	4	4
memory	0x13371337	0x13371337	3	0x13371337
K1	1	0	0	1
Zmm31	.next2	.end	.end	.next2
Final branch, everyone ends up at .end, all VMs are enabled again as they’re following VM #0 to .end

Register	VM 0	VM 1	VM 2	VM 3
ecx	4	3	4	4
memory	0x13371337	0x13371337	3	0x13371337
K1	1	1	1	1
Zmm31	.end	.end	.end	.end
Branch summary
So we saw branches will disable VMs which do not follow VM #0. When VMs are disabled all modifications to their register states or memory states are blocked by hardware. The kmask mechanism allows us to keep performance up and not use different JITs based on different branch states.

Further, VMs can come back online if they were pending to go to a location which VM #0 eventually ends up going to.

Exceptions/faults
These are really just glorified branches with a VM exit to save the input and memory/register state related to the crash. No reason to really go in depth here.

Okay but why?
Okay we’ve covered all the very high level details of how vectorized emulation is possible but that’s just academic thought. It’s pointless unless it accomplishes something.

At this point all of the next topics are going to be their own blogs and thus are only lightly touched on

Differential coverage / Hardware accelerated taint tracking
Differential coverage is a special type of coverage that we are able to gather with this vectorized emulation model. This is the most important aspect of all of this tooling and is the main reason it is worth doing.

Since we are running 16 VMs at a time we are able to very cheaply (a few cycles) do a horizontal comparison with other VMs. Since VMs are deterministic and only have differing user-controlled inputs any situation where VMs have different branches, different register states, different memory states, etc is when the user input directly or indirectly caused a change in behavior.

I would consider this to be the holy grail of coverage. Any affect the input has on program state we can easily and cheaply detect.

How differential coverage combats state explosion
If we wanted to track all register states for all instructions the state explosion would be way too huge. This can be somewhat capped by limiting the amount of state each instruction can generate. For example instead of storing all unique register values for an instruction we could simply store the minimums and maximums, or store up to n unique values, etc. However even when limited to just a few values per instruction, the state explosion is too large for any real application.

However, since most memory and register states are not influenced by user input, with differential coverage we can greatly reduce the amount of instructions which state is stored on as we only store state that was influenced by user data.

This works for code coverage as well, for example if we hit a printf with completely uncontrolled parameters that would register as potentially hundreds of new blocks of coverage. With differential coverage all of this state can be ignored.

How differential coverage is great for performance
While the focus of this tool is not performance, the performance costs of updating databases on every instruction is not feasible. By filtering only instructions which have user-influenced data we’re able to perform much more complex operations in the case that new coverage was detected.

For example all of my register loads and stores start with a horizontal compare and a quick jump out if they all match. If one differs it’s a rare enough occasion that it’s feasible to spend a few more cycles to do a hash calculation based on state and insertion into the global input and coverage databases. Without differential coverage I would have to unconditionally do this every instruction.

Soft MMU
Since the soft MMU deserves a blog entirely on it’s own, we’ll just go slightly into the details.

As mentioned before, we interleave memory at the dword level, but for every byte there is also a corresponding permission byte. In memory this looks like 16 32-bit dwords representing the permissions, followed by 16 32-bit dwords containing their corresponding memory contents. This allows me to read a 64-byte cache line with the permissions which are checked first, followed by reading the 64-byte cache line directly following with the contents.

For permissions: the read, write, and execute bits are completely separate. This allows more exotic memory models like execute-only memory.

Since permissions are at the byte level, this means we can punch a one-byte hole anywhere in memory and accessing that byte would cause a fault. For some targets I’ll do special modifications to permissions and punch holes in unused or padding fields of structures to catch overflows of buffers contained inside structures.

Further I have a special read-after-write (RaW) bit, which is used to mark memory as uninitialized. Memory returned from allocators is marked as RaW and thus will fault if ever read before written to. This is tracked at the byte level and is one of the most useful features of the MMU. We’ll talk about how this can be made fast in a subsequent blog.

Performance
Performance is not the goal of this project, however the numbers are a bit better than expected from the theorycrafting.

In reality it’s possible to hit up to 2 trillion emulated instructions per second, which is the clickbait title of this blog. However this is on a 32-deep unrolled loop that is just adding numbers and not hitting memory. This unrolling makes the branch divergence checking costs disappear, and integer operations are almost a 1-to-1 translation into AVX-512 instructions.

For a real target the numbers are more in the 40 billion to 120 billion emulated instructions per second range. For a real target like OpenBSD’s DHCP client I’m able to do just over 5 million fuzz cases per second (fuzz case is one DHCP transaction, typically 1 or 2 packets). For this specific target the emulation speed is 54 billion instructions per second. This is while gathering PC-level coverage and all register and memory divergence coverage.

So it’s just academic?
I’ve been working on this tooling for almost 2 years now and it’s been usable since month 3. It’s my primary tool for fuzzing and has successfully found bugs in various targets. Sadly most of these bugs are not public yet, but soon.

This tool was used to find a remote bluescreen in Windows Firewall: CVE-2018-8206 (okay technically I found it first manually, but was able to find it with the fuzzer with a byte flipper even though it has much more complex constraints)

It was also used to find a theoretical OOB in OpenBSD’s dhclient: dhclient bug . This is a fun one as really no tradtional fuzzer would find this as it’s an out-of-bounds by 1 inside of a structure.

Future blogs
Description of the IL used, as it’s got some specific designs for vectorized emulation

Internal details of the MMU implementation

Showing the power of differential coverage by looking a real example of fuzzing an HTTP parser and having a byte flipper quickly (under 5 seconds) find the basic “VERB HTTP/number.number\r\n". No magic, no `strings` feedback, no static analysis. Just a useless fuzzer with strong harnessing.

Talk about the new IL which handles graphs and can do cross-block optimizations

Showing better branch divergence handling via post-dominator analysis and stepping VMs until they sync up at a known future merge point

Gamozo Labs Blog
Gamozo Labs Blog
b@gamozolabs.com
 gamozolabs
 gamozolabs
I blog about random things security, everything is broken, nothing scales, shared memory models are flawed.

Gamozo Labs Blog
About
Vectorized Emulation: MMU Design
Nov 19, 2018

Softserve

New vectorized emulator codenamed softserve

Tweeter
Follow me at @gamozolabs on Twitter if you want notifications when new blogs come up.

Check out the intro
This is the continuation of a multipart series. See the introduction post here

This post assumes you’ve read the intro and doesn’t explain some of the basics of the vectorized emulation concept. Go read it if you haven’t!

Further this blog is a lot more technical than the introduction. This is meant to go deep enough to clear up most/all potential questions about the MMU. It expects that you have a general knowledge of page tables and virtual addressing models. Hopefully we do a decent job explaining these things such that it’s not a hard requirement!

The code
This blog explains the intent behind a pretty complex MMU design. The code that this blog references can be found here. I have no plans to open source the vectorized emulator and this MMU is just a snapshot of what this blog is explaining. I have no intent to update this code as I change my MMU model. Further this code is not buildable as I’m not open sourcing my assembler, however I assume the syntax is pretty obvious and can be read as pseudocode.

By sharing this code I can talk at a higher level and allow the nitty-gritty details to be explained by the actual implementation.

It’s also important to note that this code is not being used in production yet. It’s got room for micro-optimizations and polish. At least it should be doing the correct operations and hopefully the tests are verifying this. Right now I’m trying to keep it simple to make sure it’s correct and then polish it later using this version as reference.

Intro
Today we’re going to talk about the internals of the memory management unit (MMU) design I have used in my vectorized emulator. The MMU is responsible for creating the fake memory environment of the VMs that run under the emulator. Further the MMU design used here also is designed to catch bugs as early as possible. To do this we implement what I call a “byte-level MMU”, where each byte has it’s own permission bits. Since vectorized emulation is meant for fuzzing it’s also important that the memory state can quickly be restored to the original state quickly so a new fuzz iteration can be started.

During this blog we introduce a few big concepts:

Differential restores
Byte-level permissions
Read-after-write memory (uninitialized memory tracking)
Gage fuzzing
Aliased/CoW memory
Deduplicated memory
Technical details about the IL relevant to the MMU
Painful details about everything
Since this emulator design is meant to run multiple architectures and programs in different environments, it’s critical the MMU design supports a superset of the features of all the programs I may run. For example, system processes typically are run in the high memory ranges 0xffff... and above. Part of the design here is to make sure that a full guest address space can be used, including high memory addresses. Things like x86_64 have 48-bit address spaces, where things like ARM64 have 49-bit address spaces (2 separate 48-bit address spaces). Thus to run an ARM64 target on x86 I need to provide more bits than actually present. Luckily most systems use this address space sparsely, so by using different data structures we can support emulating these targets with ease.

The problem
Before we get into describing the solution, let’s address what the problem is in the first place!

When creating an emulator it’s important to create isolation between the emulated guest and the actual system. For example if the guest accesses memory, it’s important that it can only access it’s own memory, and it isn’t overwriting the emulator’s memory itself. To do this there are multiple traditional solutions:

Restrict the address space of the guest such that it can fit entirely in the emulator’s address space
Use a data structure to emulate a sparse guest’s memory space
Create a new process/VM with only the guest’s memory mapped in
The first solution is the simplest, fastest, but also the least portable. It typically consists of allocating a buffer the size of the guest’s address space, and then any guest memory accesses are added to the base of this buffer and ensured to not go out of bounds. A model like this can rely on the hardware’s permission checking by setting permissions via mmap or VirtualProtect. This is an extremely fast model and allows for running applications that fit inside of the emulator’s address space. When running a 64-bit VM this can become tough as most OSes do not provide a means of allocating memory in the high part of the address space 0xffff... and beyond. This memory is typically reserved for the kernel. This is the model used by things like qemu-user as it is super fast and works great for well-behaving userland applications. By setting the QEMU_GUEST_BASE environment variable you can change this base and set the size with QEMU_RESERVED_VA.

The second solution is fairly slow, but allows for more strict memory permissions than the host system allows. Typically the data structure used to access the guest’s memory is similar to traditional page table models used in hardware. However since it’s implemented in software it’s possible to change these page tables to contain any metadata or sizes as desired. This is the model I ultimately use, but with a few twists from traditional page tables.

The third solution leverages something like VT-x or a thin process to almost directly use the target hardware’s page table models for a VM. This will make the emulator tied to an architecture, might require a driver, and like the first solution, doesn’t allow for stricter memory models. This is actually one of the first models I used in my emulator and I’ll go into it a bit more in the history section

History
Feel free to skip this section if you don’t care about context

To give some background on how we ended up where we ended up it’s important to go through the background of the MMU designs used in the past. Note that the generations aren’t the same MMU improving, it’s just different MMUs I’ve designed over time.

First generation
The first generation of my MMU was a simple modification to QEMU to allow for quick tracking of which memory was modified. In this case my target was a system level target so I was not using qemu-user, but rather qemu-system. I ripped out the core physical memory manager in QEMU and replaced it with my own that effectively mimicked the x86 page table model. I was most comfortable with the x86 page table model and since it was implemented in hardware I assumed it was probably well engineered. The only interest I had in this first MMU was to quickly gather which memory was modified so I could restore only the dirtied memory to save time during reset time. This had a huge improvement for my hypervisor so it was natural for me to just copy it over the QEMU so I could get the same benefits.

Second generation
While still continuing on QEMU modifications I started to get a bit more creative. Since I was handling all the physical memory accesses directly in software, there was no reason I couldn’t use page tables of my own shape. I switched to using a page table that supported 32-bit addresses (my target was MIPS32 and ARM32) using 8-bits per table. This gave me 256-byte pages rather than traditional 4-KiB x86 pages and allowed me to reset more specific dirty pages and reduces the overall work for resets.

Third generation
At this point I was tinkering around with different page table shapes to find which worked fast. But then I realized I could set the final translation page size to 1-byte and I would be able to apply permissions to any arbitrary location in memory. Since memory of the target system was still using 4-KiB pages I wasn’t able to apply byte-level permissions in the snapshotted target, however I was able to apply byte-level permissions to memory returned from hooked functions like malloc(). By setting permissions directly to the size actually requested by malloc() we could find 1-byte out-of-bounds memory accesses. This ended up finding a bug which was only slightly out-of-bounds (1 or 2 bytes), and since this was now a crash it was prioritized for use in future fuzz cases. This prioritization (or feedback) eventually ended up with the out-of-bounds growing to hundreds of bytes, which would crash even on an actual system.

Fourth generation
I ended up designing my own emulator for MIPS32, performance wasn’t really the focus. I basically copied the model I used for the 3rd generation. I also kept the 1-byte paging as by this point it was a potent tool in my toolbag. However once I switched this emulator to use JIT I started to run into some performance issues. This caused me to drop the emulated page tables and byte level permissions and switch to a direct-memory-access model.

At this time I was doing most of my development for my emulator to run directly on my OS. Since my OS didn’t follow any traditional models this allowed me to create a user-land application with almost any address space as I wanted. I directly used the MMU of the hardware to support running my JIT in the context of a different address space. In this model the JITted code just directly accessed memory, which except for a few pages in the address space, was just the exact same as the actual guest’s address space.

For example if the guest wanted to access address 0x13370000, it would just directly dereference the memory at 0x1337000. No translation, not base applied, simple.

You can see this code in the srcs/emu folder in falkervisor.

I used this model for a long time as it was ideal for performance and didn’t really restrict the guest from any unique address space shapes. I used this memory model in my vectorized emulator for quite a while as well, but with a scale applied to the address as I interleaved multiple VM’s memory.

Fifth generation
The vectorized emulator was initially designed for hard targets, and the primary goal was to extract as much information as possible from the code under test. When trying to improve it’s ability to find bugs I remembered that in the past I had done a byte-level MMU with much success. I had a silly idea of how I could handle these permission checks. Since in the JIT I control what code is run when doing a read or write, I could simply JIT code to do the permission checks. I decided that I would simply have 1 extra byte for every byte of the target. This byte would be all of the permissions for the corresponding byte in the memory map (read, write, and/or execute).

Since now I needed to have 2 memory regions for this, I started to switch from using my OS and the stripped down user-land process address space to using 2 linear mappings in my process. Since this was more portable I decided to start developing my vectorized emulator to run on just Windows/Linux. On a guest memory access I would simply bounds check the address to make sure it’s in a certain range, and then add the address to the base/permission allocations. This is similar to what qemu-user does but with a permission region as well. The JIT would check these permissions by reading the permissions memory first and checking for the corresponding bits.

Sixth generation
The performance of the fifth generation MMU was pretty good for JIT, but was terrible for process start times. I would end up reserving multiple terabytes of memory for the guest address spaces. This made it take a long time to start up processes and even tear them down as they blocked until the OS cleaned up their resources. Further commit memory usage was quite high as I would commit entire 4-KiB guest pages, which were actually 128-KiB (16 vectorized VMs * 2 regions (permission and memory region) * 4 KiB). To mitigate these issues we ended up at the current design….

Page Tables
Before we hop into soft MMU design it’s important to understand what I mean when I say page tables. Page tables take some bit-slice of the address to be translated and use it as the index for an element in a first level table. This table points to another table which is then indexed by a different bit-slice of the same address. This may continue for however many levels are used in the page table. In my case the shape of this page table is dynamically configurable and we’ll go into that a bit more.

Page table

In the case of 64-bit x86 there is a 4 level lookup, where 9 bits are used for each level. This means each page table contains 512 entries. Each entry is a pointer to the next page table, or the actual page if it’s the final level. Finally the bottom 12 bits of the address are used as the offset into the page to find the specific byte. This paging model would show up as [9, 9, 9, 9, 12] according to my dynamic paging model. This syntax will be explained later.

For x86 there are alignment requirements for the page table entries (must be 4-KiB aligned). Further physical addresses are only 52-bits. This leaves 12 bits at the bottom of the page table entry and 12 bits at the top for use as metadata. x86 uses this to store information such as: If the page is present, writable, privileged, caching behavior, whether it’s been accessed/modified, whether it’s executable, etc. This metadata has no cost in hardware but in software, traversing this has a cost as the metadata must be masked off for the pointer to be extracted. This might not seem to matter but when doing billions of translations a second, the extra masking operations add up.

Here’s the actual metadata of a 4 KiB page on 64-bit Intel:

Page table metadata

The overall design
My vectorized emulator is being rewritten to be 64-bit rather than 32-bit. We’re now running 2048 VMs rather than 4096 VMs as we can only run 8 VMs per thread. All of this design is for 64-bits.

When designing the new MMU there were a few critical features it needed:

Byte level permissions
Fast snapshot/restore times
A data structure that could be quickly traversed in JIT
Quick process start times
The ability to handle full 64-bit address spaces
Low memory usage (we need to run 2048 VMs)
Quick methods for injecting fuzz inputs (we need a way to get fuzz inputs in to the memory millions of times per second)
Must be easily tested for correctness
Ability to track uninitialized memory at a byte-level
Read-only memory shared between all cores
Applying byte-level permissions
So we have this byte-level permission goal, but how do we actually get byte-level information to apply anyways?

Since most fuzzing is done from an already-existing snapshot from a real system with 4 KiB paging and permissions, we cannot just magically get byte-level permissions. We have to find locations that can be restricted to specific byte-level sizes.

The easiest way to do this is just ignore everything in the snapshot. We can apply byte-level permissions to only new memory allocations that we emulate by adding breakpoints to the target’s allocate routines. Further by hooking frees we can delete the mappings and catch use-after-frees.

We can get a bit more fancy if we’re enlightened as to the internals of the allocator of the target under test. Upon loading of the snapshot we could walk the heap metadata and trim down allocations to the byte-level sizes they originally requested. If the heap does not provide the requested size then this is not possible. Further allocations which fit perfectly in a bin might not have any room after them to place even a single guard byte.

To remedy these problems there a few solutions. We can use page heap in the application we’re taking a snapshot in, which will always ensure we have a page after the allocation we can play with for guard bytes. Further page heap has the requested size in the metadata so we can get perfect byte-level applied.

If page heap is not available for the allocator you’re gonna have to get really creative and probably replace the allocator. You could also hack it up and use a debugger to always add a few bytes to each allocation (ensuring room for guard bytes), while logging the requested sizes. This information could then be used to create a perfect byte heap.

Getting even fancier
When going at a really hard target you could also start to add guard bytes between padding fields of structures (using symbol information or compiler plugins) and globals. The more you restrict, the more you can detect.

Design features
Basics of the vectorized model
This was covered in the intro, but since it’s directly applicable to the MMU it’s important to mention here.

Memory between the different lanes on a given core is interleaved at the 8-byte level (4-byte level for 32-bit VMs). This means that when accessing the same address on all VMs we’re able to dispatch a single read at one address to load all 8 VM’s memory. This has the downside of unaligned memory accesses being much more expensive as they now require multiple loads. However the common case most memory is accessed at the same address, and memory does not straddle a 8-byte boundary. It’s worth it.

For reference the cost of a single load instruction vmovdqa64 is about 4-5 cycles, where a vpgatherqq load is 20-30 cycles. Unless memory is so frequently accessed from different addresses and straddling 8-byte boundaries it is always worth interleaving.

VM interleaving looks as follows:

chart simplified to show 4 lanes instead of 8

Guest Address	Host Address	Qword 1	Qword 2	Qword 3	…	Qword 8
0x0000	0x0000	1	2	3	…	33
0x0008	0x0040	32	74	55	…	45
0x0010	0x0080	24	24	24	…	24
This interleaves all the memory between the VMs at an 8-byte level. If a memory access straddles an 8-byte value things get quite slow but this is a rare case and we’re not too concerned about it.

How do we build a testable model?
To start off development it was important to build a good foundation that could be easily tested. To do this I tried to write everything as naive as possible to decrease the chance of mistakes. Since performance is only really required in the JIT, the Rust-level MMU routines were written cleanly and used as the reference implementation to test against. If high-performance methods were needing for modifying memory or permissions they would be supplemental and verified against the naive implementation. This set us up to be in good shape for testing!

64-bit address spaces
To support full 64-bit address spaces we are forced to use some data structure to handle memory as nothing in x86 can directly use a 64-bit address space. Page tables continue to be the design we go with here.

Since we were writing the code in a naive way, it was easy to make most of the MMU model configurable by constants in the code. For example the shape of the page tables is defined by a constant called PAGE_TABLE_LAYOUT. This is used in reality in the form: const PAGE_TABLE_LAYOUT: [u32; PAGE_TABLE_DEPTH] = [16, 16, 16, 13, 3];.

This array defines the number of bits used for translating each level in the page table, and PAGE_TABLE_DEPTH sets the number of levels in the page table. In the example above this shows that we use the top 16-bits for the first level as the index, the next 16-bits for the next level, the next 16-bits again for another level, a 13-bit level, and finally a 3-bit page size. As long as this PAGE_TABLE_LAYOUT adds up to 64-bits, contains at least 2 entries (1 level page table), and at least has a final translation size of 8-byte (like in the example), the MMU and JITs will be updated. This allows profiling to be done of a specific target and modify the page table to whatever shape works best. This also allows for changes between performance and memory usage if needed.

Fast restores
When writing my hypervisor I walked the SVM page tables looking for dirty pages to restore. On x86 there are only dirty bits on the last level of the page tables. For all other levels there’s only an ‘accessed’ bit (updated when the translation is used for any access). I would walk every entry in each page table, if it was accessed I would recurse to the next level, otherwise skip it, at the final level I would check for the dirty bit and only restore the memory if it was marked as dirty. This meant I walked the page tables for all the memory that was ever used, but only restored dirty memory. Walking the page tables caused quite a bit of cache pollution which would cause significant damage to the performance of other cores.

To speed this up I could potentially place a dirty bit on every page table level, and then I would only ever start walking a path that contains a dirty page. I used this model at some point historically, however I’ve got a better model now.

Instead of walking page tables I just now append the address to a vector when I first set a dirty bit. This means when resetting a VM I only read a linear array of addresses to restore. I still need a dirty bit somewhere so I make sure I don’t add duplicates to this list. Since I no longer need to walk page tables I only put dirty bits on the final level. This was a decision driven by actual data on real targets, it’s much faster.

If during execution I run out of entries in this dirty list I exit out of the VM with a special VM-exit status indicating this list is full. This then allows me to append this list at Rust-level to a dynamically sized allocation. Since the size of this list is tunable it would grow as needed and converge to never hitting VM-exits due to dirty list exhaustion. Further this dirty list is typically pretty tiny so the cost isn’t that high.

Interestingly Intel introduced (not sure if it’s in silicon yet) a way of getting a similar thing for VMs (this is called Page Modification Logging). The processor itself will give you a linear list of dirty pages. We do not use this as it is not supported in the processor we are using.

Permissions
On classic x86 (and really any other architecture) permissions bits are added at each level of the page table. This allows for the processor to abort a page table walk early, and also allows OSes to change permissions for large regions of memory by updating a single entry. However since we’re running multiple VM’s at the same time it’s possible each VM has different memory mapped in. To handle this we need a permission byte for each byte for each VM.

Since we can’t handle the permissions checks during the page table walk (technically could be possible if the permissions are a superset of all the VM’s permissions), we get to have a metadata-less page table walk until the final level where we store the dirty bit. This means that during a page table walk we do not need to mask off bits, we can just directly keep dereferencing.

There are currently 4 permission bits. A read bit, a write bit, an execute bit, and a RaW bit (see next section). All of these bits are completely independent. This allows for arbitrary permission sets like write-only memory, and execute-only memory.

In some older versions of my MMU I had a page table for both permissions and data. This is pretty pointless as they always have the same shape. This caused me to perform 2 page table walks for every single memory access.

In the new model I interleave the memory and permissions for the VMs such that one walk will give me access to the permissions and memory contents. Further in memory the permissions come first followed by the contents. Since permissions are checked first this allows for the memory to be accessed linearally and potentially get a speedup by the hardware prefetchers.

When permissions and contents are laid out in a pretty format it looks something like:

Simplified to 4 lanes instead of 8 MMU layout

We can see every byte of contents has a byte of permissions and the permissions come first in memory. This image displays directly how the memory looks if you were to dump the MMU region for a page as qwords.

Uninitialized memory tracking
To track uninitialized memory I introduce a new permission bit called the RaW (read-after-write) bit. This bit indicates that memory is only readable after it has been written to. In allocator hooks or by manual application to regions of memory this bit can be set and the read bit cleared.

On all writes to memory the RaW it is unconditionally copied to the read bit. It’s done unconditionally because it’s cheaper to shift-and-or every time than have a conditional operation.

Simple as that, now memory marked as RaW and non-readable will become readable on writes! Just like all other permission bits this is byte-level. malloc()ing 8 bytes, writing one byte to it, and then reading all 8 bytes will cause an uninitialized memory fault!

Gage fuzzing
Okay there’s probably a name for this already but I call it ‘gage’ fuzzing (from gage blocks, precisely ground measurement references). It’s a precise fuzzing technique I use where I start without a snapshot at all, but rather just the code. In this case I load up a PE/ELF, mark all writable regions as read-after-write, and point PC to a function I want to fuzz. Further I set up the parameters to the function, and if one of the parameters happens to be a pointer to memory I don’t understand yet, I can mark the contents of the pointer to read-after-write as well.

As globals and parameters are used I get faults telling me that uninitialized memory was used. This allows me to reverse out the specific fields that the function operates on as needed. Since the memory is read-after-write, if the function writes to the memory prior to reading it then I don’t have to worry what that memory is at all.

This process is extremely time consuming, but it is basically dynamic-driven reversing/source auditing. You lazily reverse the things you need to, which forces you to understand small pieces at a time. While you build understanding of the things the function uses you ultimately learn the code and learn potential places to audit more or add things like guard bytes.

This is my go-to methodology for fuzzing extremely hard targets where every advantage is required. Further this works for fuzzing codebases which are not runnable, or you only have partial snapshots of. Works great for kernel fuzzing or firmware fuzzing when you don’t have a great way of getting a snapshot!

I mention ‘function’ in this case but there’s nothing restricting you from fuzzing a whole application with this model. Things without global state can be trivially fuzzed in their entirety with a model like this. Further, I’ve done things like call the init routine for a class/program and then jump to the parser when init returns to skip some of the manual processing.

Theory into practice
So we know the features and what we want in theory, however in practice things get a lot harder. We have to abide by the design decisions while maintaining some semblance of performance and support for edge cases in the JIT.

We’ve got a few things that could make this hard to JIT. First of all performance is going to be an issue, we need to find a way to minimize the frequency of page table walks as well as decrease the cost of a walk itself. Further we have to be able to support edge cases where VMs are disabled, pages are not present, and VMs are accessing different memory at the same time.

64-bit saves the day
Since now the vectorized emulator is 64-bit rather than 32-bit, we can hold pointers in lanes of the vector. This allows us to use the scatter and gather instructions during page table walks. However, while magical and fast at what they do, these scatter/gather instructions are much slower than their standard load/store counterparts.

Thus in the edge case where VMs are accessing different memory we are able to vectorize the page table walks. This means we’re able to perform 8 completely different page table walks at the same time. However in most cases VMs are accessing the same memory and thus it’s cheaper for us to check if we’re accessing different memory first, and either perform the same walk for all VMs (same address), or perform a vectorized page table walk with scatter/gather instructions.

In the case of differing addresses this vectorized page table walk is much faster than 8 separate walks and provides a huge advantage over the previous 32-bit model.

Handling non-present pages
Typically in most architectures there is a present bit used in the page tables to indicate that an entry is present. This really just allows them to map in the physical address NULL in page tables. Since we’re running as a user application using virtual addresses we cheat and just use the pointers for page table entries.

If an entry is NULL (64-bit zero), then we stop the walk and immediately deliver a fault. This means to perform the page table walk until the final page we simply read a page table entry, check if it’s zero, and go to the next level. No need to mask off permission/present bits. For the final level we have a dirty bit, and a few more bits which we must mask off. We’ll discuss these other bits later.

What is a page fault?
In the case of a non-present page in the page table, or a permission bit not being present for the corresponding operation we need a way to deliver a page fault. Since the VM is just effectively one big function, we’re able to set a register with a VM exit code and return out. This is an implementation detail but it’s important that a ret allows us to exit from the emulator at any time.

Further since it’s possible VMs could have different permissions or page tables, we report a caused_vmexit mask, which indicates which lanes of the vector were responsible for causing the exception. This allows us to record the results, disable the faulting VMs, and re-enter the emulator to continue running the remaining VMs.

Memory costs
Since we’re running vectorized code we interleave 8 VMs at the same time. Further there is a permission byte for every byte. We also have a minimum page size of 8-bytes. Meaning the smallest possible actual commit size for a page on a single hardware thread is 128 bytes. PAGE_SIZE (8 bytes) * NUM_VMS (8) * 2 (permission byte and content byte). This is important as a single 4096-byte x86 page is actually 64 KiB. Which is… quite large. The larger the page size the better the performance, but the higher memory usage.

Saving time and memory
We’ve discussed that the previous MMU model used was quite slow for startup and shutdown times. This mean it could take 30+ seconds to start the emulator, and another 30 seconds to exit the process. Even with a hard ctrl+c.

To remedy this, everything we do is lazy. When I say lazy I mean that we try to only ever create mappings, copies, and perform updates when absolutely required.

VMs have no memory to start off
When a VM launches it has zero memory in it’s MMU. This means creating a VM costs almost nothing (a few milliseconds). It creates an empty page table and that’s it.

So where does memory come from?
Since a VM starts off with no memory at all, it can’t possibly have the contents of the snapshot we are executing from. This is because only the metadata of the snapshot was processed. When the VM attempts to touch the first memory it uses (likely the memory containing the first instruction), it will raise an exception.

We’ve designed the MMU such that there is an ability to install an exception handler. This means that on an exception we can check if the input snapshot contained the memory we faulted on. If it did then we can read the memory from the snapshot and map it in. Then the VM can be resumed.

This has the awesome effect of only memory that is ever touched is brought in from disk. If you have a 100 terabyte memory snapshot but the fuzz case only touches 1 MiB of memory, you only ever actually read 1 MiB from disk (plus the metadata of the snapshot, eg. PE/ELF headers). This memory is pulled in based on the page granularity in use. Since this is configurable you can tweak it to your hearts desire.

Sharing memory / forking
Memory which is only ever read has no reason to be copied for every VM. Thus we need a mechanism for sharing read-only memory between VMs. Further memory is shared between all cores running in the same “IL session”, or group of VMs fuzzing the same code and target.

We accomplish this by using a forking model. A ‘master’ MMU is created and an exception handler is installed to handle faults (to lazily pull in memory contents). The master MMU is the template for all future VMs and is the state of memory upon a reset.

When a core comes up, a fork from this ‘master’ MMU is created. Once again this is lazy. The child has no memory mapped in and will fault in pages from the master when needed.

When a page is accessed for reading only by a child VM the page in the child is directly mapped to the master’s copy. However since this memory could theoretically have write-permissions at the byte level, we protect this memory by setting an aliased bit on the last level page table, next to the dirty bit. This gives us a mechanism to prevent a master’s memory from ever getting updated even if it’s writable.

To allow for writes to the VM we add another bit to the last level page tables, a cow, or copy-on-write, bit. This is always accompanied with the aliased bit, and instead of delivering a fault on a write-to-aliased-memory access, we create a copy of the master’s page and allow writes to that.

An example in aliased/CoWed memory access
This leads us to a pretty sophisticated potential model of fault patterns. Let’s walk through a common case example.

An empty master MMU is created
An exception handler is added to the master MMU that faults in pages from the disk on-demand
A child is forked from the master
A read occurs to a page in the child
This causes an exception in the child as it has no memory
The exception handler recognizes there’s a master for this child and goes to access the master’s memory for this page
The master has no memory for this page and causes an exception
The master’s exception handler handles loading the page from disk, creating an entry
The master returns out with exception handled
The child directly links in the master’s page as aliased
Child returns with no exception
Child then dispatches a write to the same memory
The page is marked as aliased and thus cannot be written to
A copy of the master’s page is made
The permissions are checked in the page for write-access for all bytes being written to
The write occurs in the child-owned page
Success
While this is quite slow for the initial access, the child maintains it’s CoWed memory upon reset. This means that while the first few fuzz cases may be slow as memory is faulted in and copied, this cost eventually completely disappears as memory reaches a steady-state.

The overall result of this model is that memory only is ever read from disk if ever used, it then is only ever copied if it needs to be mutated. Memory which is only ever read is shared between all cores and greatly reduces cache pollution.

In theory a copy of all pages should be made for every NUMA node on the system to decrease latency in the case of a cache miss. This increases memory usage but increases performance.

All of this is done at page granularity which is configurable. Now you can see how big of an impact 8-byte pages can have as memory which may be writable (like a stack) but never is written to for a specific 8-byte region can be shared without extra memory cost.

This allows running 2048 4 GiB VMs with typically less than 200 MiB of memory usage as most fuzz cases touch a tiny amount of memory. Of course this will vary by target.

Deduplicated memory
Ha! You thought we were all done and ready to talk about performance? Not quite yet, we’ve got another trick up our sleeves!

Since we’re already sharing memory and have support for aliased memory, we can take it one step further. When we add memory to the VM we can deduplicate it.

This might sound really complex, but the implementation is so simple that there’s almost no reason to not do it. Rather than directly creating memory in the the master, we can instead maintain a HashSet of pages and create aliased mappings to the entries in this set. When memory is added to a VM it is added to the deduplicated HashSet, which will create a new entry if it does not exist, or do nothing if it already exists. The page tables then directly reference the memory in this HashSet with the aliased bit set. Since pages contain the permissions this automatically handles creating different copies of the same memory with different permissions

Ta-da! We now will only create one read-only copy of each unique page. Say you have 1 MiB of read-writable zeros (would be 16 MiB when interleaved and with permissions), and are using 8-byte pages, you end up only ever creating one 8-byte page (128-byte actual backing) for all of this memory! As with other aliased memory, it can be cow memory and cloned if modified.

The gain from this is minimal in practice, but the code complexity increase given we already handle cow and aliased memory is so little that there’s really no reason to not do it. Since the Xeon Phi has no L3 cache, anything I can do to reduce cache pollution helps.

For example with a child with memory contents “AAAA00:D!!” where the “:D” was written in at offset 6.

cow_and_dedup

Performance
Alright so we’ve talked about everything we implement in the MMU, but we haven’t talked at all about the JIT or performance.

There are two important aspects to performance:

The JIT
Injecting fuzz cases / allocating memory
The JIT performance being important should be obvious. Memory accesses are the most expensive things we can do in our emulator and are responsible from bringing our best case 2 trillion instructions/second benchmark to about 40-120 billion instructions/second in an actual codebase (old numbers, old MMU, 32-bit model). The faster we can make memory accesses, the closer we can get to this best-case performance number. This means we have a potential 50x speedup if we were to make memory accesses cost nothing.

Next we have the maybe-not-so-obvious performance-critical aspect. Getting fuzz cases into the VMs and handling dynamic allocations in the VMs. While this is pretty much never a problem in traditional fuzzers, on a small target I may be running between 2-5 million fuzz cases per second. Meaning I need to somehow perform 2-5 million changes to the MMU per second (typically 1024-or-so byte inputs).

Further the VM may dynamically allocate memory via malloc() which we hook to get byte-level allocation support and to track uninitialized memory. A VM might do this a few times a fuzz case, so this could result in potentially tens of millions of MMU modifications per second.

The JIT / IL
We’re not going to go into insane details as I’ve open sourced the actual JIT used in the MMU described by this blog. However we can hit on some of the high-level JIT and IL concepts.

When we’re running under the JIT there may be arbitrary VMs running (the VM-0-must-always-be-running restriction described in the intro has been lifted), as well as potential differing addresses that they are accessing.

Differing addresses
Since a vectorized page table walk is more expensive than a single page walk, we first always check whether or not the VMs that are active are accessing the same memory. If they’re accessing the same memory then we can extract the address from one of the VMs and perform a single scalar page walk. If they differ then we perform the more expensive vectorized walk (which is still a huge improvement from the 32-bit model of a different scalar walk for every differing address).

Since the only metadata we store in the page tables are the aliased, CoW, and dirty bits, the scalar page walk is safe to do for all VMs. If permissions differ between the VMs that’s fine as those bytes are stored in the page itself.

The part of the page walk that gets complex during a vectorized walk is updating the dirty bits. In a scalar walk it’s simple. If the dirty bit is not set and we’re performing a write, then we add to the dirty list and set the dirty bit. Otherwise we skip updating the dirty bit and dirty list. This prevents duplicate entries in the dirty list. Further we store the guest address and the translated address in the dirty list so we do not have to re-translate during a reset. If an exception occurs at any point during the walk, all VMs that are enabled are reported to have caused the exception.

We also perform the aliased memory check if and only if the dirty bit was not set. This aliased memory check is how we prevent writing to an aliased page. Since this check has a non-zero cost, and since dirty memory can never be aliased, we simply skip the check if the memory is already dirty. As it’s guaranteed to not be aliased if it’s dirty.

Vectorized translation
However in a vectorized walk it gets really tricky. First it’s possible that the different addresses fail translation at differing levels (during page table walks and during permission checks). Further they can have differing dirtiness which might require multiple entries to be added to the dirty list.

To handle translations failing at different points, we mask off VMs as they fail at various points. At the end of the translation we determine if any VM failed, and if it did we can report the failure correctly for all VM’s that failed at any point during the translation. This allows us to get a correct caused_vmexit mask from a single translation, rather than getting a partial report and getting more exceptions at a different translation stage on the next re-entry.

Vectorized dirty list updating
Further we have to handle dirty bits. I do this in a weird way right now and it might change over time. I’m trying to keep all possible JIT at parity with the interpreted implementation. The interpreted version is naive and simply performs the translations on all VMs in left-to-right order (see the JIT tests for this operation). This also maintains that no duplicates ever exist in the dirty lists.

To prevent duplicates in the dirty list we rely on the dirty bit in the page table, however when handling differing addresses we could potentially update the same address twice and create two dirty entries. The solution I made for this is to perform vectorized checks for the dirty bits, and if they’re already set we skip the expensive setting of the dirty bits. This is the fast path.

However in the slow path we store the addresses to the stack and individually update dirty bits and dirty entries for each lane. This prevents us from adding duplicates to the dirty list and keeps the JIT implementation at parity with the interpreter (thus allowing 1-to-1 checks for JIT correctness against the interpreter). Since we skip this slow path if the memory is already dirty, this probably won’t matter for performance. If it turns out to matter later on I might drop the no-duplicates-in-the-dirty-list restriction and vectorize updates to this list.

IL MMU routines
I’m going to have a whole blog on my IL, but it’s a simple SSA IL.

Memory accesses themselves are pretty fast in my vectorized model, however the translations are slow. To mitigate this I split up translations and read/write operations in my IL. Since page walks, dirty updates, and permission checks are done in my translate IL instruction, I’m able to reuse translations from previous locations in the IL graph which use the same IL expression as the address.

For example, a 4-byte translate for writing of rsp+0x50 occurs at the root block of a function. Now at future locations in the graph which read or write at the same location for 4-or-fewer bytes can reuse the translation. Since it’s an SSA the rsp+0x50 value is tied to a certain version of rsp, thus changes to rsp do not cause the wrong translation to be used. This effectively deletes the page walks for stack locals and other memory which is not dynamically indexed in the function. It’s kind of like having a TLB in the IL itself.

Since the initial translate was responsible for the permission checks and updates of things like the RaW bits and dirty bits, we never have to run these checks again in this case. This turns memory operations into simple loads and stores.

Since stores are supersets of loads and larger sizes are supersets of smaller sizes, I can use translations from slightly different sizes and accesses.

Since it’s possible a VM exit occurs and memory/permissions are changed, I must have invalidate these translations on VM exits. More specifically I can invalidate them only on VM entries where a page table modification was made since the last VM exit. This makes the invalidate case rare enough to not matter.

The performance numbers
These are the performance numbers (in cycles) for each type and size of operation. The translate times are the cost of walking the page tables and validating permissions, the access times are the cost of reading/writing to already translated memory. The benchmarks were done on a Xeon Phi 7210 on a single hardware thread. All times are in cycles for a translation and access times for all 8 lanes.

These are best-case translate/access times as it’s the same memory translated in a loop over and over causing the tables and memory in question to be present in L1 cache.

The divergent cases are ones where different addresses were supplied to each lane and force vectorized page walks.

Write: false | opsize: 1 | Diverge: false | Translate    37.8132 cycles | Access    10.5450 cycles
Write: false | opsize: 2 | Diverge: false | Translate    39.0831 cycles | Access    11.3500 cycles
Write: false | opsize: 4 | Diverge: false | Translate    39.7298 cycles | Access    10.6403 cycles
Write: false | opsize: 8 | Diverge: false | Translate    35.2704 cycles | Access     9.6881 cycles
Write: true  | opsize: 1 | Diverge: false | Translate    44.9504 cycles | Access    16.6908 cycles
Write: true  | opsize: 2 | Diverge: false | Translate    45.9377 cycles | Access    15.0110 cycles
Write: true  | opsize: 4 | Diverge: false | Translate    44.8083 cycles | Access    16.0191 cycles
Write: true  | opsize: 8 | Diverge: false | Translate    39.7565 cycles | Access     8.6500 cycles
Write: false | opsize: 1 | Diverge: true  | Translate   140.2084 cycles | Access    16.6964 cycles
Write: false | opsize: 2 | Diverge: true  | Translate   141.0708 cycles | Access    16.7114 cycles
Write: false | opsize: 4 | Diverge: true  | Translate   140.0859 cycles | Access    16.6728 cycles
Write: false | opsize: 8 | Diverge: true  | Translate   137.5321 cycles | Access    14.1959 cycles
Write: true  | opsize: 1 | Diverge: true  | Translate   158.5673 cycles | Access    22.9880 cycles
Write: true  | opsize: 2 | Diverge: true  | Translate   159.3837 cycles | Access    21.2704 cycles
Write: true  | opsize: 4 | Diverge: true  | Translate   156.8409 cycles | Access    22.9207 cycles
Write: true  | opsize: 8 | Diverge: true  | Translate   156.7783 cycles | Access    16.6400 cycles
Performance analysis
These numbers actually look really good. Just about 10 or so cycles for most accesses. The translations are much more expensive but with TLBs and caching the translations in the IL tree we should hopefully do these things rarely. The divergent translation times are about 3.5x more expensive than the scalar counterparts which is pretty impressive. 8 separate page walks at only 3.5x more cost than a single walk! That’s a big win for this new MMU!

TLBs (not implemented as of this writing)
Similar to the cached translations in the IL tree, I can have a TLB which caches a limited amount of arbitrary translations, just like an actual CPU or many other JITs. I currently plan on having TLB entries for each type of operation such that no permission checks are needed on read/write routines. However I could use a more typical TLB model where translations are cached (rather than permission checks and RaW updates), and then I would have to perform permission checks and RaW updates on all memory accesses (but not the translation phase).

I plan to just implement both models and benchmark them. The complexity of theorizing this performance difference is higher than just doing it and getting real measurements…

Fast injection/permission modifications
To support fast fuzz case injection and permission changes I have a few handwritten AVX-512 routines which are optimized for speed. These can then be tested against the naive reference implementation for correctness as there’s a much higher chance for mistakes.

I expose 3 different routines for this. A vectorized broadcast (writing the same memory to multiple VMs), a vectorized memset (applying the same byte to either memory contents or permissions), and a vectorized write-multiple.

Vectorized broadcast
This one is pretty simple. You supply an address in the VM, a payload, and a mask (deciding which VMs to actually write to). This will then write the same payload to all VMs which are enabled by the mask. This surprisingly doesn’t have a huge use case that I can think of yet.

Vectorized memset
Since permissions and memory are stored right next to each other this memset is written in a way that it can be used to update either permissions or contents. This takes in an address, a byte to broadcast, a bool indicating if it should write to permissions or contents, and a mask of VMs to broadcast to.

This routine is how permissions are updated in bulk. I can quickly update permissions on arbitrary sets of VMs in a vectorized manner. Further it can be used on contents to do things like handle zeroing of memory on a hooked call like malloc().

Vectorized write-multiple
This is how we get fuzz cases in. I take one address, a VM mask, and multiple inputs. I then inject those inputs to their corresponding VMs all at the same address. This allows me to write all my differing fuzz cases to the same location in memory very quickly. Since most fuzzing is writing an input to all VMs at the same location this should suffice for most cases. If I find I’m frequently writing multiple inputs to multiple different locations I’ll probably make another specialized routine.

Due to the complexities of handling partial reads from the input buffers in a vectorized way, this routine is restricted to writing 8-byte size aligned payloads to 8-byte aligned addresses. To get around this I just pad out my fuzz inputs to 8-byte boundaries.

Are these fast routines really needed?
For example the benchmarks for the Rust implementation for a page table of shape: [16, 16, 16, 13, 3]

Note that the benchmarks are a single hardware thread running on a Xeon Phi 7210

Empty SoftMMU created in                            0.0000 seconds
1 MiB of deduped memory added in                    0.1873 seconds
1024 byte chunks read per second                30115.5741
1024 byte chunks written per second             29243.0394
1024 byte chunks memset per second              29340.3969
1024 byte chunks permed per second              34971.7952
1024 byte chunks write multiple per second       6864.1243
And the AVX-512 handwritten implementation on the same machine and same shape:

Empty SoftMMU created in                            0.0000 seconds
1 MiB of deduped memory added in                    0.1878 seconds
1024 byte chunks read per second                30073.5090
1024 byte chunks written per second            770678.8377
1024 byte chunks memset per second             777488.8143
1024 byte chunks permed per second             780162.1310
1024 byte chunks write multiple per second     751352.4038
Effectively a 25x speedup for the same result!

With a larger page size ([16, 16, 16, 6, 10]) this number goes down as I can use the old translation longer and I spend less time translating pages:

Rust implementations:

Empty SoftMMU created in                            0.0001 seconds
1 MiB of deduped memory added in                    0.0829 seconds
1024 byte chunks read per second                30201.6634
1024 byte chunks written per second             31850.8188
1024 byte chunks memset per second              31818.1619
1024 byte chunks permed per second              34690.8332
1024 byte chunks write multiple per second       7345.5057
Hand-optimized implementations:

Empty SoftMMU created in                            0.0001 seconds
1 MiB of deduped memory added in                    0.0826 seconds
1024 byte chunks read per second                30168.3047
1024 byte chunks written per second          32993840.4624
1024 byte chunks memset per second           33131493.5139
1024 byte chunks permed per second           36606185.6217
1024 byte chunks write multiple per second   10775641.4470
In this case it’s over 1000x faster for some of the implementations! At this rate we can trivially get inputs in much faster than the underlying code possibly could run!

Future improvements/ideas
Currently a full 64-bit address space is emulated. Since nothing we emulate uses a full 64-bit address space this is overkill and increases the page table memory size and page table walk costs. In the future I plan to add support for partial address space support. For example if you only define the page table to handle 16-bit addresses, it will, optionally based on another constant, make sure addresses are sign-extended or zero-extended from these 16-bit addresses. By supporting both sign-extended and zero-extended addresses we should be able to model all architecture’s specific behaviors. This means if running a 32-bit application in our 64-bit JIT we could use a 32-bit address space and decrease the cost of the MMU.

I could add more fast-injection routines as needed.

I may move permission checks to loads/stores rather than translation IL operations, to allow reuse of TLB entries for the same page but differing offsets/operations.

Gamozo Labs Blog
Gamozo Labs Blog
b@gamozolabs.com
 gamozolabs
 gamozolabs
I blog about random things security, everything is broken, nothing scales, shared memory models are flawed.

Gamozo Labs Blog
About
Miniblog: How conditional branches work in Vectorized Emulation
Oct 7, 2019

Twitter
Follow me at @gamozolabs on Twitter if you want notifications when new blogs come up. I also do random one-off posts for cool data that doesn’t warrant an entire blog!

Let me know if you like this mini-blog format! It takes a lot less time than a whole blog, but I think will still have interesting content!

Prereqs
You should probably read the Introduction to Vectorized Emulation blog!

Or perhaps watch the talk I gave at RECON 2019

Summary
I spent this weekend working on a JIT for my IL (FalkIL, or fail). I thought this would be a cool opportunity to make a mini-blog describing how I currently handle conditional branches in vectorized emulation.

This is one of the most complex parts of vectorized emulation, and has a lot of depth. But today we’re going to go into a simple merging example. What I call the “auto-merge”. I call it an auto-merge because it doesn’t require any static analysis of potential merging points. The instructions that get emit simply allow for auto re-merging of divergent VMs. It’s really simple, but pretty nifty. We have to perform this logic on every branch.

FalkIL Example
Here’s an example of what FalkIL looks like:

FalkIL example

JIT
Here’s what the JIT for the above IL example looks like:

FalkIL JIT example

Ooof… that exploded a bit. Let’s dive in!

JIT Calling Convention
Before we can go into what the JIT is doing, we have to understand the calling convention we use. It’s important to note that this calling convention is custom, and follows no standard convention.

Kmask Registers
Kmask registers are the bitmasks provided to us in hardware to mask off certain operations. Since we’re always executing instructions even if some VMs have been disabled, we must always honor using kmasks.

Intel provides us with 8 kmask registers. k0 through k7. k0 is hardcoded in hardware to all ones (no masking). Thus, we’re not able to use this for general purpose masking.

Online VM mask
Since at any given time we might be performing operations with VMs disabled, we need to have one kmask register always dedicated to holding the mask of VMs that are actively running. Since k1 is the first general purpose kmask we can use, that’s exactly what we pick. Any bit which is clear in k1 (VM is disabled), must not have its state modified. Thus you’ll see k1 is used as a merging mask for almost every single vectorized operation we do.

By using the k1 mask in every instruction, we preseve the lanes of vector registers which are disabled. This provides near-zero-cost preservation of disabled VM states, such that we don’t have to save/restore massive ZMM registers during divergence.

This mask must also be honored during scalar code that emulates complex vectorized operations (for example divs, which have no vectorized instruction).

“Following” VM mask
At some points during emulation we run into situations where VMs have to get disabled. For example, some VMs might take a true branch, and others might take the false (or “else”) side of a branch. In this case we need to make a decision (very quickly) about which VM to follow. To do this, we have a VM which we mark as the “following” VM. We store this “following” VM mask in k7. This always contains a single bit, and it’s the bit of the VM which we will always follow when we have to make divergence decisions.

The VM we are “following” must always be active, and thus (k7 & k1) != 0 must always be true! This k7 mask only has to be updated when we enter the JIT, thus the computation of which VM to “follow” may be complex as it will not be a common expense. While the JIT is executing, this k7 mask will never have to be updated unless the VM we are following causes a fault (at which point a new VM to follow will be computed).

Kmask Register Summary
Here’s the complete state of kmask register allocation during JIT

K0    - Hardcoded in hardware to all ones
K1    - Bitmask indicating "active/online" VMs
K2-K6 - Scratch kmask registers
K7    - "Following" VM mask
ZMM registers
The 512-bit ZMM registers are where we store most of our active contextual data. There are only 2 special case ZMM registers which we reserve.

“Following” VM index vector
Following in the same suit of the “Following VM mask”, mentioned above, we also store the index for the “following” VM in all 8 64-bit lanes of zmm30. This is needed to make decisions about which VM to follow. At certain points we will need to see which VM’s “agree with” the VM we are following, and thus we need a way to quickly broadcast out the following VMs values to all components of a vector.

By holding the index (effectively the bit index of the following VM mask) in all lanes of the zmm30 vector, we can perform a single vpermq instruction to broadcast the following VM’s value to all lanes in a vector.

Similar to the VM mask, this only needs to be computed when the JIT is entered and when faults occur. This means this can be a more expensive operation to fill this register up, as it stays the same for the entirity of a JIT execution (until a JIT/VM exit).

Why this is important
Lets say:

zmm31 contains [10, 11, 12, 13, 14, 15, 16, 17]

zmm30 contains [3, 3, 3, 3, 3, 3, 3, 3]

The CPU then executes vpermq zmm0, zmm30, zmm31

zmm0 now contains [13, 13, 13, 13, 13, 13, 13, 13]… the 3rd VM’s value in zmm31 broadcast to all lanes of zmm0

Effectively vpermq uses the indicies in its second operand to select values from the third operand.

“Desired target” vector
We allocate one other ZMM register (zmm31) to hold the block identifiers for where each lane “wants” to execute. What this means is that when divergence occurs, zmm31 will have the corresponding lane updated to where the VM that diverged “wanted” to go. VMs which were disabled thus can be analyzed to see where they “wanted” to go, but instead they got disabled :(

ZMM Register Summary
Here’s the complete state of ZMM register allocation during JIT

Zmm0-Zmm3  - Scratch registers for internal JIT use
Zmm4-Zmm29 - Used for IL register allocation
Zmm30      - Index of the VM we are following broadcast to all 8 quadwords
Zmm31      - Branch targets for each VM, indicates where all VMs want to execute
General purpose registers
These are fairly simple. It’s a lot more complex when we talk about memory accesses and such, but we already talked about that in the MMU blog!

When ignoring the MMU, there are only 2 GPRs that we have a special use for…

Constant storage database
On the Knights Landing Xeon Phi (the CPU I develop vectorized emulation for), there is a huge bottleneck on the front-end and instruction decode. This means that loading a constant into a vector register by loading it into a GPR mov, then moving it into the lowest-order lane of a vector vmovq, and then broadcasting it vpbroadcastq, is actually a lot more expensive than just loading that value from memory.

To enable this, we need a database which just holds constants. During the JIT, constants are allocated from this table (just appending to a list, while deduping shared constants). This table is then pointed to by r11 during JIT. During the JIT we can load a constant into all active lanes of a VM by doing a single vpbroadcastq zmm, kmask, qword [r11+OFFSET] instruction.

While this might not be ideal for normal Xeon processors, this is actually something that I have benchmarked, and on the Xeon Phi, it’s much faster to use the constant storage database.

Target registers
At the end of the day we’re emulating some other architecture. We hold all target architecture registers in memory pointed to by r12. It’s that simple. Most of the time we hold these in IL registers and thus aren’t incurring the cost of accessing memory.

GPR summary
r11 - Points to constant storage database (big vector of quadword constants)
r12 - Points to target architecture registers
Phew
Okay, now we know what register states look like when executing in JIT!

Conditional branches
Now we can get to the meat of this mini-blog! How conditional branches work using auto-merging! We’re going to go through instruction-by-instruction from the JIT graph we showed above.

Here’s the specific code in question for a conditional branch:

Conditional Branch

Well that looks awfully complex… but it’s really not. It’s quite magical!

The comparison
comparison

First, the comparision is performed on all lanes. Remember, ZMM registers hold 8 separate 64-bit values. We perform a 64-bit unsigned comparison on all 8 components, and store the results into k2. This means that k2 will hold a bitmask with the “true” results set to 1, and the “false” results set to 0. We also use a kmask k1 here, which means we only perform the comparison on VMs which are currently active. As a result of this instruction, k2 has the corresponding bits set to 1 for VMs which were actively executing at the time, and also resulted in a “true” value from their comparisons.

In this case the 0x1 immediate to the vpcmpuq instruction indicates that this is a “less than” comparison.

vpcmpq/vpcmpuq immediate
Note that the immediate value provided to vpcmpq and the unsigned variant vpcmpuq determines the type of the comparison:

cmpimm

The comparison inversion
comparison inversion

Next, we invert the comparison operation to get the bits set for active VMs which want to go to the “false” path. This instruction is pretty neat.

kandnw performs a bitwise negation of the second operand, and then ands with the third operand. This then is stored into the first operand. Since we have k2 as the second operand (the result of the comparison) this gets negated. This then gets anded with k1 (the third operand) to mask off VMs which are not actively executing. The result is that k3 now contains the inverted result from the comparison, but we keep “offline” VMs still masked off.

In C/C++ this is simply: k3 = (~k2) & k1

The branch target vector
branch targets

Now we start constructing zmm0… this is going to hold the “labels” for the targets each active lane wants to go to. Think of these “labels” as just a unique identifier for the target block they are branching to. In this case we use the constant storage database (pointed to by r11) to load up the target labels. We first load the “true target” labels into zmm0 by using the k2 kmask, the “true” kmask. After this, we merge the “false target” labels into zmm0 using k3, the “false/inverted” kmask.

After these 2 instructions execute, zmm0 now holds the target “labels” based on their corresponding comparison results. zmm0 now tells us where the currently executing VMs “want to” branch to.

The merge into master
merge into master

Now we merge the target branches for the active VMs which were just computed (zmm0), into the master target register (zmm31). Since VMs can be disabled via divergence, zmm31 holds the “master” copy of where all VMs want to go (including ones which have been masked off and are “waiting” to execute a certain target).

zmm31 now holds the target labels for every single lane with the updated results of this comparison!

Broadcasting the target
broadcasting the target

Now that we have zmm31 containing all of the branch targets, we now have to pick the one we are going to follow. To do this, we want a vector which contains the broadcasted target label of the VM we are following. As mentioned in the JIT calling convention section, zmm30 contains the index of the VM we are following in all 8 lanes.

Example
Lets say for example we are following VM #4 (zero-indexed).

zmm30 contains [4, 4, 4, 4, 4, 4, 4, 4]

zmm31 contains [block_0, block_0, block_1, block_1, block_2, block_2, block_2, block_2]

After the vpermq instruction we now have zmm1 containing [block_2, block_2, block_2, block_2, block_2, block_2, block_2, block_2].

Effectively, zmm1 will contain the block label for the target that the VM we are following is going to go to. This is ultimately the block we will be jumping to!

Auto-merging
auto-merging

This is where the magic happens. zmm31 contains where all the VMs “want to execute”, and zmm1 from the above instruction contains where we are actually going to execute. Thus, we compute a new k1 (active VM kmask) based on equality between zmm31 and zmm1.

Or in more simple terms… if a VM that was previously disabled was waiting to execute the block we’re about to go execute… bring it back online!

Doin’ the branch
branching

Now we’re at the end. k2 still holds the true targets. We and this with k7 (the “following” VM mask) to figure out if the VM we are following is going to take the branch or not.

We then need to make this result “actionable” by getting it into the eflags x86 register such that we can conditionally branch. This is done with a simple kortestw instruction of k2 with itself. This will cause the zero flag to get set in eflags if k2 is equal to zero.

Once this is done, we can do a jnz instruction (same as jne), causing us to jump to the true target path if the k2 value is non-zero (if the VM we’re following is taking the true path). Otherwise we fall through to the “false” path (or potentially branch to it if it’s not directly following this block).

Update
After a little nap, I realized that I could save 2 instructions during the conditional branch. I knew something was a little off as I’ve written similar code before and I never needed an inverse mask.

updated JIT

Here we’ll note that we removed 2 instructions. We no longer compute the inverse mask. Instead, we initially store the false target block labels into zmm31 using the online mask (k1). This temporarly marks that “all online VMs want to take the false target”. Then, using the k2 mask (true targets), merge over zmm31 with the true target block labels.

Simple! We remove the inverse mask computation kandnw, and the use of the zmm0 temporary and merge directly into zmm31. But the effect is exactly the same as the previous version.

Not quite sure why I thought the inverse mask was needed, but it goes to show that a little bit of rest goes a long way!

Due to instruction decode pressure on the Xeon Phi (2 instructions decoded per cycle), this change is a minimum 1 cycle improvement. Further, it’s a reduction of 8 bytes of code per conditional branch, which reduces L1i pressure. This is likely in the single digit percentages for overall JIT speedup, as conditional branches are everywhere!

Fin
And that’s it! That’s currently how I handle auto-merging during conditional branches in vectorized emulation as of today! This code is often changed and this is probably not its final form. There might be a simpler way to achieve this (fewer instructions, or lower latency instructions)… but progress always happens over time :)

It’s important to note that this auto-merging isn’t perfect, and most cases will result in VMs hanging, but this is an extremely low cost way to bring VMs online dynamically in even the tightest loops. More macro-scale merging can be done with smarter static-analysis and control flow decisions.

I hope this was a fun read! Let me know if you want more of these mini-blogs.

Gamozo Labs Blog
Gamozo Labs Blog
b@gamozolabs.com
 gamozolabs
 gamozolabs
I blog about random things security, everything is broken, nothing scales, shared memory models are flawed.

Gamozo Labs Blog
About
Some thoughts on ToB's GPU-based fuzzing
Oct 23, 2020

The blog
The blog we’re looking at today is an incredible blog by Ryan Eberhardt on the Trail of Bits blog! You should read it first, it’s really neat, there’s also some awesome graphics in it which makes it a super fun read!

Let’s build a high-performance fuzzer with GPUs!

Summary
In the ToB blog, they talk about using GPUs to fuzz. More specifically, they talk about lifting a target architecture into LLVM IR, and then emitting the LLVM IR to a binary which can run on a GPU. In this case, they’re targeting PTX assembly to run on the NVIDIA Tesla T4 GPU. This is done using a tool ToB has been working on for quite a while, called remill, which is designed for binary translation. Remill alone is incredibly impressive.

The target they picked as a benchmark is the BFP packet filtering code in libpcap, pcap_filter_with_aux_data. This function is pretty simple, and it executes a compiled BPF filter and uses it to extract information and filter a packet.

The blog talks about some of the hurdles in getting performant execution on GPUs, organization of data, handing virtual memory, etc. Once again, go read it. It’s really neat, the graphics alone make it a worthwhile read!

I’m super excited about this blog, mainly because it’s very similar to vectorized emulation that I’ve worked on in the past, and it starts answering questions about GPU-based fuzzing that I have been too lazy to look into. While this blog goes into some criticisms, it’s important to note that the research is only just starting, there is much progress to be had! It’s also important to note that this research has been being done by Ryan for only 2 months. That is incredible progress.

The Problems
Nevertheless, I have a few problems with the blog that stood out to me. I’m kind of always the asshole pointing these things out, but I think there are some important things to discuss.

The comparison
In the blog, the comparison being done and being presented is largely about comparing the performance of libfuzzer, against their GPU based fuzzer. Further, the comparisons are largely about the number of executions per second (or as I call them, fuzz cases per second), per unit US dollar. This comparison is largely to emphasize the cost efficiencies of fuzzing on the GPU, so we’ll keep that in mind. We don’t want to stray too far from their actual point.

Their hardware they’re testing on are 2 different Google Cloud Compute nodes which have various specs. The one used to benchmark libfuzzer is an n1-standard-8, this is a 4 core, 8 hyperthread, Intel Skylake machine. This costs $0.38/hour according to their blog, and of course, this checks out.

The other machine they’re testing on, for their GPU metrics, is a NVIDIA Tesla T4 single GPU compute node from Google Cloud Project. They claim this costs $0.35/hour, and once again, that’s accurate. This means the two machines are effectively the same price, and we’ll be able to compare them at a rough level without really taking into consideration their costs.

In their blog, they mention that “This isn’t an entirely fair comparison.”, and this is largely referring to that their fuzzer is not providing mutated inputs to the function, whereas libfuzzer is. This is a major issue. However, their fuzzer is resetting the state of the target every fuzz case, and libfuzzer is relying on the function not having any peristant state that needs to be reset. This gives libfuzzer a large advantage. Finally, the GPU based fuzzer also works on binaries, where libfuzzer requires source, so once again, there’s a lot of variables at play here. It is important to note, they’re mainly looking for order-of-magnitude estimates. But… this is a lot more than should be controlled for in my opinion. Important to also note that the blog concludes with a ~4x improvement from libfuzzer, thus, it’s well below the order-of-magnitude concerns of unfairness.

Of course, if you’ve read my blogs before. You’ll know I absolutely hate comparisons between problems with multiple variables. First of all, the cost of mutating an input is incredibly expensive, especially for a potentially large packet, say 1500 bytes. Further, the target which is being picked is a single function which does very little processing from first glance, but we’ll look into this more later.

So, let’s start off by eliminating one variable right away. What is the cost of generating an input from libfuzzer, and what is the cost of the actual function under test. This will effectively tell us how “fair” the execution comparison is, the binary vs source is subjective and clearly the binary-based engine is more impressive.

How do we do this? Well, let’s first figure out how fast libfuzzer can execute something that does literally nothing. This will give us a baseline of libfuzzer performance given it’s targeting something that does literally nothing.

#include <stdlib.h>
#include <stdint.h>

extern int LLVMFuzzerTestOneInput(const uint8_t *Data, size_t Size) {
  return 0;  // Non-zero return values are reserved for future use.
}
clang-12 -fsanitize=fuzzer -O2 test.c
We’ll run this test on a Intel(R) Xeon(R) Gold 6252N CPU @ 2.30GHz turboing to 3.6 GHz. This isn’t the same as their GCP setup, but we’ll do some of our own comparisons locally, thus we’re talking about relatives and not absolutes.

They don’t talk much in their blog about what they used to seed libfuzzer, so we’ll just give it no seeds and cap the input size to 1500 bytes, or about a single MTU for a network packet.

pleb@grizzly:~/libpcap/harness$ ./a.out -max_len=1500
INFO: Running with entropic power schedule (0xFF, 100).
INFO: Seed: 2252408900
INFO: Loaded 1 modules   (1 inline 8-bit counters): 1 [0x4ea0b0, 0x4ea0b1), 
INFO: Loaded 1 PC tables (1 PCs): 1 [0x4c0840,0x4c0850), 
INFO: A corpus is not provided, starting from an empty corpus
#2      INITED cov: 1 ft: 1 corp: 1/1b exec/s: 0 rss: 27Mb
#8388608        pulse  cov: 1 ft: 1 corp: 1/1b lim: 1500 exec/s: 4194304 rss: 28Mb
#16777216       pulse  cov: 1 ft: 1 corp: 1/1b lim: 1500 exec/s: 3355443 rss: 28Mb
#33554432       pulse  cov: 1 ft: 1 corp: 1/1b lim: 1500 exec/s: 3050402 rss: 28Mb
#67108864       pulse  cov: 1 ft: 1 corp: 1/1b lim: 1500 exec/s: 3195660 rss: 28Mb
#134217728      pulse  cov: 1 ft: 1 corp: 1/1b lim: 1500 exec/s: 3121342 rss: 28Mb
Hmm, it seems it has settled in at about 3.12 million executions per second on a single core. Hmm, that seems a bit fast compared to the 1.9 million executions per second they see on their 8 thread machine in GCP, but maybe the target is really that complex and slows down performance.

Next, lets see how expensive the target code is outside of libfuzzer.

use std::time::Instant;
use pcap_sys::*;

#[link(name = "pcap")]
extern {
    fn bpf_filter_with_aux_data(
        pc: *const bpf_insn,
        p:  *const u8,
        wirelen: u32,
        buflen:  u32,
        aux_data: *const u8,
    );
}

fn main() {
    const ITERS: u64 = 100_000_000;

    unsafe {
        let mut program: bpf_program = std::mem::zeroed();

        // Ethernet linktype + 1500 snapshot length
        let pcap = pcap_open_dead(1, 1500);
        assert!(!pcap.is_null());

        // Compile the program
        let status = pcap_compile(pcap, &mut program,
            "dst host 1.2.3.4 or tcp or udp or ip or ip6 or arp or rarp or \
            atalk or aarp or decnet or iso or stp or ipx\0"
            .as_ptr() as *const _,
            1, PCAP_NETMASK_UNKNOWN);
        assert!(status == 0, "Failed to compile pcap thingy");

        let buf = vec![0u8; 1500];

        let time = Instant::now();
        for _ in 0..ITERS {
            // Filter a packet
            bpf_filter_with_aux_data(
                program.bf_insns,
                buf.as_ptr(),
                buf.len() as u32,
                buf.len() as u32,
                std::ptr::null()
            );
        }
        let elapsed = time.elapsed().as_secs_f64();

        print!("{:14.2} packets/second\n", ITERS as f64 / elapsed);
    }
}
We’re just going to compile the filter they mention in their blog, and then call bpf_filter_with_aux_data in a loop, applying the filter, and then we’ll print the number of iterations per second that we can do. In my specific case, I’m using libpcap-1.9.1 as distributed as a source code zip, this may differ slightly from their version.

pleb@grizzly:~/libpcap/harness$ RUSTFLAGS="-L../libpcap-1.9.1" cargo run --release
    Finished release [optimized] target(s) in 0.01s
     Running `target/release/harness`
   18703628.46 packets/second
Uh oh, that’s a bit concerning. The target can be executed about 18.7 million times per second, however libfuzzer is capped at pretty much a maximum of 3.1 million executions a second. This means the overhead of libfuzzer, which is not part of this comparison, is a factor of about 6. This means that libfuzzer is given about a 6x penalty, compared to the GPU fuzzer, which immediately gets rid of the ~4.4x advantage that the GPU fuzzer had over libfuzzer in their blog.

This unfortunately, was exactly as I expected. For a target this small, the overhead of creating an input greatly exceeds the cost of the target execution itself. This, unfortunately, makes the comparison against libfuzzer pretty much invalid in my eyes.

Trying to make the comparison closer
I’m lucky in that I have many binary-based snapshot fuzzers sitting around. It’s kind of my specialty. It’s important to note, from this point on, this comparison is for myself. It’s not to critique the blog, it’s simply for me to explore my performance against ToB’s GPU performance. I don’t care which one is better, this is largely for me to figure out if I personally want to start investing some time and money into GPU based fuzzing.

So, to start off, I’m going to compare the GPU fuzzer against my vectorized emulation. Vectorized emulation is a technique that I use to execute multiple VMs in parallel using AVX-512. In this specific case, I’m targeting a RISC-V processor (rv64ima) which will be emulated on my Intel machines by using AVX-512. Since 512 bits / 64 bits is 8, that means I’m running 8 VMs per hardware thread.

Vectorized emulation entirely contains only my own code. I wrote the lifters, the IL, the optimization passes, the JITs, the assemblers, the APIs, everything. This gives me a massive amount of control over adapting it to various targets, and make rapid changes to internals when needed. But, it also means, my code generation should be significantly worse than something like LLVM, as I do only the most basic optimizations (DCE, deduplication, etc). I don’t do any reordering, loop unrolling, memory access elision, etc.

Let’s try it!

The environment
To try to get as close to comparing against ToB’s GPU fuzzer, I’m going to fuzz a binary target and provide no mutation of the inputs. I’m simply going to use a 1500-byte buffer containing zeros. Unfortunately, there’s no specifics about what they used as an input, so we’re making the assumption that a 1500-byte zeroed out input and simply invoking bpf_filter_with_aux_data, waiting for it to return, then resetting VM memory back to the original state and running again is fair. Due to how many or conditions are used in the filter, and given the packet doesn’t match any, should mean we’re seeing the worst case performance (eg. evaluating all expressions). I’m not perfectly familiar with BPF filtering, but I’d imagine there’s an early exit on a match, and thus if the destination was 1.2.3.4, I’d suspect the performance would be improved. Without this being clarified from the ToB blog, we’re just going with worst case (unless I’m incorrect in my understanding of BPF filters, maybe there’s no early exit).

Anyways, the target code that I’m using is as such:

use std::time::Instant;
use pcap_sys::*;

#[link(name = "pcap")]
extern {
    fn bpf_filter_with_aux_data(
        pc: *const bpf_insn,
        p:  *const u8,
        wirelen: u32,
        buflen:  u32,
        aux_data: *const u8,
    );
}

#[no_mangle]
pub extern fn fuzz_external() {
    const ITERS: u64 = 1;

    unsafe {
        let mut program: bpf_program = std::mem::zeroed();

        // Ethernet linktype + 1500 snapshot length
        let pcap = pcap_open_dead(1, 1500);
        assert!(!pcap.is_null());

        // Compile the program
        let status = pcap_compile(pcap, &mut program,
            "dst host 1.2.3.4 or tcp or udp or ip or ip6 or arp or rarp or \
            atalk or aarp or decnet or iso or stp or ipx\0"
            .as_ptr() as *const _,
            1, PCAP_NETMASK_UNKNOWN);
        assert!(status == 0, "Failed to compile pcap thingy");

        let buf = vec![0x41u8; 1500];

		// Filter a packet
		bpf_filter_with_aux_data(
			program.bf_insns,
			buf.as_ptr(),
			buf.len() as u32,
			buf.len() as u32,
			std::ptr::null()
		);
    }
}

fn main() {
    fuzz_external();
}
This is effectively the same as above, but it no longer loops. But, since I’m using a binary-based snapshot fuzzer, and so are they, we’re going to actually snapshot it. So, instead of running this entire program every fuzz case, I’m going to put a breakpoint on the first instruction of bpf_filter_with_aux_data, and run the RISC-V JIT until it hits it. Once it hits that breakpoint, I will make a snapshot of the memory state, and at that point I will create threads which will work on executing it in a loop.

Further, I will add another breakpoint on the return site of bpf_filter_with_aux_data to immediately terminate the fuzz case upon return. This avoids having the program do cleanup (like freeing buf), and otherwise bubbling up to an exit() syscall. Their blog isn’t super clear about this, but from their wording, I suspect this is a pretty similar setup. Effectively, only bpf_filter_with_aux_data is executing, and once it is not, the VM is reset and run again.

My emulator has many different operating modes. I have different coverage levels (covering blocks, covering PCs, etc), different levels of memory protection (eg. byte-level permissions which cause every byte to have its own permissions), uninitialized memory tracking (accessing allocated memory and stacks is invalid unless it has been written to first), as well as register taint tracking (logging when user input affected register state for both register reads and writes).

Since many of these vary in performance, I’ve set up a few tests with a few common configurations. Further, I’ve provisioned a 60 core c2-standard-60 (30 Cascade Lake Intel cores, totalling 60 hyper-threads) machine from Google Cloud Project to try to apples-to-apples as best I can. This machine costs $3.1321/hour, and thus, we’ll have to divide by these costs to make it fair when we do dollar-based comparisons.

Here… we… go!

image

Okay cool, so what is this graph telling us? Well, it’s showing us the number of iterations per second per core on the Y axis, against the number of cores being used on the X axis. This is not just telling me the overall performance, but also the scaling performance of the fuzzer, or how well it uses cores.

We’re going to ignore all lines other than the top line, the one in purple (blue?). We see that the line is relatively flat until 30 cores, then it starts falling off. This is great! This lines up with ideally what we want. The emulator is scaling linearly as cores are added, until we start getting past 30 cores, where they become hyperthreads and they’re not actually physical cores. The fact that the line is flat until 30 cores makes me very happy, and a lot of painstaking engineering went into making that work!

Anyways, we have multiple lines here. The top line, to no surprise, is gathering no coverage information, isn’t tracking taint, nor is it checking permissions. Of course it’s the fastest. The next line, in green, only adds block-level code coverage. It’s almost no performance hit, and nor would I expect it to be. The JIT self-modifies once coverage has been reported, and thus the only cost is a bit of icache pollution due to some nopped out code being jumped over.

Next, we have the light blue line, which at this stage, is the first line that actually matters. This one adds checking of permissions, as well as uninitialized memory tracking. This is done at a byte-level, and thus behaves very similarly to ASAN (in fact, it allows arbitrary byte-sized holes in memory, where ASAN can only mark trailing bytes as inaccessible). This of course, has a performance cost. And this is the real line, there’s no way I’d ever run a fuzzer without permission checks as the target would simply crash the host. I could use a more relaxed permission checking model (like using the hardware MMU on Intel to provide 512-byte-level permissions (4096-byte pages / 8 VMs interleaved per page)), and I’d have the green line in performance, but it’s not worth it. Byte level is too important to me.

Finally, we have the orange line. This one adds register “taint” tracking. This effectively horizontally looks at neighboring VMs during execution to determine if one VM has written or read a different value to a register. This allows me to observe and feed back information about which register values are influenced by the user input, and thus is important information for cutting down on mutation wastes. That being said, we’re not mutating, so it doesn’t really matter, we’re just looking at the runtime costs of this instrumentation.

Where does this leave us? Well, we see that on the 60 core machine, with the light blue line (the one we care about), we end up getting about 4.1 million iterations per second per core. Since we’re running 60 cores (technically 60 threads) at this rate, we can just multiply to see that we’re getting about 250 million iterations per second on this 60 core c2-standard-60 machine.

Well, this is the number we want. What does this come out to for iterations/second/$? Simply divide 250 million by $3.1321/hour, and we get about 79.8 million iters/second/dollar/hour.

I don’t have access to their GPU code so I can’t reproduce it, but their number they claim is 8.4M iterations/second on the $0.35/hour GPU, and thus, 23.9 million iters/second/dollar/hour.

This gives vectorized emulation about a 3x advantage for performance per dollar compared to the GPU based compute. It’s important to note, both technologies have some pretty large improvements to performance which may be possible. I suspect with some optimization both could probably see 2-3x improvements, but at that point they start hitting some very real hardware limitations in performance.

Where does this leave us?
I have some suspicions that GPUs will struggle with low latency memory accesses, especially when so many VMs are diverging and doing different things. These benchmarks are best case for both these technologies, as the inputs aren’t affecting execution flow, and the memory utilization is quite low.

GPUs have some major memory limitations, that I think make them impractical for fuzzing. As mentioned in the ToB blog, a 16 GiB GPU running 40,000 threads only has 419 KiB per thread available for storage. This means the corpuses, coverage databases, and all modified memory by a fuzz case must be below 419 KiB. This unfortunately isn’t a very practical limit. Right now I’m doing some freetype2 fuzzing in light of the Google Project Zero CVE-2020-15999, and I’m pusing 50 GiB of memory use for the 1,536 VMs I run. Vecemu does memory deduplication and CoW for all memory, and thus my memory use is quite low. Ultimately, there are user-controlled allocations that occur and re-claiming the memory every fuzz case doesn’t prove very feasible. This is also a tiny target, I fuzz many targets where the input alone exceeds 1 MiB, let alone other memory used by the target.

Nevertheless, I think these problems may be solvable with creative use of transferring memory in blocks, or maybe chunking fuzz cases into sections which use less than 400 KiB at a time, or maybe just reduce the number of threads. There’s definitely solutions here, and I definitely don’t doubt that it’s possible, but I do wonder if the overheads and complexities beat what can be done directly on the CPU with massive caches and access to all memory at a relatively low cost (as opposed to GPU<->CPU memory access).

Is there more perf?
It’s important to note that my vectorized emulation is not running faster than native execution. I’m still emulating RISC-V and applying some really strict memory permission checks that slow things down, this makes my memory accesses really expensive. I am happy to see though, that vectorized emulation looks to be within about ~3x of native execution (18M packets/second in our native libpcap harness mentioned early on, 5.5M with ours). This is pretty crazy, given we’re working with binaries and applying byte-level permissions to a target which isn’t even supported by ASAN! How cool is that!?

Vectorized emulation runs close to or faster than native execution when the target has few memory loads and stores. This is by far the bottleneck (~80%+ of CPU time is spent doing my memory translations). Doing some optimization passes to reduce memory loads and stores in my IL would probably allow me to realize some of these gains.

Since I’m not running at native speeds, we know that this isn’t as fast as could be done by just building libpcap for x86 and running it. Of course this requires source, but we know that we can get about a 3x speedup by fuzzing it natively. Thus, if I have a 3x improvement on the GPU fuzzing cost effectiveness, and there’s a 3x speedup from my emulation to just “running it natively on x86”, then there’s a 9x improvement from GPU execution to just run it natively.

This kinda… proves my earlier point. The benchmark is not comparing libfuzzer to the GPU fuzzer, it’s comparing the GPU fuzzer running a target, compared to libfuzzer performing orchistration of a fuzzer and mutations. It’s just… not really comparing anything valuable. But of course, like I always complain about, public fuzzer performance is often not great. There are improvements we can get to our fuzzing harnesses, and as always, I implore people to explore the powers of in-memory, snapshot based fuzzing! Every time you do IPC, update an atomic, update/check a database, do an allocation, etc, you lose a lot of performance (when running at these speeds). For example, in vectorized emulation for this very blog, I had to batch my fuzz case increments to only happen a few times a second. Having all threads updating an atomic ~250M times a second resulted in about a 60% overall slowdown of the entire harness. When doing super tight loop fuzzing like this (as uncommon as it may be), the way we write fuzzing harnesses just doesn’t work.

But wait… what even are these dollar amounts?
So, it seems that vectorized emulation is only slightly faster than the GPU results (~3x). Vectorized emulation also has years of research into it, and the GPU research is fairly new. This 3x advantage is honestly not a big deal, it’s below the noise floor of what really matters when it comes to accessibility of hardware. If you can get GPUs or GPU developers easier than AVX-512 CPUs and developers, the 3x difference isn’t going to make a difference.

But we have to ask, why are we comparing dollar amounts? The dollar amounts are largely to determine what is most cost effective, that makes sense. But… something doesn’t seem right here.

The GPU they are using is an NVIDIA Tesla T4 and costs $0.35/hour on Google Cloud Project. The CPU they are using (for libfuzzer) is a quad core Skylake which costs $0.38/hour, or almost 10% more. What? An NVIDIA Tesla T4 is $2,152 (cheapest price I could find), and a quad core Skylake is $150. What the?

Once again, I hate the cloud. It’s a pretty big ripoff for long-running compute, but of course, it can save you IT costs and allow you to dynamically spin up.

But, for funsies, let’s check the performance per dollar for people who actually buy their hardware rather than use cloud compute.

For these benchmarks I’m going to use my own server that I host in my house and purchased for fuzzing. It’s a quad socket Xeon 6252N, which means that in total it has 96 cores and 192 threads, clocking at 2.3 GHz base, turboing to 3.6 GHz. The MSRP (and price I paid) for these processors is $1788. Thus, ~$7,152 for just the processors. Throw in about $2k for a server-grade chassis + motherboard + power supplies, and then ~$5k for 768 GiB of RAM, and you get to the $14-15k mark that I paid for this server. But, we’ll simplify it a bit, we don’t need 768 GiB of RAM for our example, so we’ll figure out what we want in GPUs.

For GPUs, the Tesla T4s are $2,152 per GPU, and have 16 GiB of RAM each. Lets just ignore all the PCI slotting, motherboards, and CPU required for a machine to host them, and we’ll just say we build the cheapest possible chassis, motherboard, PSU, and CPUs, and somehow can socket these in a $1k server. My server is about $9k just for the 4 CPUs + $2k in chassis and motherboards, and thus that leaves us with $8k budget for GPUs. Lets just say we buy 4 Tesla T4s and throw them in the $1k server, and we got them for $2k each. Okay, we have a 4 Tesla T4 machine and a 4 socket Xeon 6252N server for about $9k. We’re fudging some of the numbers to give the GPUs an advantage since a $1k chassis is cheap, so we’ll just say we threw 64 GiB into the server to match the GPUs ram and call it “even”.

Okay, so we have 2 theoretical systems. One with 96C/192T of Xeon 6252Ns and 64 GiB RAM, and one with 4 Tesla T4s with 64 GiB VRAM. They’re about $9k-$11k depending on what deals you can get, so we’ll say each one was $9k.

Well, how does it stack up?

I have the 4x 6252N system, so we’ll run vectorized emulation in “light blue” line mode (block coverage, byte-level permissions, uninitialized mem tracking, and no register taint tracking), this is a common mode for when I’m not fuzzing too deep on a target. Well, lets light up those cores.

lolcores

Sweet, we’re under 10 GiB of memory usage for the whole system, so we’re not really cheating by skimping on the memory in our theoretical 64 GiB build.

Well, we’re getting about 700 million fuzz cases per second on the whole system. Woo! That’s a shitton! That is 77k iters/second/$. Obviously this seems “lower” than what we saw before, but this is the iters/second for a one time dollar investment, not a per-hour cloud fee.

So… what do we get on the GPU? Well, they concluded with getting 8.4 million iters/sec on the cloud compute GPU. Assuming it’s close to the performance you get on bare metal (since they picked the non-preemptable GPU option), we can just multiply this number by 4 to get the iters/sec on this theoretical machine. We get 33.6 million iterations per second total, if we had 4 GPUs (assuming linear scaling and stuff, which I think is totally fair). Well… that’s 3,733 iters/second/$… or about 21x more expensive than vectorized emulation.

What gives? Well, the CPUs will definitely use more power, at 150W each you’ll be pushing 600W minimum, but I observe more in the ballpark of 1kW when running this server, when including peripherals and others. The Tesla T4 is 70W each, totalling 280W. This would likely be in a system which would be about 200W to run the CPU, chassis, RAM, etc, so lets say 500W. Well, it’d be about 1/2 the wattage of the CPU-based solution. Given power is pretty cheap (especially in the US), this difference isn’t too major, for me, I pay $0.10/kWh, thus the CPU server would cost about $0.20 per hour, and the GPU build would cost about $0.10 per hour (doubled for cooling). These are my “cloud compute” runtime costs, and thus the GPUs are still about 10x more expensive to run than the CPU solution.

Conclusion
As I’ve mentioned, this GPU based fuzzing stuff is incredibly cool. I can’t wait to see more. Unfortunately, some of the methodologies of the comparison aren’t very fair and thus I think the claims aren’t very compelling. It doesn’t mean the work isn’t thrilling, amazing, and incredibly hard, it just means it’s not really time yet to drop what we’re doing to invest in GPUs for fuzzing.

There’s a pretty large discrepency in the cost effectiveness of GPUs in the cloud, and this blog ends up getting a pretty large advantage over libfuzzer for something that is really just a pricing decision by the cloud providers. When purchasing your own gear, the GPUs are about 10x more expensive than the CPUs that were used in the blogs tests (quad-core Skylake @ $200 or so vs a NVIDIA T4 @ $2000). The cloud prices do not reflect this difference, and in the cloud, these two solutions are the same price. That being said, those are real gains. If GPUs are that much more cost effective in the cloud, then we should definitely try to use them!

Ultimately, when buying the hardware, the GPU solution is about 20x less cost effective than a CPU based solution (vectorized emulation). But even then, vectorized emulation is an emulator, and slower than native execution by a factor of 3, thus, compared to a carefully crafted, low-overhead fuzzer, the GPU solution is actually about 60x less cost effective.

But! The GPU solution (as well as vectorized emulation) allow for running closed-source binary targets in a highly efficient way, and that definitely is worth a performance loss. I’d rather be able to fuzz something at a 10x slowdown, than not being able to fuzz it at all (eg. needing source)!

Hats off to everyone at Trail of Bits who worked on this. This is incredibly cool research. I hope this blog didn’t come off as harsh, it’s mainly just me recording my thoughts as I’m always chasing the best solution for fuzzing! If that means I throw away vecemu to do GPU-based fuzzing, I will do it in a heartbeat. But, that decision is a heavy one, as I would need to invest thousands of hours in GPU development and retool my whole server room! These decisions are hard for me to make, and thus, I have to be very critical of all the evidence.

I can’t wait to see more research from you all! This is incredible. You’re giving me a real run for my money, and in only 2 months of work, fucking amazing! See you soon!

Random opinions
I’ve been asked a few things about my opinion on the GPU-based fuzzing, I’ll answer them here.

Is not having syscalls a problem?
No. It’s not. It is for people who want to use the tool. But this is a research tool and is for exploring what is possible, the act of fuzzing on a GPU by running binary translated code is incredible, that’s the focus here! GPUs are turing complete, we can definitely emulate syscalls on them if needed. It might be a lot of work, a lot of plumbing, maybe a perf hit, but it doesn’t stop it from being possible. Most of my fuzzers rely on emulating syscalls.

There’s also nothing preventing GPUs from being used to emulate an whole OS. You’d have to handle self-modifying code and virtual memory, which can get very expensive in an emulator, but with making software TLBs these things can be manageable to a level it’s still worth doing!

Social
I’ve been streaming a lot more regularly on my Twitch! I’ve developed hypervisors for fuzzing, mutators, emulators, and just done a lot of fun fuzzing work on stream. Come on by!

Follow me at @gamozolabs on Twitter if you want notifications when new blogs come up. I often will post data and graphs from data as it comes in and I learn!

Gamozo Labs Blog
Gamozo Labs Blog
b@gamozolabs.com
 gamozolabs
 gamozolabs
I blog about random things security, everything is broken, nothing scales, shared memory models are flawed.

Gamozo Labs Blog
About
Some thoughts on fuzzing
Aug 11, 2020

Foreward
This blog is a bit weird, this is actually a message I posted in response to a fuzzbench issue, but honestly, I think it warranted a blog, even if it’s a bit unpolished!

You can find the discussion at fuzzbench issue tracker #654

Social
I’ve been streaming a lot more regularly on my Twitch! I’ve developed hypervisors for fuzzing, mutators, emulators, and just done a lot of fun fuzzing work on stream. Come on by!

Follow me at @gamozolabs on Twitter if you want notifications when new blogs come up. I often will post data and graphs from data as it comes in and I learn!

The blog
Hello again Today!

So, I’d like to address a few things that I’ve thought of a bit more over time and want to emphasize.

Visualizations, and what I’m often looking for in data
When it comes to visualizations, I don’t really mind much which graphs are displayed by default, linear vs logscale, time-based or per-case-based, but they should both be toggleable in the default. I’m not web dev, but having an interactive graph would be pretty nice, allowing for turning on and off of certain lines, zooming in and out, and changing scales/axes. But, I think we’re in agreement here. I personally believe that logscale should be default, and I don’t see how it’s anything but better, unless you only care about seeing where things “flatten out”. But in that case, it’s just as visible in logscale, you just have to be logscale aware.

Here’s an example of typically what I graph when I’m running and tuning a fuzzer. I’m using doing side-by-side comparisons of small fuzzer tweaks, to my prior best runs, and plotting both on a time domain and a fuzz case domain. I’ve included the linear-scale plots just for comparison with the way we currently do things, but I personally never use linear scale as I just find it to be worse in all aspects.

image

By using a linear scale, we’re unable to see anything about what happens in the fuzzer in the first ~20 min or so. We just see a vertical line. In the log scale we see a lot more which is happening. This graph is comparing a fuzzer which does rand() % 20 rounds of mutation (medium corruption), versus rand % 5 rounds of the same corruption (low corruption). We can see that early on medium corruption has much better properties, as it explores more aggressively. But there’s actually a point where they cross, and this is likely the point where the corruption becomes too great on average in the medium corruption, and ends up “ruining” previously good inputs, dramatically reducing the frequency we see good cases. It’s important to note, that since the medium corruption is a superset of low corruption (eg, there’s a chance to do low corruption), both graphs would eventually converge to the exact same value.

There’s just so much information in this graph that stands out to me. I see that something about medium corruption performs well in the first ~100 seconds. There’s a really good lead at the early few seconds, and it tapers off throughout. This gives me feedback on maybe when and where I should use this level of corruption.

Further, since I have both a fuzz case graph and a time graph, I can see that medium corruption early on actually has better performance than low corruption. Once again, this makes sense, the more corruption, the more likely you are to make a more invalid input which is parsed more shallow. But from the coverage over case, I see that this isn’t a long term thing and eventually the performance seems to converge between the two. It’s important to note, the intersection point of the two lines varies by quite a bit in both the case domain and the time domain. This tells me that even though I just changed the mutator, it has affected the performance, likely due to the depth of the average input in the corpus, really neat!

Example analysis conclusion
I see that medium corruption in this case is giving me about 10x speedup in time-to-same-coverage, and also some performance benefits early on. I should adopt a dynamic corruption model which tunes this corruption amount maybe against time, or ideally, some other metric I could extract from the target or stats. I see that long-term, the low corruption starts to win, and for something that I’d run for a week, I’d much rather run the low corruption.

Even though this program is very simple, these graphs could pretty arbitrary be stretched out to different time axis. If fuzzbench picks a deadline, for example, 1000 seconds, we would never know this about the fuzzer performance. I think this is likely what many fuzzers are now being tuned to, as the benchmarks often are 12/24/72 hour increments. Fuzzers often get some extra blips even deeper in the runs, and it’s really hard to estimate if these crosses would ever occur.

The case for cases
I personally extract most information from graphs which are plotted against number of fuzz cases rather than time. By doing benchmarks in a time domain, you factor in the performance of the fuzzer. This is this ground truth, and what really matters at the end of the day with complete products. But it’s not the ground truth for fuzzers in development. For example, if I wanted to prototype a new mutation strategy for AFL, I would be forced to do it in C, avoid inefficient copies, avoid mallocs, etc. I effectively have to make sure my mutator is at-or-better than existing AFL mutator performance to use benchmarks like this.

When you do development on fuzz cases, you can start to inspect the efficiency of the fuzzer in terms of quality of cases produced. I could prototype a mutator in python for all I care, and see if it performs better than the stock AFL mutators. This allows me to cut corners and spend 1 day trying out a mutator, rather than 1 month making a mutator and then doing complex optimizations to make it work. During early stages of development, I would expect a developer to understand the ramifications of making it faster, and to have a ballpark idea of where it could be if the O(n^3) logic was turned into O(log n), and whether it’s possible.

Often times, the first pass of an attempt is going to be crude, and for no reason other than laziness (and not in a negative way)! There’s a time and a place to polish and optimize a technique, and it’s important that there can be information learned from very preliminary results. Most performance in standard feedback mechanisms and mutation strategies can be solved with a little bit of engineering, and most developers should be able to gauge the best-case big-O for their strategy, even if that’s not the algorithmic complexity of their initial implementation.

Yep, looking at coverage over cases adds nuance, but I think we can handle it. Given most fuzzing tools, especially initial passes, are already so un-optimized, I’m really not worried about any performance differences in AFL/libfuzzer/etc when it comes to single-core performance.

Scaling
Scaling of performance is really missing from fuzzbench. At every company I’ve worked at, big and small, even for the most simple targets we’re fuzzing we’re running at least ~50-100 cores. I presume (I don’t know for sure) that fuzzbench is comparing single core performance. That’s great, it’s a useful stat and one I often control for, as single-core, coverage/case is often controlled for both scaling and performance, leading to great introspection into the raw logic of the fuzzer.

However, in reality, the scaling of these tools is critical for actual use. If AFL is 20% faster single-core, that’ll likely make it show up at the top of fuzzbench, given relative parity of mutation strategies. That’s great, the performance takes engineering effort and should not be undervalued. In fact, most of my research is focused around making fuzzers fast, I’ve got multiple fuzzers that can handle 10s of billions of fuzz cases per second on a single machine. It’s a lot of work to make these tools scale, much more so than single-core performance, which is often algorithmic fixes.

If AFL is 20% faster single-core, but bottlenecks on fork(), or write(), and thus only scales to 20-30 cores (often where I see AFL really fall apart, on medium size targets, 5-10 cores for small targets). But something like libfuzzer manages things in memory and can scale linearly with as many cores as you throw it, libfuzzer is going to blow away any 20% performance gains seen single-core.

This information is very hard to benchmark. Well, not hard, but costly. Effectively, I’d like to see benchmarks of fuzzers scaled to ~16 cores on a single server, and ~128 cores distributed across at least 4 servers. This benchmarks. A. the possibility the fuzzer can scale in the first place, if it can’t that’s a big hit to real-world usability. B. the possibility it can scale across servers (often, over the network). Things like AFL-over-SMB would have brutal scaling properties here. C. the scalability properties between cores on the same server, and how they transfer over the network.

I find it very unlikely that these fuzzers being benchmarked even remotely have similar scaling properties. AFL struggles to scale even on a single server, even in persistent mode, due to the heavy use of syscalls and blocking IPC every fuzz case (signal(), read(), write(), per case IIRC, ~3-4 syscalls).

Scaling also puts a lot of pressure on infeasible fuzzing strategies proposed in papers. We’ve all seen them, the high-introspection techniques which extract memory, register, taint state from a small program and promise great results. I don’t disagree with the results, the more information you extract, pretty much directly correlates to an increase in coverage/case. But, eventually the data load gets very hard to share between cores, queue between servers, and even just process.

Measuring symbolic
Measuring symbolic was brought up a few times, as it would definitely have a much better coverage/case than a traditional fuzzer. But this nuance can easily be handled by looking at both coverage/case and coverage/time graphs. Learning what works well algorithmicly should drive our engineering efforts to solve problems. While symbolic may have huge performance issues, it’s very likely, that many of the parts of it (eg. taint tracking) can be approximated with lossy algorithms and data capturing, and it’s more about learning where it has strengths and weaknesses. Many of the analyses I’ve done on symbolic largely lead me to vectorized emulation, which allows for highly-compressed, approximated taint tracking, while still getting near-native (or even better) execution speeds.

The case against monolithic fuzzers
Learning what works is important to figure out where to invest our engineering time. Given the quality of code in fuzzing right now (often very poor), there’s a lot of things that I’d hate to see us rule out because our current methodologies do not support them. I really care about my reset times of fuzz cases, (often: the fork() costs), as well as determinism. In a fully deterministic environment, with fast resets, a lot of approximate strategies can be used. Trying to approximate where bytes came from in an input, flipping the bytes because you have a branch target which is interesting, and then smashing those bytes in can give good information about the relation of those bytes to the input. Hell, with fast resets and forking, you can do partial fuzzing where you fork() and snapshot multiple times during a fuzz case, and you can progressively fuzz “from” different points in the parser. This works especially well for protocols where you can snapshot at each packet boundary.

These sorts of techniques and analyses don’t really work when we have monolithic fuzzers. The performance of existing fuzzers is often quite poor (AFL fork(), etc), or does not support partial execution (persistent modes, libfuzzer, etc). This leads to us not being able to even research these techniques. As we keep bolting things onto existing fuzzers and treating them like big blobs, we’ll get further and further from being able to learn the isolated properties of fuzzers and find the best places to apply certain strategies.

Why I don’t care much about fuzzer performance for benchmarking
Reset speed
AFL fork() bottlenecks for me often around 10-20k execs/sec on a single core, and about 40-50k on the whole system, even with 96C/192T systems. This is largely due to just getting stuck on kernel allocations and locks. Spinning up processes is expensive, and largely out of our control. AFL allows access of the local system and kernel to the fuzz case, thus cases are not deterministic, nor are they isolated (in the case of fuzzing something with lock files). This requires using another abstraction layer like docker, which adds more overhead to the equation. My hypervisors that I use for fuzzing can reset a Windows VM 1 million times per second on a single core, and scale linearly with cores, while being deterministic. Why does this matter? Well, we’re comparing tooling which isn’t even remotely hitting the capabilities of the CPUs, rather they’re bottlenecking on the kernel. These are solvable problems, and thus, as a consumer of good ideas but not tooling, I’m interested in what works well. I can make it go fast myself.

Determinism
Most fuzzers that we work with now are not deterministic. You cannot expect instruction-for-instruction determinism between cases. This makes it a lot harder to use complex fuzzing strategies which may rely on the results of a prior execution being identical to a future one. This is largely an engineering problem, and can be solved in both system-level and app-level targets.

Mutation performance
The performance of mutators is often not what it can be. For example, honggfuzz used (now fixed, cheers!) temporary allocations during multiple passes. During its mangle_MemSwap it made a copy of the chunk that was to be swapped, performing 3 memcpys and using a temporary allocation. This logic was able to be implemented using a single memcpy and without a dynamic allocation. This is not a criticism of honggfuzz, but more of an important note of how development often occurs. Early prototyping, success, and rare revisiting of what can be changed. What’s my point here? Well, the mutation strategies in many fuzzers may introduce timing properties that are not fundamentally required to have identical behaviors. There’s nothing wrong with this, but it is additional noise which factors into time-based benchmarks. This means a good strategy can be hurt by a bad implementation, or, just a naive one that was done early on. This is noise that I think is big to remove from analysis such that we can try to learn what ideas work, and engineer them later.

Further, I don’t know of any mutational fuzzer which doesn’t mutate in-place. This means the multiple splices and removals from an input must end up memcpy()ing the remainder. This is a very common mutation pass. This means the fuzzer exponentially slows down WRT the input file size. Something we see almost every fuzzer put insane restrictions on (AFL has a fit if you give it anything but a tiny file).

There’s nothing stopping us from making a tree-based fuzzer where a splice adds a node to the tree and updates metadata on other nodes. The input could be serialized once when it’s ready to be consumed, or even better, serialized on-demand, only providing the parts of the file which actually were used during the fuzz case.

Example:

Initial input: "foobar", tree is [pointer to "foobar", length 6]
Splice "baz" at 3: [pointer to "foo", length 3][pointer to "baz", length 3][pointer to "bar", length 3]
Program read()s 3 bytes, return "foo" without serializing the rest
Program crashes, tree can be saved or even just what has read can be saved
In this, the cost is N updates to some basic metadata, where N is the number of mutations performed on that input (often 5-10). On a new fuzz case, you start with an initial input in one node of the tree, and you can once again split it up as needed. Pretty much no memcpys() need to be performed, nor allocations, as the input can be extended such that in-memory it’s “foobarbaz”, but the metadata describes that the “baz” should come between “foo”, and “bar”.

Restructuring the way we do mutations like this allows us to probably easily find 10x improvements in mutator performance (read, not overall fuzzer performance). Meaning, I don’t really want the cost of the mutator to be part of the equation, because once again, it’s likely a result of laziness or simplicity. If something really brings a strategy to the table that is excellent, we can likely make it work just as fast (but likely even faster), than existing strategies.

Not to note the value in potentially knowing which mutations were used during prior cases, and you could potentially mutate this tree (eg, change a splice from 5 bytes to 8 bytes, without changing the offset, just changing the node in the mutation tree). This could also be used as a mechanism to dynamically weight mutation strategies based on yields, while still getting a performance gain over the naive implementation.

Performance conclusion
From previous work with fuzzers, most of the reset, overhead, and corruption logic is likely not even within an order of magnitude of the possible performance. Thus, I’m far more interested in figuring out what and where strategies work, as the implementations of them are typically not indicative of their performance.

BUT! I recognize the value in treating them as whole systems. I’m a bit more on the hard-core engineering side of the problem. I’m interested in which strategies work, not which tools. There’s definitely value in knowing which tools will work best, given you don’t have the time to tweak or rebuild them yourself. That being said, I think scaling is much more important here, as I don’t know of really anyone doing single-core fuzzing. The results of these fuzzers at scale is likely dramatically different from single-core, and would put some major pressure on some more theoretical ideas which produce way too much information to consume and handle.

Reconstructing the full picture from data
The data I would like to see fuzzbench give, and I’d give you some massive props for doing it, would be the raw, microsecond-timestamped information for each coverage gained.

This means, every time coverage increases, a new CSV record (or whatever format) is generated, including the time stamp when it was found (to the microsecond), as well as the fuzz iteration ID which indicates how many inputs have been run into the fuzzer. This should also include a unique identifier of the block which was found.

This means, in post, the entire progress of the fuzzer can be reconstructed. Every edge, which edges they were, the times they were found, and the case ID they were on when they were found allows comparing not only the raw “edge count” but also the differences between edges found. It’s crazy that this information is not part of the benchmark, as almost all the fuzzers could be finding nearly the same coverage, but a fuzzer which finds less coverage, but completely unique edges, would be devalued.

This is the firehose of data, but since it’s not collected on an interval, it very quickly turns into almost no data.

Hard problem: What is coverage?
This leads to a really hard problem. How do we compare coverage between tools? Can we safely create a unique block identifier which is universal between all the fuzzers and their targets. I have no idea how fuzzbench solves this (or even if it does). If fuzzbench is relying on the fuzzers to have roughly the same idea of what an edge is, I’d say the results are completely invalid. Having different passes which add different coverage gathering, compare information gathering, could easily affect the graphs. Even just non-determinism in clang (or whatever compiler) would make me uneasy about if afl-clang binaries have identical graph shapes to libfuzzer-clang binaries.

If fuzzbench does solve this problem, I’m curious as to how. I’d anticipate it would be through a coverage pass which is standardized between all targets. If this is the case, are they using the same binaries? If they’re not, are the binaries deterministic, or can the fuzzers affect the benchmark coverage information due to adding their own compiler instrumentation.

Further, if this is the case, it makes it much harder to compare emulators or other tools which gather their own coverage in a unique way. For example, if my emulators, which get coverage for effectively free, had to run an instrumented binary for fuzzbench to get data, it’s not a realistic comparison. My fuzzer would be penalized twice for coverage gathering, even though it doesn’t need the instrumented binary.

Maybe someone solved this problem, and I’m curious what the solution is. TL;DR: Are we actually comparing the same binaries with identical graphs, and is this fair to fuzzers which do not need compile-time instrumentation.

The end
Can’t wait for more discussion. You have been very receptive even when I’m often a bit strongly opinion-ed. I respect that a lot.

Stay cute,

gamozo

Gamozo Labs Blog
Gamozo Labs Blog
b@gamozolabs.com
 gamozolabs
 gamozolabs
I blog about random things security, everything is broken, nothing scales, shared memory models are flawed.
