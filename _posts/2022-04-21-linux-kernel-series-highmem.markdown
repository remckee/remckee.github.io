---
layout: post
title:  'Linux kernel series: HIGHMEM'
date:   2022-04-21 21:21:18 -0500
categories: Outreachy
tags:
- C
- HIGHMEM
- Linux
---
This is the second post in the Linux kernel series. The [first part of the series](../../../../outreachy/2022/04/14/linux-kernel-series-memblock) was about `memblock`. This week, the topic is `HIGHMEM`.

## What is HIGHMEM?

The Linux kernel [virtualizes](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-intro.pdf) its memory. Virtual memory addresses, which can be used by programs, are mapped to physical memory addresses. When the size of the physical memory exceeds the (maximum) size of the virtual memory, it is not possible for the kernel to map every physical address to a virtual address. So the kernel uses temporary mappings when it wants to access part of the physical memory that is not mapped to a permanent virtual memory address. The part of physical memory that does not have a permanent mapping to a virtual address is called high memory (`HIGHMEM`).<sup><a href="#ref1">[1]</a></sup>

<figure class="wp-block-image size-large is-style-default"><img src="{{ site.baseurl }}/assets/2022/04/kernel-virtmem-map.png" alt="" />
<figcaption>Source: <a href="https://linux-kernel-labs.github.io/refs/heads/master/labs/memory_mapping.html#overview">https://linux-kernel-labs.github.io/refs/heads/master/labs/memory_mapping.html#overview</a></figcaption>
</figure>

`HIGHMEM` is often needed in 32 bit systems. A 32 bit address can represent at most 2<sup>32</sup> = 4,294,967,296 different addresses, which is approximately 4 GB (or [exactly 4 GiB](https://en.wikipedia.org/wiki/Byte#Multiple-byte_units)). But many systems have more than 4 GB of physical memory.

The available virtual memory space has to be divided between kernel space and user space. A common split is 3 : 1 user space : kernel space, which would mean 3 GB for user space and 1 GB for kernel space on a 32 bit system. The part of kernel space that has a direct mapping to a permanent physical memory is called `LOWMEM`. `LOWMEM` is contiguously mapped in physical memory.<sup><a href="#ref2">[2]</a></sup>

## Creating Temporary Maps

There are several different functions for creating temporary virtual maps.<sup><a href="#ref2">[2]</a></sup>

* **`vmap()`**: used to map multiple physical pages into a contiguous virtual address space for a long duration
* **`kmap()`**: used to map a single physical page for a short duration. Prone to deadlocks when nested, and not recommended in new code
* **`kmap_atomic()`**: used to map a single physical page for a very short duration. It performs well because it is restricted to the CPU that issued it, but the issuing task must stay on the same CPU until it finishes.
* **`kmap_local_page()`**: recently developed as a replacement for kmap(). It creates a mapping that is restricted to local use by a single thread of execution. This function is preferred over the other functions and should be used when possible.<sup><a href="#ref3">&#091;3]</a></sup>

## References

<ol>
<li id="ref1"><a href="https://www.kernel.org/doc/html/latest/vm/highmem.html">High Memory Handling — The Linux Kernel documentation</a></li>
<li id="ref2"><a href="https://linux-kernel-labs.github.io/refs/heads/master/labs/memory_mapping.html">Memory Mapping — The Linux Kernel documentation</a></li>
<li id="ref3"><a href="https://lore.kernel.org/lkml/?q=kmap_local_page">LKML Mailing List</a></li>
</ol>
