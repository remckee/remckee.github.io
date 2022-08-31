---
layout: post
title:  'Internship progress: part 2'
date:   2022-08-31 15:15:00 -0500
categories: Outreachy
tags:
- C
- Linux
- memblock
- Outreachy
---

This post describes project progress since the [Internship progress](../../07/24/internship-progress.html) post in more detail than the [Internship reflection](../../08/30/internship-reflection.html) post.

## Add and update zeroed memory checks

Since the [Internship progress](../../07/24/internship-progress.html) post, I updated the following patch:

### Update tests to check if memblock_alloc zeroed memory

This patch adds an assert in `memblock_alloc()` tests where allocation is expected to occur. In the previous version of this patch, the assert only checked the first byte of the memory area.

For example:

        	char *b;
        ...
        	allocated_ptr = memblock_alloc(size, SMP_CACHE_BYTES);
        	b = (char *)allocated_ptr;
        ...
        	ASSERT_EQ(*b, 0);

The current version of the patch checks whether the entire chunk of allocated memory is cleared using the following assert macro:

        /**
         * ASSERT_MEM_EQ():
         * Check that the first @_size bytes of @_seen are all equal to @_expected.
         * If false, print failed test message (if running with --verbose) and then
         * assert.
         */
        #define ASSERT_MEM_EQ(_seen, _expected, _size) do { \
        	for (int _i = 0; _i < (_size); _i++) { \
        		ASSERT_EQ(((char *)_seen)[_i], (_expected)); \
        	} \
        } while (0)

 To make this more robust, `setup_memblock()` and `dummy_physical_memory_init()` fill the entire `MEM_SIZE` simulated physical memory with nonzero values by calling `fill_memblock()`:

        static inline void fill_memblock(void)
        {
        	memset(memory_block.base, 1, MEM_SIZE);
        }

Thus, the memory region is expected to contain only nonzero values at the start of the test.

Example of updated version:

        	allocated_ptr = memblock_alloc(size, SMP_CACHE_BYTES);
        ...
        	ASSERT_MEM_EQ(allocated_ptr, 0, size);

### Update zeroed memory check for memblock_alloc_* tests

I then added the following patch:

- memblock tests: update zeroed memory check for memblock_alloc_* tests

This patch updates the `memblock_alloc_try_nid()` and `memblock_alloc_from()` tests so that they check that the entire allocated memory region is cleared using the same method described above for `memblock_alloc()`.

## Update alloc_api and alloc_nid_api tests to test raw versions

### Update alloc_api to test memblock_alloc_raw

As discussed in the [Internship progress](../../07/24/internship-progress.html) post, the proposed tests for `memblock_alloc_raw` were very similar to the tests for `memblock_alloc`. The main difference is that `memblock_alloc()` clears the allocated region, but `memblock_alloc_raw()` doesn't clear it. One of my mentors suggested adding testing flags to cut down on repeated code.

        enum test_flags {
        	/* No special request. */
        	TEST_F_NONE = 0x0,
        	/* Perform raw allocations (no zeroing of memory). */
        	TEST_F_RAW = 0x1,
        };

The tests can then check the currently set `test_flags` to determine which memblock function to run. This is done by replacing the `memblock_alloc()` call originally in the test with a call to `run_memblock_alloc()`:

        static inline void *run_memblock_alloc(phys_addr_t size, phys_addr_t align)
        {
        	if (alloc_test_flags & TEST_F_RAW)
        		return memblock_alloc_raw(size, align);
        	return memblock_alloc(size, align);
        }

I replaced the assert checking whether the memory is cleared:

        	ASSERT_MEM_EQ(allocated_ptr, 0, size);

with

        	assert_mem_content(allocated_ptr, size, alloc_test_flags);

When the tests run `memblock_alloc()`, `assert_mem_content()` ensures that the entire memory region is zero. When the tests run `memblock_alloc_raw()`, `assert_mem_content()` ensures that the entire memory region is nonzero. The content of the memory region is initialized to nonzero by `fill_memblock()`. We expect it to remain unchanged if running `memblock_alloc_raw()`.

        static inline void assert_mem_content(void *mem, int size, int flags)
        {
        	if (flags & TEST_F_RAW)
        		ASSERT_MEM_NE(mem, 0, size);
        	else
        		ASSERT_MEM_EQ(mem, 0, size);
        }


        /**
         * ASSERT_MEM_NE():
         * Check that none of the first @_size bytes of @_seen are equal to @_expected.
         * If false, print failed test message (if running with --verbose) and then
         * assert.
         */
        #define ASSERT_MEM_NE(_seen, _expected, _size) do { \
        	for (int _i = 0; _i < (_size); _i++) { \
        		ASSERT_NE(((char *)_seen)[_i], (_expected)); \
        	} \
        } while (0)

