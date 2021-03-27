# Page Space Allocation

So allocation of space on a page happens in the `allocateSpace` function. There are three spots from which memory on a page can be allocated. They are these three, and will be attempted in this order:

```
0.)		From the freelist
1.)		From defragmentation (then proceeds to 2)
2.)		From the unallocated space
```

One more thing. There are two things commonly referred to here, which are the `gap` and the `top`. The `top` refers to the first byte of the cell content area. The gap refers to the first byte directly after the cell ptrs area (so the first byte of the cell content area).  

#### FreeList Allocation

So this is the first memory allocation attempted. This will be attempted if the `offsetFree` value of the page header is not null, and `gap+2<=top` is true (gap is more than 1 bytes above the top). It will attempt to find a free spot that is the correct size with the `pageFindSlot` function. It will search for a free page with the `pageFindSlot` function.

The `pageFindSlot` function will search for a free page. It will start with the first free list, which the offset is specified at offset `0x01` from the page geader. For the free list to be valid, each node in the list must be after the preceeding (checks occur for that). In addition to that, there is a check to ensure the free list does not extend past the end of the page.

#### Page Defragmentation

This will be attempted if the `gap+2+nByte>top` (new allocated space will go past the top). Defragmentation is the process of realligning the cells of the page, as to merge fragments together into usable blocks of freed memory. The memory will then be allocated from the `Unallocated Space` region, with the new freed up memory.

Page defragmentation happens with the `defragmentPage` function. It primarily just copies the entried to the bottom of the page to defragment the page. There are two defined methods for defragmentation, a general case and a optimized case for when there are two nodes in the free list. For defragmentation, there are several checks that happen. While it does, it has two copies of the memory page. One that acts as a source, and one that acts as a destination, each with their own index. The index for the destination is referred to as `cbrk`, and the index for the source is `pc`. The `pc` index must stay within the confines of the cell content area for the source, and `cbrk` cannot go above (be lesser than) the first cell offset. For the checks, it checks that the data that is being written do not extend past the end of the page.

#### Unallocated Space

So this will be the last resort. This just allocates the space from the unallocated region.