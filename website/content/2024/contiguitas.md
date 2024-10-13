+++
# The title of your blogpost. No sub-titles are allowed, nor are line-breaks.
title = "Contiguitas: The Pursuit of Physical Memory Contiguity in Datacenters"
# Date must be written in YYYY-MM-DD format. This should be updated right before the final PR is made.
date = 2024-10-08

[taxonomies]
# Keep any areas that apply, removing ones that don't. Do not add new areas!
areas = ["Systems"]
# Tags can be set to a collection of a few keywords specific to your blogpost.
# Consider these similar to keywords specified for a research paper.
tags = ["Virtual Memory", "Memory Fragmentation", "Address Translation"]

[extra]
author = {name = "Kaiyang Zhao", url = "http://www.cs.cmu.edu/~kaiyang2/" }
# The committee specification is  a list of objects similar to the author.
committee = [
    {name = "Phillip B. Gibbons", url = "http://www.cs.cmu.edu/~gibbons/"},
    {name = "Todd C. Mowry", url = "https://www.toddcmowry.org"},
    {name = "Sara McAllister", url = "https://saramcallister.github.io"}
]
+++

Applications access data in the memory using virtual addresses, but the virtual memory abstraction is not free. In fact, the unabating growth of the memory needs of emerging datacenter applications has exacerbated the scalability bottleneck of virtual memory. However, reducing the overhead of address translation relies on physical memory contiguity, which is in short supply. To address the problem, *Contiguitas* provides ample physical memory contiguity through the synergy of operating system memory management and architecture support. This blog post is adapted from a published [paper](https://dl.acm.org/doi/10.1145/3579371.3589079).

# Introduction

## The Worsening Virtual Memory Bottleneck

Memory capacity has increased dramatically over the last few decades, increasing from dozens of kilobytes to terabytes, yet modern operating systems have throughout stuck with a small base page size that is a relic from earlier times.
This divergence has resulted in excessive management overhead for memory-intensive applications. 
In particular, virtual memory implementations are plagued by expansive page table trees to cover the increasing memory usage by applications, and to cache the ever-larger page tables, a corresponding appetite for hardware Translation Lookaside Buffers (TLB) capacity that is difficult to satiate.
Even with hardware innovations such as larger and multi-level TLB, applications today suffer a substantial performance penalty due to TLB misses. 

Case in point, as shown in the figure below, servers at Meta have their memory capacity increased by almost eight times over a few hardware generations. However, the number of TLB entries remains stagnant, leading to minuscule and shrinking TLB coverage.

![Memory and TLB coverage of computing hardware across generations.](./bg-mem-tlb.pdf)

Google's profiling further revealed that approximately 
20% of CPU cycles in their datacenters are stalled on TLB misses [1].

Unfortunately, this problem is only bound to get worse due to:  i) the inherent hardware limits of TLB size scaling, already surpassing L2 cache latencies, ii) terabyte-scale memory capacity through technologies like Compute Express Link (CXL),  iii) additional levels of page tables, and iv) the increase of memory-intensive applications such as those for graph processing, caching and scientific computing.

## Most Solutions Rely on Memory Contiguity

A large body of prior research has focused on reducing the virtual memory overhead. Conceptually, we can separate prior work based on the amount of physical memory contiguity required and how it is exploited.

On the one hand, earlier works propose leveraging physical memory contiguity to back the application dataset [2].
These approaches create range-based translations by relying on large contiguous physical memory. However, they face the fundamental challenge that it is very hard to create multi-gigabyte contiguous physical address ranges to cover the complete application dataset.

On the other hand, another line of research has explored alternative page table structures such as hashed page tables [3]. Such solutions aim to replace sequential multi-level page tables and drastically reduce the cost of page walks by accelerating page table accesses. Notably, they relax the physical memory contiguity requirements to apply not on the whole dataset, but only on the page table organization. However, they impose strict requirements for physical memory contiguity availability on the critical path of page table creation.

Several other architectural extensions that implicitly rely on contiguity [4, 5] are hindered by the same fundamental challenge.

Today's operating systems (OS), such as Linux, have mostly relied on 2MB Transparent Huge Pages (THP) that *opportunistically* provide 2MB pages to land performance improvements. Unfortunately in today's systems, finding physical contiguity even for 2MB pages is often hard due to memory fragmentation [6, 7]. THPs have also been under scrutiny due to performance implications such as latency spikes and memory bloating. Alternative approaches, such as userspace allocators, still rely on the OS to provide physical contiguity and larger mappings. In summary, physical memory contiguity is required to address the virtual memory bottleneck.

