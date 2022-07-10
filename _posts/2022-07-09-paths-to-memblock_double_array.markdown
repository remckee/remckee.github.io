---
layout: post
title:  "Linux kernel series: paths to memblock_double_array"
date:   2022-07-09 17:38:23 -0500
categories: Outreachy
tags:
- C
- Linux
- memblock
- Outreachy
---
This post discusses the internal details of some `memblock` functions&mdash;specifically, functions that may trigger `memblock_double_array()`. The goal is to explain it in such a way that a beginner C or C++ programmer unfamiliar with the Linux kernel can comprehend it. I discussed the [basics of `memblock`](../../../../outreachy/2022/04/14/linux-kernel-series-memblock) previously.

Looking at [`memblock.c`](https://elixir.bootlin.com/linux/latest/source/mm/memblock.c), we see that `memblock_double_array()` is called by [`memblock_add_range()`](https://elixir.bootlin.com/linux/latest/source/mm/memblock.c#L649) and [`memblock_isolate_range()`](https://elixir.bootlin.com/linux/latest/source/mm/memblock.c#L735). These are both internal functions with no prototype in [`memblock.h`](https://elixir.bootlin.com/linux/latest/source/include/linux/memblock.h). By _internal function_, I mean a function that is called by a function in `memblock.h` or `memblock.c`.

`memblock_add_range()` makes two passes through a portion of its code.<sup><a href="#ref1">&#091;1]</a></sup> In the first pass, it determines `nr_new`, the number of new regions that need to be added. It then calls `memblock_double_array()` if the current number of `memblock_type *type` regions plus `nr_new` exceeds the current size of that type's `regions` array.

`memblock_isolate_range()` splits memblock regions that overlap with a given range at the boundaries of the range.<sup><a href="#ref2">&#091;2]</a></sup> It calls `memblock_double_array()` if the current number of `memblock_type *type` regions plus 2 exceeds the current size of that type's `regions` array, since at most two additional regions will be created by the splits.

We can trace `memblock_add_range()` and `memblock_isolate_range()` upwards until we find the external functions (those with prototypes in `memblock.h`) that can trigger `memblock_double_array()`.

## memblock_add_range()

`memblock_add_range()` is called by:

* `memblock_add_node()`: external function<sup><a href="#ref3">&#091;3]</a></sup>
* `memblock_add()`: external function
* `memblock_reserve()`: external and internal function
* `memblock_physmem_add()`: external function

By _external and internal function_, I mean a function that has a prototype in `memblock.h` and is called in `memblock.c` or `memblock.h`.

### memblock_reserve()

`memblock_reserve()` is called by:

* `memblock_double_array()`
* `memblock_alloc_range_nid()`: external and internal function

#### memblock_alloc_range_nid()

`memblock_alloc_range_nid()` is called by:

* `memblock_phys_alloc_range()`: external and internal function called by:
    * `memblock_phys_alloc()`: external function
* `memblock_phys_alloc_try_nid()`: external function
* `memblock_alloc_internal()`: internal function called by:
    * `memblock_alloc_exact_nid_raw()`: external function
    * `memblock_alloc_try_nid_raw()`: external and internal function called by:
        * `memblock_alloc_raw()`: external function
    * `memblock_alloc_try_nid()`: external and internal function called by:
        * `memblock_alloc()`: external function
        * `memblock_alloc_from()`: external function
        * `memblock_alloc_low()`: external function
        * `memblock_alloc_node()`: external function

## memblock_isolate_range()

This one is a bit more complicated. `memblock_isolate_range()` is called by:

* `memblock_remove_range()`: internal function
* `memblock_setclr_flag()`: internal function
* `memblock_set_node()`: external function
* `memblock_cap_memory_range()`: external and internal function

### memblock_remove_range()

`memblock_remove_range()` is called by:

* `memblock_remove()`: external function
* `memblock_phys_free()`: external function and internal function
* `memblock_enforce_memory_limit()`: external function
* `memblock_cap_memory_range()`

#### memblock_phys_free()

`memblock_phys_free()` is called by:

* `memblock_free()`: external function and internal function called by:
    * `memblock_double_array()`
* `free_memmap()`: internal function called by:
    * `free_unused_memmap()`: internal function called by:
        * `memblock_free_all()`: external function

### memblock_setclr_flag()

Each `memblock_region` has a set of [`memblock_flags`](https://elixir.bootlin.com/linux/latest/source/include/linux/memblock.h#L44). `memblock_setclr_flag()` uses `memblock_isolate_range()` to isolate a specific region so that it can set or clear flags for that region. The regions are re-merged at the end of the function, so there is no net increase in the number of regions.

`memblock_setclr_flag()` is called by:

* `memblock_mark_hotplug()`: external function
* `memblock_clear_hotplug()`: external function and internal function called by:
    * `free_low_memory_core_early()`: internal function called by `memblock_free_all()`
* `memblock_mark_mirror()`: external function
* `memblock_mark_nomap()`: external function
* `memblock_clear_nomap()`: external function

### memblock_cap_memory_range()

`memblock_cap_memory_range()` is called by:

* `memblock_mem_limit_remove_map()`: external function

## External functions

Thus, most of the external functions in memblock can trigger `memblock_double_array()`. This includes:

* `memblock_add_node()`
* `memblock_add()`
* `memblock_reserve()`
* `memblock_physmem_add()`
* `memblock_alloc_range_nid()`
* `memblock_phys_alloc_range()`
* `memblock_phys_alloc()`
* `memblock_phys_alloc_try_nid()`
* `memblock_alloc_exact_nid_raw()`
* `memblock_alloc_try_nid_raw()`
* `memblock_alloc_raw()`
* `memblock_alloc_try_nid()`
* `memblock_alloc()`
* `memblock_alloc_from()`
* `memblock_alloc_low()`
* `memblock_alloc_node()`
* `memblock_set_node()`
* `memblock_cap_memory_range()`
* `memblock_remove()`
* `memblock_phys_free()`
* `memblock_enforce_memory_limit()`
* `memblock_free()`
* `memblock_free_all()`
* `memblock_mark_hotplug()`
* `memblock_clear_hotplug()`
* `memblock_mark_mirror()`
* `memblock_mark_nomap()`
* `memblock_clear_nomap()`
* `memblock_mem_limit_remove_map()`

## References

<ol>
<li id="ref1"><a href="https://elixir.bootlin.com/linux/latest/source/mm/memblock.c#L596">memblock.c: memblock_add_range(): repeat</a></li>
<li id="ref2"><a href="https://elixir.bootlin.com/linux/latest/source/mm/memblock.c#L705">memblock.c: memblock_isolate_range()</a></li>
<li id="ref3"><a href="https://elixir.bootlin.com/linux/latest/source/include/linux/memblock.h">memblock.h</a></li>
</ol>
