# How Linux finds physical address through virtual memory

## Why

Hardware can map virtual address to physical address or use physical address directly.

The virtual address to physical address mapping is stored in memory.

When virtual address is enabled, hardware will help us do the virtual address to physical address mapping.

Sometimes, kernel needs do same thing.

For example when kernel allocates new memory for process kernel needs walk through and setup the mapping.

So kernel needs to know how page table works.

Now let's see how the hardware handle the virtual addresses.

## x86 4-level paging

X86 supports different kinds of pages, they are similar. So let's consider 4 levels paging and 4KB page only.

![Screen Shot 2021-08-11 at 9 14 57 PM](https://user-images.githubusercontent.com/3775525/129035231-13bdac79-ebf8-4b1f-8e51-e988ddfa3eee.png)

As the image above, register `CR3` is point to start address of global(L3) directory page, and virtual address' 47 ~ 39 bits will used for global(L3) directory page(L3)'s offset.

Then the content will point to next level directory's start address, then we can use 38 ~ 30 bits of virtual address for offset of upper(L2) directory page.

Then middle(L1) directory page and finally we arrive at page table(L0).

### Linux `follow_page` method

Linux `follow_page` is used for doing same thing.

Main follow as before:

``` c#
// in mm/gup.c
follow_page(vma, address, flags)
pgd = pgd_offset(mm, address);
follow_p4d_mask(vma, address, pgd, flags, ctx);
  p4d = p4d_offset(pgdp, address); // level 2
  follow_pud_mask(vma, address, p4d, flags, ctx);
    pud = pud_offset(p4dp, address);
    follow_pmd_mask(vma, address, pud, flags, ctx);
      pmd = pmd_offset(pudp, address); // level 1
      follow_page_pte(vma, address, pmd, flags, &ctx->pgmap);
        ptep = pte_offset_map(mm, pmd, address, &ptl); // level 0
        pte = *ptep;
        page = vm_normal_page(vma, address, pte);
```

`pgd_offset`'s defination is:

 ``` c
 #define PGDIR_SHIFT 39
 #define PTRS_PER_PGD 512

#define pgd_offset(mm, address) pgd_offset_pgd((mm)->pgd, (address))

 static inline pgd_t *pgd_offset_pgd(pgd_t *pgd, unsigned long address)
 {
    return (pgd + pgd_index(address));
 };

 #define pgd_index(a)  (((a) >> PGDIR_SHIFT) & (PTRS_PER_PGD - 1))

 ```

`#define pgd_index(a)  (((a) >> PGDIR_SHIFT) & (PTRS_PER_PGD - 1))` will shift right 39 bits then mark out 9 bits which will give us 47 ~ 39 bits of a virtual address.

The result is the offset of the page global directory's offset.

We add it to `pgd` by `pgd + pgd_index(address)` and the result is the next level directory's start address.


`p4d_offset` is used for 5-level paging, code as below:

``` c
static inline p4d_t *p4d_offset(pgd_t *pgd, unsigned long address)
{
	if (!pgtable_l5_enabled())
		return (p4d_t *)pgd;
	return (p4d_t *)pgd_page_vaddr(*pgd) + p4d_index(address);
}
```

`pgtable_l5_enabled()` will return `false` we use level 4 paging, so `p4d_offset` will just return `pgd` back.

Then `pud_offset` for the upper directory, then `pmd_offset` for the middle directory and finally reached the last level by calling `pte_offset`.

The calling is follow_page: pgd_offset -> p4d_offset -> pud_offset -> pmd_offset -> pte_offset.

## Paging
There are different kinds of paging in x84: 32-bit paging, PAE paging, 4-level paging and 5-level paging.

Basically they are same, but just uses different bits for finding pages.

More detail can be fond in `Intel® 64 and IA-32 Architectures Software Developer’s Manual` or `Understanding Linux Kernel`.

So I will not explain all of them.

There are many interesting things about page table or addressing such as copy on write and mmap.

I will explain some of them in the following articles.