## Memory Contiguity is Scarce in Datacenters

We started with a detailed investigation of physical memory contiguity at hyperscale across Meta's datacenters. We sampled servers across the fleet and the memory fragmentation distribution is shown in the following figure.

![Contiguity availability as the percentage of free memory.](./bg-frag.pdf)

We see that 23% of servers do not even have physical memory contiguity for a single 2MB huge page. We also find that it is practically impossible to dynamically allocate 1GB pages in production.

Furthermore, our analysis shows that there is little to no correlation between memory contiguity and server uptime, with the Pearson correlation coefficient between server uptime and the number of free 2MB pages being only 0.00286.
Pertinently, this means that fragmentation affects all servers. In practice, servers can quickly get heavily fragmented within the first hour after boot-up while the mean server uptime is days or weeks---turning memory fragmentation into a major challenge.

Finally, our study exposes unmovable memory allocations as the root cause for the lack of physical memory contiguity. In particular, we identify several sources of unmovable allocations, including networking buffers, slab, filesystems, and page tables. 

## Our Work: Contiguitas

To address the lack of memory contiguity and alleviate the virtual memory overhead, we introduce **Contiguitas** with the goal of eliminating fragmentation due to unmovable allocations. Contiguitas separates movable allocations from unmovable ones by placing them into two different regions and dynamically adjusting the boundary of the two regions on demand. 
To avoid wasting memory in the unmovable region, Contiguitas solves two problems: i) how to dynamically resize the unmovable region and place unmovable allocations; and ii) how to drastically reduce unmovable allocations.

For the first problem, Contiguitas performs resizing by tracking the demand for unmovable allocations.
Moreover, it reduces internal fragmentation of the unmovable region by differentiating types of allocations. 

For the second problem, Contiguitas focuses on unmovable pages that cannot be moved with software alone because access to such pages cannot be blocked for a migration to take place. At Meta, networking allocations account for 73% of unmovable pages. We expect unmovable pages to become an increasingly bigger problem because of new input/output (I/O) technologies such as kernel-bypass and Remote Direct Memory Access (RDMA) for networking and storage, Graphics Processing Units (GPU), and other accelerators that heavily really on unmovable pages.

Our experiments in Meta's production datacenters
show that Contiguitas successfully confines unmovable allocations, leading to significant performance gains. Full-system simulations showcase the effectiveness of Contiguitas's hardware. We are currently in the process of upstreaming the software part of Contiguitas into Linux.

# Contiguitas Design

The goal of Contiguitas is to provide ample physical memory contiguity by reducing memory fragmentation due to unmovable allocations.
To that end, Contiguitas redesigns memory management in the OS to confine unmovable allocations and completely separate them from movable ones. 
In addition, Contiguitas drastically reduces unmovable pages in datacenters.
Specifically, Contiguitas introduces a set of hardware extensions in the LLC
that enable the transparent migration of unmovable pages while in use.

## Overview

![Contiguitas design overview.](./overview_main_v2.pdf)

First, Contiguitas redesigns memory management in the OS to confine unmovable allocations and completely separate them from movable ones (Step 1), preventing unmovable allocations from scattering across the address space. Then, Contiguitas dynamically resizes regions in response to memory pressure (Step 2). Finally, Contiguitas drastically reduces unmovable pages through hardware extensions in the LLC that enable transparent migration of unmovable pages while in use (Step 3).

## Unmovable Confinement

The first design principle of
Contiguitas is to strictly separate unmovable from movable allocations using two dedicated regions, the movable and the unmovable regions in the physical address space. 

Allocations are confined in their respective region. Contiguitas categorizes the physical pages based on their addresses and keeps them on distinct free lists for each region. Memory in each region can only be allocated from pages in the free lists belonging to that region. When a page is freed, it is returned back to its respective list. This approach simplifies the critical path of allocations as the OS can quickly pick a free page while avoiding mixing different types of allocations.
For allocations that are first allocated as movable but later become unmovable, Contiguitas migrates them to the unmovable region and marks them as unmovable. This approach avoids the dynamic pollution of the movable region and subsequent compaction failures.

The crucial part in designing confinement is the sizing of the unmovable region. If it is too big, unused memory in the unmovable region is wasted while there is limited movable memory for the  applications, causing frequent reclaims, swapping, or even allocation failures. On the other hand, if the unmovable region is too small, it may fail unmovable allocations. Therefore, Contiguitas dynamically balances the sizes of the movable and unmovable regions while not negatively affecting application performance. There are three major challenge in dynamic resizing.

