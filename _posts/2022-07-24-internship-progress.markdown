---
layout: post
title:  'Internship progress'
date:   2022-07-24 19:58:34 -0500
categories: Outreachy
tags:
- C
- Linux
- memblock
- Outreachy
---
## Add VERBOSE and MEMBLOCK_DEBUG Makefile options

The first project goal I accomplished was [adding verbose output and the Makefile flags VERBOSE and MEMBLOCK_DEBUG](https://lore.kernel.org/all/cover.1656907314.git.remckee0@gmail.com/) to Memblock simulator. These were items 1 and 2 from the TODO list:

1. Add verbose output (e.g., what is being tested and how many tests cases are
   passing)

2. Add flags to Makefile:
   + verbosity level
   + enable `memblock_dbg()` messages (i.e. pass "`-D CONFIG_DEBUG_MEMORY_INIT`"
     flag)

I sent my first version of this patchset to my mentors on June 11th. After a few rounds of internal reviews, I sent the patchset to the maintainers and mailing lists on June 22nd, and the final version was [accepted on July 4th](https://git.kernel.org/pub/scm/linux/kernel/git/rppt/memblock.git/log/?h=for-next&qt=range&q=fe833b4edc59...28e1a8f4b0ff). I received a lot of helpful feedback and suggestions, so the final version was very different from the initial version. When I wrote the timeline for my project, I only expected this task to take a week.

## Writing new tests

In the middle of week 5, since I was mostly done with that patchset, I started working on the next items on the TODO list:

<ol start="3">
<li>
<p>Add tests trying to <code class="language-plaintext highlighter-rouge">memblock_add()</code> or <code class="language-plaintext highlighter-rouge">memblock_reserve()</code> 129th region.
   This will trigger <code class="language-plaintext highlighter-rouge">memblock_double_array()</code>, make sure it succeeds.</p>

<p><em>Important:</em> These tests require valid memory ranges, use dummy physical
                memory block from common.c to implement them. It is also very
                likely that the current <code class="language-plaintext highlighter-rouge">MEM_SIZE</code> won't be enough for these
                test cases. Use <code class="language-plaintext highlighter-rouge">realloc</code> to adjust the size accordingly.</p>
</li>
<li>
<p>Add test cases using these functions (implement them for both directions):</p>
<ul>
<li><code class="language-plaintext highlighter-rouge">memblock_alloc_raw()</code></li>
<li><code class="language-plaintext highlighter-rouge">memblock_alloc_exact_nid_raw()</code></li>
<li><code class="language-plaintext highlighter-rouge">memblock_alloc_try_nid_raw()</code></li>
</ul>
</li>
</ol>

Item 3 didn't seem like it would be terribly complicated. I thought that it would take some time to figure out how to do it, but writing the code wouldn't be difficult once I figured that out. I decided that I was also going to try to write a test that would [trigger `memblock_double_array()` by using `memblock_remove()`](../09/paths-to-memblock_double_array.html) to repeatedly divide the memblock regions until there were 129 regions.

However, item 3 turned out to be really tricky. Part of the problem was that I was not keeping organized records of what I had already tried, so when I asked my mentors for help, they suggested things that I thought that I had already tried. It was difficult communicating this because I didn't remember what I did or what the result was when I tried it.

Item 4 seemed pretty straightforward now that I was familiar with the Memblock simulator code, but I had allocated two weeks for it on the timeline. Since `memblock_alloc_raw()` is very similar to `memblock_alloc()`, I thought that it would make sense to use similar tests. The main difference is that `memblock_alloc()` clears the allocated region, but `memblock_alloc_raw()` doesn't clear it. Then I noticed that the existing tests for `memblock_alloc()` don't check if the region was cleared like the tests for `memblock_alloc_try_nid()` and `memblock_alloc_from()` do. I decided to update the `memblock_alloc()` tests so that they check if allocated regions were cleared.

`memblock_alloc_exact_nid_raw()` and `memblock_alloc_try_nid_raw()` following a similar pattern. They are very similar to each other and to `memblock_alloc_try_nid()`, but `memblock_alloc_exact_nid_raw()` and `memblock_alloc_try_nid_raw()` don't clear allocated regions and `memblock_alloc_try_nid()` does. The difference between the `exact` and `try` versions is that `memblock_alloc_exact_nid_raw()` will not allocate memory if the requested NUMA node can not hold the requested memory, but both functions with `try` in their name will fallback to allocating in any node in the system.

Currently, all the tests for `memblock_alloc_try_nid()` set the nid parameter to `NUMA_NO_NODE`. So I would need to add new tests or modify existing tests to be able to test this specification. I suspected writing tests with NUMA support would be really complicated. I wasn't ready to dive into another complicated task yet since I was still working on item 3 on the TODO list.

So I wrote tests for `memblock_alloc_raw()`, `memblock_alloc_exact_nid_raw()`, and `memblock_alloc_try_nid_raw()` that were very similar to the existing tests. It seemed too easy, so I put those tests on the back burner (actually, a separate git branch) so I could add additional tests and/or add NUMA support later. I continued working on item 3 and got nowhere with it. By the middle of week 6, I decided to put work on item 3 on hold.

## Change build options to run-time options

I moved on to a task that one of my mentors suggested, changing some of the Makefile build options to runtime options. It seemed like it would be an easy task, and it was. I ended up changing verbose and movable node to runtime options. I submitted it as a single patch, and the first version I sent to the mailing lists was [accepted on July 20th](https://git.kernel.org/pub/scm/linux/kernel/git/rppt/memblock.git/commit/?h=for-next&id=06c8580aa23ddc9b1c37b9c7c5e2cc504e460207).

## Add additional tests for basic api and memblock_alloc()

During week 7, I started the next task: adding additional tests for the basic api [`memblock_add()`, `memblock_reserve()`, `memblock_remove()`, and `memblock_free()`] and for `memblock_alloc()`. I looked at whether there were any missing paths in the existing tests for these functions to come up with additional tests and also added a few tests that I thought would be interesting:

`memblock_add()` and `memblock_reserve()`:
- add/reserve a memory block in the gap between two existing memory
  blocks, and check that all three blocks are merged into one region
- try to add/reserve memblock regions that extend past `PHYS_ADDR_MAX`

`memblock_remove()` and `memblock_free()`:
- remove/free a region when it is the only available region
    - These tests ensure that the [first region is overwritten](https://elixir.bootlin.com/linux/v5.19-rc5/source/mm/memblock.c#L346) with a
      "dummy" region when the last remaining region of that type is
      removed or freed.
- remove/free a region that overlaps with two existing regions of the
  relevant type
- try to remove/free memblock regions that extend past `PHYS_ADDR_MAX`

memblock_alloc():
- try to allocate a region that is larger than the total size of available
  memory (`memblock.memory`)

Next, I decided to add labels to verbose output for generic alloc tests. Generic tests for `memblock_alloc*()` functions do not use separate functions for testing top-down and bottom-up allocation directions. Therefore, the function name that is displayed in the verbose testing output does not include the allocation direction.

Then, I added some tests for additional functions to the basic api:
- add tests for `memblock_*bottom_up` functions:
    + `memblock_set_bottom_up()`
    + `memblock_bottom_up()`
- add tests for `memblock_trim_memory`

These were fairly trivial, but I thought that they should be included in the test suite. I wanted to get them out of the way so I could move on to more interesting and important tasks.

## Second patchset

Then, I put together a large patchset with the following items:
- memblock tests: update tests to check if `memblock_alloc` zeroed memory
- memblock tests: add labels to verbose output for generic alloc tests
- memblock tests: add additional tests for basic api and `memblock_alloc`
- memblock tests: add `memblock_alloc_raw` tests
- memblock tests: add `memblock_alloc_exact_nid_raw` tests
- memblock tests: add `memblock_alloc_try_nid_raw` tests
- memblock tests: remove completed TODO item
- memblock tests: add tests for `memblock_*bottom_up` functions
- memblock tests: add tests for `memblock_trim_memory`

Preparing this patchset was a bit of a feat. The commits were on several different branches, and I ended up having to insert commits, reorder commits, and resolve merge conflicts several times. I originally had several of the commits on the branch that I started when I first wrote the `memblock_alloc_raw()`, `memblock_alloc_exact_nid_raw()`, and `memblock_alloc_try_nid_raw()` tests. Not realizing that this branch was based on an old version of the commits beneath it, I initially tried to put the other commits on top of this branch. When that didn't work, I figured out what the issue was. Then, I tried moving the commits on the tip of the old branch to the new branch, and fortunately that applied cleanly.

## NUMA support

Since I am waiting for this patchset to be reviewed, I have started working on what I had skipped/avoided earlier: introducing NUMA support. Surprisingly, this is not as difficult as I expected it to be. It is just difficult enough to be interesting. :) First, I created functions to set up a memory layout with NUMA nodes:

        /**
         * setup_numa_memblock_generic:
         * Set up a memory layout with multiple numa nodes in a previously allocated
         * dummy physical memory.
         * @nodes: an array containing the amount of memory in each node
         * @node_cnt: the size of @nodes
         *
         * The nids will be set to 0 through node_cnt - 1.
         */
        void setup_numa_memblock_generic(const phys_addr_t nodes[],
        				 int node_cnt)
        {
        	phys_addr_t base = (phys_addr_t)memory_block.base;
        	reset_memblock_regions();

        	for (int i = 0; i < node_cnt; i++) {
        		if (i > 0)
        			base += nodes[i - 1];
        		memblock_add_node(base, nodes[i], i, MEMBLOCK_HOTPLUG);
        	}
        }

        void setup_numa_memblock(void)
        {
        	setup_numa_memblock_generic(node_sizes, NUMA_NODES);
        }

Where `node_sizes` defines the amount of memory in each node. Currently, it is set up like this:

        #define NUMA_NODES				7

        static const phys_addr_t node_sizes[] = {
        	SZ_1K,
        	SZ_1K,
        	SZ_2K,
        	SZ_1K,
        	SZ_2K,
        	SZ_1K,
        	SZ_1K
        };

Then, I created some tests that call `setup_numa_memblock()` to set up their memory layout. In addition to tests for the `memblock_alloc_*_nid*()` functions, this also includes tests specified in the next TODO list item:

<ol start="5">
<li>
Add tests for <code class="language-plaintext highlighter-rouge">memblock_alloc_node()</code> to check if the correct NUMA node is set
for the new region
</li>
</ol>

Work on these tests is still in progress.

## Plans for the remainder of the internship

These are the tasks that I plan to do during the remainder of the internship, though not necessarily in this order:
- continue work on NUMA support
- update the patchset that is currently being reviewed after I receive feedback
- add tests for `memblock_alloc_range_nid()`
- revisit the tests trying to trigger `memblock_double_array()` so that I can at least carefully document what I've tried and the issues I've had

If I complete the above and still have time left, I will start these tasks:
- investigate whether it is feasible to write tests for `memblock_alloc_low()`
- write tests for the `memblock_phys*()` functions

During the last week, if not before, I will document any issues I've run into on the README and any remaining tasks on the TODO file.