### Update alloc_nid_api to test memblock_alloc_try_nid_raw

I also updated the `memblock_alloc_try_nid()` tests to run tests for `memblock_alloc_try_nid_raw()`. The `memblock_alloc_try_nid()` call originally in the tests was replaced with a call to `run_memblock_alloc_try_nid()`:

        static inline void *run_memblock_alloc_try_nid(phys_addr_t size,
        					       phys_addr_t align,
        					       phys_addr_t min_addr,
        					       phys_addr_t max_addr, int nid)
        {
        	if (alloc_nid_test_flags & TEST_F_RAW)
        		return memblock_alloc_try_nid_raw(size, align, min_addr,
        						  max_addr, nid);
        	return memblock_alloc_try_nid(size, align, min_addr, max_addr, nid);
        }

## Add NUMA support

### Add NUMA tests for memblock_alloc_try_nid*

I created a set of tests that use a simulated physical memory that is set up with multiple NUMA nodes. Additionally, most of these tests set `nid != NUMA_NO_NODE`. See [https://lore.kernel.org/all/cover.1661578435.git.remckee0@gmail.com](https://lore.kernel.org/all/cover.1661578435.git.remckee0@gmail.com) for details about the tests.

`setup_numa_memblock_generic()` and `setup_numa_memblock()`, are used to set up a simulated physical memory with multiple NUMA nodes. These functions use a previously allocated dummy physical memory. They can be used in place of `setup_memblock()` in tests that need to simulate a NUMA system.

Updated version of `setup_numa_memblock_generic()` and `setup_numa_memblock()`:

        /**
         * setup_numa_memblock_generic:
         * Set up a memory layout with multiple NUMA nodes in a previously allocated
         * dummy physical memory.
         * @nodes: an array containing the amount of memory in each node
         * @node_cnt: the size of @nodes
         * @factor: a factor that will be used to scale the memory in each node
         *
         * The nids will be set to 0 through node_cnt - 1.
         */
        void setup_numa_memblock_generic(const phys_addr_t nodes[],
        				 int node_cnt, int factor)
        {
        	phys_addr_t base;
        	int flags;

        	reset_memblock_regions();
        	base = (phys_addr_t)memory_block.base;
        	flags = (movable_node_is_enabled()) ? MEMBLOCK_NONE : MEMBLOCK_HOTPLUG;

        	for (int i = 0; i < node_cnt; i++) {
        		phys_addr_t size = factor * nodes[i];

        		memblock_add_node(base, size, i, flags);
        		base += size;
        	}
        	fill_memblock();
        }

        void setup_numa_memblock(void)
        {
        	setup_numa_memblock_generic(node_sizes, NUMA_NODES, MEM_FACTOR);
        }

I added these tests to the current tests (range tests) for `memblock_alloc_try_nid()` and `memblock_alloc_try_nid_raw()`.

### Add tests for memblock_alloc_exact_nid_raw

Next, I added tests for `memblock_alloc_exact_nid_raw()`. The range tests are very similar to the range tests for `memblock_alloc_try_nid_raw()`. The NUMA tests have the same set up as the corresponding test for `memblock_alloc_try_nid_raw()`, but several of the `memblock_alloc_exact_nid_raw()` tests fail to allocate memory in setups where the `memblock_alloc_try_nid_raw()` test would allocate memory.

### Add tests for memblock_alloc_node and memblock_set_node

The tests that I wrote for `memblock_alloc_node()` use a simulated physical memory set up with multiple NUMA nodes. The tests are run for both allocation directions.

The tests for `memblock_set_node()` attempt to set the nid for the regions within a given range. The test scenarios include tests where the region counter is expected to increase, decrease, or not change.

### Add tests for memblock_alloc_range_nid

The tests for `memblock_alloc_range_nid()` are in progress.
