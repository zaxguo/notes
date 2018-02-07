# Page Allocation



### Page operations on the write path of pagecache 

A simple (buffered) ``write()`` in EXT2 file system uses the generic kernel interface which uses page cache to buffer all the content to be written.

```
generic_perform_write() -> 
		ext2_write_begin() ->
			block_write_begin(.., ext2_get_block) ->
				grab_cache_page_write_begin() ->
					pagecache_get_page() --..-->
						__aloc_pages_nodemask() 
				...
				__block_write_begin(..., ext2_get_block)		
```

``__block_write_begin()`` will call ``ext2_get_block()`` to map physical disk blocks to scattered buffers mapped to that page.

``grab_cache_page_write_begin()`` will take the address space of the *FILE* and an index (converted from the offset within the file) and returns a page struct allocated on the per-addressing space page cache tree.

``pagecache_get_page()`` is a the worker routine called by ``grab_cache_page_write_begin()`` which is a generic function to grab a page cache. 
It first search through the page cache tree to find if there's already an valid entry (i.e., a page struct that's already associated with a physical page), if there is one, it is returned, otherwise  it will further call ``__page_cache_alloc(gfp_mask)`` to allocate a page, which is simply a wrapper of ``alloc_pages(gfp, 0)``.

Subsequent calls to page allocation converges to NUMA page allocation policy, with a fixed node number *0* to indicate non-NUMA memory allocation. 
And this won't be discussed here. 

After the above functions are successfully executed, the preparation for a write is complete (i.e., find page to hold user content, allocate disk blocks to these content).
At this point, however, the write is not done yet since the content has not been copied from user space.

``iov_iter_copy_from_user_atomic(struct page *page, struct iov_iter *i, unsigned long offset, size_t bytes)`` does the job.
Taking the allocated page, it copies the content from user space to a kernel page.
The main function to convert a *page struct* to a usable data page to hold data is ``kmap_atomic(page)``:

```c
void *kmap_atomic(struct page *page)
{	... 
	if (!PageHighMem(page))
		return page_address(page); /* Used if not HIGHMEM */

#ifdef CONFIG_DEBUG_HIGHMEM
	/*
	 * There is no cache coherency issue when non VIVT, so force the
	 * dedicated kmap usage for better debugging purposes in that case.
	 */
	if (!cache_is_vivt())
		kmap = NULL;
	else
#endif
		kmap = kmap_high_get(page);
	if (kmap)
		return kmap;
	/* idx is a CPU-related offset so that kmap can be atomic */
	vaddr = __fix_to_virt(idx);
	/* setup the PTE */
	set_fixmap_pte(idx, mk_pte(page, kmap_prot));
	return (void *)vaddr;
}
```

And the actual physical pages can be retrieved ``page_address()``, note in ARM, when ``WANT_PAGE_VIRTUAL`` is not set, this function only works for lowmem which is always mapped.
Otherwise it'll map some highmem from FIXMAP.

```c
**
 * page_address - get the mapped virtual address of a page
 * @page: &struct page to get the virtual address of
 *
 * Returns the page's virtual address.
 */
void *page_address(const struct page *page)
{
	unsigned long flags;
	void *ret;
	struct page_address_slot *pas;

	if (!PageHighMem(page))
		return lowmem_page_address(page);

	pas = page_slot(page);
	ret = NULL;
	spin_lock_irqsave(&pas->lock, flags);
	if (!list_empty(&pas->lh)) {
		struct page_address_map *pam;

		list_for_each_entry(pam, &pas->lh, list) {
			if (pam->page == page) {
				ret = pam->virtual;
				goto done;
			}
		}
	}
done:
	spin_unlock_irqrestore(&pas->lock, flags);
	return ret;
}

```

The phyiscal page associated with a page struct is stored in ``page_address_map->virtual`` and is set by ``set_page_address(struct page *page, void *virtual)`` if the page is in highmem (with ``WANT_PAGE_VIRTUAL`` set).
This function takes a page struct and a virtual address, assigns the virtual address to the corresponding ``page_address_map`` entry in the hash table, adds it to the ``page_address_slot`` list.
If the page is not in highmem (i.e., with a highmem flag set), it is linearly mapped as are other kernel space addresses.

Since the pagecache subsystem directly uses the facility provided by page allocator, and it does not specify which kind of page it wants (i.e., through GFP flags), the page used to hold user data can come from **both highmem and lowmem**.
After wrting is done, the system will unmap the virtual address of the data page.
``page->private`` is not the same as the physical data page neither. 

This (also from experiments), implies that when writing to some data page, ``page struct`` does not keep a record of which phyiscal page it is associated with (?).
It only cares about the **accounting** of these pages (i.e., through page struct).
Whenever we need to modify the physical page associated with a page struct, we should call ``kmap_atomic()`` to map that physical data page so that we may write/read.


### Page operations on the read path of pagecache 
```c
Sys_read() -->
	vfs_read() -->
		__vfs_read() -->
			generic_file_read_iter() -->
				pagecache_get_page() -->
```











