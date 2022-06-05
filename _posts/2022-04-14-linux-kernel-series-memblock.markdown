---
layout: post
title:  'Linux kernel series: memblock'
date:   2022-04-14 23:41:12 -0500
categories: Outreachy
tags:
- C
- Linux
- memblock
- Outreachy
---
This post is the first in a series in which I discuss the basics of a component or concept in the Linux kernel. The goal is to explain it in such a way that a beginner C or C++ programmer unfamiliar with the Linux kernel can comprehend it. There will be at least 2 posts in this series, including this one.

This week, the topic is `memblock`.

## What is memblock?

When programming in [kernel space](https://en.wikipedia.org/wiki/User_space_and_kernel_space), you cannot use the C standard library memory allocation functions available in user space, such as `malloc`, `free`, `calloc`, `realloc`, or `reallocarray`.<sup><a href="#ref1">&#091;1]</a></sup> There are kernel space equivalents, such as `kmalloc` or `kzalloc`, `kfree`, `kcalloc`, `krealloc_array`, `krealloc`,<sup><a href="#ref2">&#091;2]</a></sup> [and more](https://www.kernel.org/doc/html/latest/core-api/memory-allocation.html). However, these memory allocators are set up during the booting process. In the period of time before these allocators are available, the system still needs to allocate memory. During that period of time, a specialized allocator called `memblock` is used to allocate memory.<sup><a href="#ref3">&#091;3]</a></sup>

`memblock` views the system memory as a collection of contiguous regions. `memblock` is made up of memblock types, which are themselves made up of multiple memory regions. There are several structures and functions for working with `memblock`.<sup><a href="#ref4">[4]</a></sup>

## Structures

### struct memblock

The `struct memblock`<sup><a href="#ref5">[5]</a></sup> wraps the metadata for the **memory** and **reserved** memblock types, along with `current_limit`, the physical address of the current allocation limit, and `bottom_up`, the allocation direction. The allocation limit limits allocations to memory that is currently accessible and can be set with `memblock_set_current_limit(phys_addr_t limit)`. The `memblock` structure is statically initialized at build time.

    struct memblock {
    	bool bottom_up;
    	phys_addr_t current_limit;
    	struct memblock_type memory;
    	struct memblock_type reserved;
    };

### struct memblock_type

An array of several memory regions of the same type can be represented with `struct memblock_type`.<sup><a href="#ref5">[5]</a></sup> This struct defines an array of memory regions `regions` of type `name` with `cnt` number of regions. The total size of the array is `max`, and the size of all the regions is `total_size`.

    struct memblock_type {
    	unsigned long cnt;
    	unsigned long max;
    	phys_addr_t total_size;
    	struct memblock_region *regions;
    	char *name;
    };

The memory region types are **memory**, **reserved**, and **physmem**.

* **memory** is the physical memory available to the kernel.
* **reserved** refers to the regions that were actually allocated.
* **physmem** is the total physical memory available during boot regardless of whether the kernel can access it. It is not available on some architectures.

### struct memblock_region

The `struct memblock_region`<sup><a href="#ref5">&#091;5]</a></sup> represents a memory region with base address `base`, size `size`, memory region attributes `flags`, and NUMA node id `nid`. NUMA is a system with [non-uniform memory access](https://www.kernel.org/doc/html/latest/vm/numa.html).


    struct memblock_region {
    	phys_addr_t base;
    	phys_addr_t size;
    	enum memblock_flags flags;
    #ifdef CONFIG_NUMA
    	int nid;
    #endif
    };

The memory region attributes are defined in `enum memblock_flags`.<sup><a href="#ref5">&#091;5]</a></sup>

    enum memblock_flags {
    	MEMBLOCK_NONE		= 0x0,	/* No special request */
    	MEMBLOCK_HOTPLUG	= 0x1,	/* hotpluggable region */
    	MEMBLOCK_MIRROR		= 0x2,	/* mirrored region */
    	MEMBLOCK_NOMAP		= 0x4,	/* don't add to kernel direct mapping */
    	MEMBLOCK_DRIVER_MANAGED = 0x8,	/* always detected via a driver */
    };

## Functions

`memblock` is initialized in `setup_arch()` and taken down in `mem_init()`.

`memblock` has many useful APIs. These APIs include functions for adding and removing memory regions, and allocating and freeing memory, among [many other functions](https://www.kernel.org/doc/html/latest/core-api/boot-time-mm.html#c.for_each_physmem_range).

### Adding and removing

These functions add or remove a memblock region with the given base address and size. They return an int indicating whether the operation succeeded. The `memblock_add_node()` version adds the region within a NUMA node and includes parameters for the node id and memblock flags.

    memblock_add(base, size)
    memblock_add_node(base, size, nid, flags)
    memblock_remove(base, size)

### Allocating

These functions allocate a memory block and return the physical address of the memory.

    memblock_phys_alloc(size, align)
    memblock_phys_alloc_range(size, align, start, end)
    memblock_phys_alloc_try_nid(size, align, nid)

These functions allocate a memory block and return the virtual address of the memory.

    memblock_alloc(size, align)
    memblock_alloc_from(size, align, min_addr)
    memblock_alloc_low(size, align)
    memblock_alloc_node(size, align, nid)

### Freeing

The functions `memblock_free()` and `memblock_phys_free()` free a boot memory block that was previously allocated; it is not released to the buddy allocator (which handles allocation after boot process completes). The `*ptr` parameter in `memblock_free()` is the starting address of the boot memory block. The functions `memblock_free_all()` and `memblock_free_late()` free pages to the buddy allocator.

    memblock_free(*ptr, size)
    memblock_phys_free(base, size)
    memblock_free_all(void)
    memblock_free_late(base, size)

## References

<ol>
<li id="ref1"><a href="https://man7.org/linux/man-pages/man3/free.3.html">malloc(3) — Linux manual page</a></li>
<li id="ref2"><a href="https://www.kernel.org/doc/html/latest/core-api/mm-api.html#c.kmalloc">Memory Management APIs — The Linux Kernel documentation</a></li>
<li id="ref3"><a href="https://www.kernel.org/doc/html/latest/core-api/boot-time-mm.html#">Boot time memory management — The Linux Kernel documentation</a></li>
<li id="ref4"><a href="https://www.kernel.org/doc/html/latest/core-api/boot-time-mm.html#functions-and-structures">Boot time memory management — The Linux Kernel documentation: Functions and structures</a></li>
<li id="ref5">Code snippet from <a href="https://github.com/torvalds/linux/blob/master/include/linux/memblock.h">linux/memblock.h</a></li>
</ol>
