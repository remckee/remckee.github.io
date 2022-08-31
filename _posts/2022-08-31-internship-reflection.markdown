---
layout: post
title:  'Internship reflection'
date:   2022-08-31 14:15:00 -0500
categories: Outreachy
tags:
- C
- Linux
- memblock
- Outreachy
---

This post gives an overall summary of my internship. In my next post, I will discuss my project progress since the [Internship progress](../../07/24/internship-progress.html) post in more detail.

## API improvements

During my internship, I have made several improvements to memblock simulator, the memblock testing suite.

The first improvement was the [addition of verbose testing output](https://git.kernel.org/pub/scm/linux/kernel/git/rppt/memblock.git/log/?h=for-next&qt=range&q=76586c00e74d...28e1a8f4b0ff) that can be optionally enabled by the user. The verbose output allows the user to verify details of the tests without having to look through the code. This includes which tests are being run, which memblock functions are being tested, and whether both allocation directions are tested for `memblock_alloc*()` functions. It has been very useful to me as a developer to check my work; I hope other users and developers will find it useful as well.

I also added a [build option to enable `memblock_dbg()` messages](https://git.kernel.org/pub/scm/linux/kernel/git/rppt/memblock.git/commit/?h=for-next&id=c55b31a124a68171e5915b1036ca42d8a683eca2). These are debug messages built into several functions in memblock that include information such as the name of the function and the size and start and end addresses of the memblock region.

Next, I [changed the verbose and movable node build options to run-time options](https://git.kernel.org/pub/scm/linux/kernel/git/rppt/memblock.git/commit/?h=for-next&id=06c8580aa23ddc9b1c37b9c7c5e2cc504e460207) that are passed on the command line. This improved usability because now the user can enable and disable verbose output and simulated movable node without rebuilding. I also [added a help run-time option](https://git.kernel.org/pub/scm/linux/kernel/git/rppt/memblock.git/commit/?h=for-next&id=c0f1bc4e91c52be73ae1a5e6fd53371f5a7f0333) and a help menu to describe these command line options.

Later, I [added labels to the verbose output for generic tests](https://git.kernel.org/pub/scm/linux/kernel/git/rppt/memblock.git/commit/?h=for-next&id=fb2e97fe853ff515df473d4acec6707816e05d87) for `memblock_alloc*()` functions. The labels specify which allocation direction is set when a given test is running. Generic tests are tests that do not use separate functions for testing top-down and bottom-up allocation directions because the allocation direction does not change the outcome of those tests. The labels add information that was not previously included in the verbose output, because the function name that is displayed in the verbose testing output for generic functions does not include the allocation direction.

## Test improvements

I improved the tests for:
- [memblock_alloc()](https://git.kernel.org/pub/scm/linux/kernel/git/rppt/memblock.git/commit/?h=for-next&id=ac76d803c4f6c2a32c9c7436d14467e099fd2bfa)
- [memblock_alloc_try_nid() and memblock_alloc_from()](https://git.kernel.org/pub/scm/linux/kernel/git/rppt/memblock.git/commit/?h=for-next&id=25b9defb5bc4aee8beb51ded07838e12745426f9)

These `memblock_alloc*()` tests are expected to clear the memory region that they allocate if the allocation is successful. I improved the tests so that they check that the entire memory region was cleared. Previously, the tests for `memblock_alloc()` did not include this check, and the tests for `memblock_alloc_try_nid()` and `memblock_alloc_from()` only checked the first byte. To make this robust, `setup_memblock()` and `dummy_physical_memory_init()` fill the entire `MEM_SIZE` simulated physical memory with nonzero values by calling `fill_memblock()` so that the memory region contains only nonzero values at the beginning of the test.

## New tests

I [added more tests](https://git.kernel.org/pub/scm/linux/kernel/git/rppt/memblock.git/commit/?h=for-next&id=21a233f68afe55aafa8b79705c97f7a1d37be3e1) for:
- memblock_add()
- memblock_reserve()
- memblock_remove()
- memblock_free()
- memblock_alloc()

I introduced test coverage for:
- [memblock_alloc_raw()](https://git.kernel.org/pub/scm/linux/kernel/git/rppt/memblock.git/commit/?h=for-next&id=deee033e0f8ea66a9f4acfc1eb069fdef3013bec)
- [memblock_alloc_try_nid_raw()](https://git.kernel.org/pub/scm/linux/kernel/git/rppt/memblock.git/commit/?h=for-next&id=ae544fd62c14265dc663a65b3f9c6c5a6134098a)
- [memblock_set_bottom_up() and memblock_bottom_up()](https://git.kernel.org/pub/scm/linux/kernel/git/rppt/memblock.git/commit/?h=for-next&id=a541c6d428f775efcfe25236062c96b59e31b57a)
- [memblock_trim_memory()](https://git.kernel.org/pub/scm/linux/kernel/git/rppt/memblock.git/commit/?h=for-next&id=dcd45ad2ad784c35bfba8ae93c285574bc2a8a1e)

To add coverage for `memblock_alloc_raw()`, the alloc_api was updated so that it runs through all the existing tests twice: once for `memblock_alloc()` and once for `memblock_alloc_raw()`. Similarly, the alloc_nid_api was updated to run through its tests twice: once for `memblock_alloc_try_nid()` and once for `memblock_alloc_try_nid_raw()`. I made these `memblock_alloc_*raw()` tests more robust by having them check that the entire memory region was **not** cleared. As mentioned above, the content of the memory region is initialized to nonzero, so we expect it to remain unchanged for the `memblock_alloc_raw()` tests.

### In review: NUMA tests

Additionally, I worked on adding NUMA support to the testing suite. These patches are currently [in review](https://lore.kernel.org/all/cover.1661578435.git.remckee0@gmail.com).

These patches include additional tests for:
- memblock_alloc_try_nid()
- memblock_alloc_try_nid_raw()

These tests use a simulated physical memory that is [set up with multiple NUMA nodes](https://lore.kernel.org/all/25b727327d365a435cc1c56c10fb96cd71b6898a.1661578435.git.remckee0@gmail.com). Additionally, most of these tests set nid != NUMA_NO_NODE. These tests are run twice, once for `memblock_alloc_try_nid()` and once for `memblock_alloc_try_nid_raw()`, so that both functions are tested with the same set of tests. When the tests `run memblock_alloc_try_nid()`, they test that the entire memory region is zero. When the tests run `memblock_alloc_try_nid_raw()`, they test that the entire memory region is nonzero.

## Future directions

I have some in progress tests that I am planning to finish within the next week:
- memblock_alloc_exact_nid_raw()
- memblock_alloc_node()
- memblock_set_node()

These are described briefly at the end of this post.

While writing this post, I thought of a possible future improvement to the testing suite:
- addition of information about the tests to the help message

## Personal comments

This internship has been a great opportunity for me. I have learned so much. One of the best parts of the internship was getting thorough feedback on my patches from my mentors as well as the Linux kernel community. This has helped me become a better programmer. I feel more confident contributing to open source projects. I even made some great connections with other Outreachy interns, despite my tendency to be an introvert.