### Challenge 1: Putting Resizing Off the Critical Path

The first challenge is to move resizing operations off the critical path of memory allocation.
Contiguitas performs resizing off the critical path of memory allocation to avoid latency overheads. This is accomplished by monitoring the amount of free memory when periodic memory reclaim is triggered by the kernel. Contiguitas extends reclaim to wake up a kernel thread to perform resizing when the free memory in either region falls below a low-watermark threshold.

### Challenge 2: Resizing Policies

The second challenge is to decide when and how much to resize.
Contiguitas introduces the concept of per-region memory pressure and extends Pressure Stall Information (PSI) [6] to track time wasted due to lack of free memory in the movable and unmovable region separately, which is then used to calculate the expansion/shrinking amount influenced by tunable parameters.

### Challenge 3: Making Resizing Cheap

The third challenge is to ease resizing and reduce data movement.
Contiguitas introduces a bias to prefer physical pages further away from the region border. 
Some unmovable allocations that are inherently long lived, e.g., kernel code pages, are safely placed by Contiguitas early on, at the end of the unmovable region that is farthest from the movable region. On the other hand, pages that are initially in the movable region and later on migrated to the unmovable region often exhibit short lifetimes.
In general, Contiguitas prefers allocating pages away from the region border as long as sufficient free space is available.
This approach increases the chance that resizing will be successful and need little data movement.

## Hardware Migration of Unmovables

The second design principle guiding 
Contiguitas is to drastically reduce the amount of unmovable allocations by turning them into movable ones. 
Our study at Meta's datacenters revealed that a significant portion of unmovable allocations used for input/output (I/O) are impossible to move as access to the page cannot be blocked for a software migration to take place.

Hardware support is required because it is impossible for software to move such pages as it cannot atomically perform both the translation update and the page copy operation. Hence, software has to block access to the page for the duration of page migration in order to avoid spurious writes to it. Even if access to the page could be blocked, software page migration induces a long downtime due to (a) TLB shootdowns that scale poorly with the number of victim TLBs, and (b) the page copy. 

To this end, Contiguitas-HW enables transparent page migration while the page remains in use. Such migrations can substantially reduce the size of the unmovable region and lead to more efficient defragmentation and memory management as the vast majority of pages can be moved on demand. While Contiguitas-HW is motivated by unmovable allocations, its design is  suitable for both movable and unmovable allocations.

The hardware extensions of Contiguitas-HW are located in the last-level cache (LLC) as shown in the figure below. Contiguitas-HW targets a multi-core processor with a cache-coherent interconnect. The hardware platform further includes an Input-Output Memory Management Unit (IOMMU) with local TLBs.

![Contiguitas hardware overview.](./arch_overview.pdf)

At a high level, Contiguitas-HW aliases a physical page under migration with a destination page and redirects appropriate traffic to the destination page based on the progress of the migration.
Specifically, a page migration is initiated in Step 1 by the OS that provides the source and destination physical page numbers (PPN) to the Contiguitas-HW. Contiguitas-HW stores them in a metadata table (part b in the figure) along a *Ptr* field that points to the next line to be copied, and enables access redirection.

The OS then modifies the page table entry to point to the destination page and starts local TLB invalidations, shown in Step 5. In Contiguitas-HW, TLB shootdowns do not require inter-processor interrupts (IPI), as both mappings are concurrently active during the migration.

During this process, a page may be accessed with either the source or the destination mapping. If a request hits in the private caches, it is serviced normally as in a regular cache hit. As shown in Step 4, on a miss the Contiguitas-HW checks whether a line is currently stored in private caches with the opposite mapping of the request i.e., if a request is for the source mapping and the line is stored with the destination mapping, and vice versa. If so, it invalidates any cached copy. Otherwise, the request is serviced regularly. This invariant allows the caching of lines under migration as only the source or the destination mapping is active in the private caches.

When the TLB invalidations are complete, the OS notifies the Contiguitas-HW to start the copy. At this point only the destination mapping is active at any given TLB. Contiguitas-HW copies a cacheline by bringing it into the LLC (Step 2) and copying it from the source to the destination (Step 3). Finally, it increments *Ptr*. This process continues until the page is copied and Contiguitas-HW notifies the OS. 

# Results: Ample Contiguity in Datacenters

We build the OS component of Contiguitas into Linux and run our experiments in Meta's production datacenters with four production workloads:  a continuous-integration service (CI), a web server (Web), and two caching services (Cache A and Cache B).

## Potential Memory Contiguity

To quantify the impact of Contiguitas on memory contiguity, we compare each workload's steady state under Linux and Contiguitas.
Specifically, we quantify the contiguous regions that can be formed if we run a perfect software compaction in order to service allocation requests of 2MB and 1GB pages.

![Potential memory contiguity as a percentage of total memory.](./eval-contiguity.pdf)

With Linux we see that some 2MB allocation are possible given that less than half the memory is composed of unmovable 2MB pages as we discussed above. 
However, Linux struggles as we search for larger contiguous regions, and fails to find even a single 1GB page. 
On the other hand, Contiguitas, by design, isolates the unmovable region and hence the whole movable region can potentially be used after compaction for large contiguous allocations, even 1GB
pages.

## End-to-End Performance

To measure Contiguitas's improvements to end-to-end performance due to increased memory contiguity, we use requests per second under certain latency SLAs based on the characteristics of each workload.
We consider two setups, *Full Fragmentation* and *Partial Fragmentation*.
*Full Fragmentation* represents the case where a workload lands on a server whose memory is already fully fragmented. *Partial Fragmentation* represents the case where a workload lands on a partially fragmented server that is representative of the majority of servers at Meta.

![End-to-end performance over Meta's production workloads.](./eval-app-perf.pdf)

Contiguitas achieves performance improvements between 2-9% for partially fragmented servers that represent the majority of the servers, and between 7-18% for highly fragmented servers that represent nearly a quarter of Meta's fleet. Notably, Contiguitas's contiguity gains enable Web, one of Meta's largest services, to dynamically allocate 1GB huge pages, leading to a 7.5% performance win (shown as the red bar) that is unattainable with 2MB pages alone. 

## Hardware Evaluation

We use full-system simulations to show that Contiguitas-HW efficiently migrates unmovable allocations without affecting application performance.
We consider two open source applications Memcached and NGINX to cover applications that do and do not benefit from huge page availability.

Even at a rate of 1000 pages migrated per second, which would be unwarranted for a real environment, Contiguitas-HW does not have an impact on both applications' performance.
When combined with the benefits of contiguity and 2MB huge pages, Memcached performance improves by 7%. 

Furthermore, Contiguitas-HW scales well with the number of TLBs, keeping the page unavailable time during a page migration constant and equal to a local TLB invalidation, whereas under status quo the page unavailable time increases linearly. 

Overall, Contiguitas-HW does not negatively impact applications that do not benefit from contiguity while improving contiguity for those that do.

# Conclusion and Impacts

Contiguitas is a holistic solution that addresses the long-standing problem of memory contiguity in datacenters. Contiguitasis already having a major impact as it is in the process of being upstreamed to the Linux kernel [7], and Meta is actively working on deploying it in production. 

# References

[1] A. Hunter, C. Kennelly, P. Turner, D. Gove, T. Moseley, and P. Ranganathan, “Beyond malloc efficiency to fleet efficiency: a hugepage-aware memory allocator,” in 15th USENIX Symposium on Operating Systems Design and Implementation (OSDI), 2021.

[2] A. Basu, J. Gandhi, J. Chang, M. D. Hill, and M. M. Swift, “Efficient Virtual Memory for Big Memory Servers,” in 40th International Symposium on Computer Architecture (ISCA), 2013.

[3] D. Skarlatos, A. Kokolis, T. Xu, and J. Torrellas, “Elastic cuckoo page tables: Rethinking virtual memory translation for parallelism,” in 25th International Conference on Architectural Support for Programming Languages and Operating Systems (ASPLOS), 2020.

[4] B. Pham, V. Vaidyanathan, A. Jaleel, and A. Bhattacharjee, “CoLT: Coalesced Large-Reach TLBs,” in 45th International Symposium on Microarchitecture (MICRO), 2012.

[5] S. Gupta, A. Bhattacharyya, Y. Oh, A. Bhattacharjee, B. Falsafi, and M. Payer, “Rebooting virtual memory with midgard,” in 48th International Symposium on Computer Architecture (ISCA), 2021.

[6] J. Weiner, N. Agarwal, D. Schatzberg, L. Yang, H. Wang, B. Sanouillet, B. Sharma, T. Heo, M. Jain, C. Tang, and D. Skarlatos, “TMO: Transparent memory offloading in datacenters,” in 27th International Conference on Architectural Support for Program-ming Languages and Operating Systems (ASPLOS), 2022.

[7] “A Reliable Huge Page Allocator,” [https://lore.kernel.org/lkml/20230418191313.268131-1-hannes@cmpxchg.org/](https://lore.kernel.org/lkml/20230418191313.268131-1-hannes@cmpxchg.org/), 2023.