# Deep Dive: f_mapping and the Page Cache

## What Is the "mapping"? Why Does Every Inode Have One?

---

# Chapter 1: The Problem — Where Are My File's Pages?

You have a database file that's 40KB (10 pages). Some of those pages
are currently cached in RAM (the page cache). Some are only on disk.

```
Your database file: 10 pages

  Page 0: on disk AND in RAM (cached)
  Page 1: on disk AND in RAM (cached)
  Page 2: on disk only (not cached)
  Page 3: on disk AND in RAM (cached, DIRTY — modified!)
  Page 4: on disk only
  Page 5: on disk only
  Page 6: on disk AND in RAM (cached)
  Page 7: on disk only
  Page 8: on disk only
  Page 9: on disk AND in RAM (cached, DIRTY)
```

Now when you call `write(fd, data, 5)` to write at file offset 8192
(which is page 2 of the file), the kernel needs to answer:

```
"Is page 2 of THIS FILE already in memory?"
  → If YES: write directly to the cached page (fast!)
  → If NO: allocate a new page in memory, maybe read from disk first,
           then write to it
```

But HOW does the kernel find "page 2 of file X" in memory?
The kernel might have millions of cached pages from thousands of files.
It needs a fast lookup: **(file, page_number) → cached page in RAM**.

**THAT is what the `mapping` (address_space) is.** It's a data structure
that answers: "For THIS specific file, which of its pages are currently
in memory, and where?"


---

# Chapter 2: The Analogy — A Library Catalog

Imagine a huge library (RAM) with millions of books (pages) on shelves.
You need to find a specific page from a specific file.

**Without a catalog:** Scan every shelf in the entire library looking for
"page 2 of database.db". With millions of pages from thousands of files,
this takes forever.

**With a catalog (one per file):** Each file has its own index card in a
catalog drawer:

```
Catalog drawer for "database.db":
  ┌───────────────────────────────────────┐
  │ Page 0 → shelf A, position 47         │
  │ Page 1 → shelf C, position 12         │
  │ Page 2 → (not in library)             │
  │ Page 3 → shelf B, position 89 (DIRTY) │
  │ Page 6 → shelf D, position 3          │
  │ Page 9 → shelf A, position 102 (DIRTY)│
  └───────────────────────────────────────┘

Catalog drawer for "logfile.txt":
  ┌───────────────────────────────────────┐
  │ Page 0 → shelf E, position 55         │
  │ Page 1 → shelf E, position 56         │
  └───────────────────────────────────────┘

Catalog drawer for "config.yaml":
  ┌───────────────────────────────────────┐
  │ Page 0 → shelf F, position 1          │
  └───────────────────────────────────────┘
```

Each file has its own catalog drawer. To find "page 2 of database.db",
you open database.db's drawer and look up page 2. Instant.

**That catalog drawer IS the `address_space` (the mapping).**
Each file (inode) has exactly ONE address_space.
The address_space maps page numbers → physical memory pages.


---

# Chapter 3: The Data Structures — How It's Actually Built

## 3.1: The inode owns the address_space

Every file on disk is represented by an `inode`. The inode has a field
called `i_mapping` that points to the address_space for this file:

```
struct inode (for database.db, inode number 4521):

  ┌──────────────────────────────────────────────────────────┐
  │ i_ino = 4521           (inode number)                    │
  │ i_size = 40960         (file size: 40KB = 10 pages)      │
  │ i_mode = 0100644       (regular file, rw-r--r--)         │
  │                                                          │
  │ i_mapping ──────────────────────┐                        │
  │                                 │                        │
  │ i_data (struct address_space) ◄─┘                        │
  │   ┌──────────────────────────────────────────────────┐   │
  │   │ host = &this_inode  (points back to the inode)   │   │
  │   │ nrpages = 5         (5 pages currently cached)   │   │
  │   │ i_pages = xarray    (THE LOOKUP STRUCTURE)       │   │
  │   │ a_ops = &ext4_aops  (page cache operations)      │   │
  │   └──────────────────────────────────────────────────┘   │
  │                                                          │
  └──────────────────────────────────────────────────────────┘
```

**Key detail:** The address_space (`i_data`) is EMBEDDED inside the inode.
It's not a separate allocation — it lives right inside the inode struct.
And `i_mapping` is a pointer TO this embedded struct. So:

```c
inode->i_mapping == &inode->i_data

// They point to the same thing!
// i_mapping is a POINTER to i_data.
// i_data is the actual address_space struct.
```

**Why have both i_mapping and i_data?**

Normally `i_mapping = &i_data` (points to itself). But for block devices,
multiple inodes can share the SAME address_space. For example, `/dev/sda`
and `/dev/sda1` might share cached pages. In that case, `i_mapping` on
one inode can point to ANOTHER inode's `i_data`. For regular files
(like your database), they always point to their own.

## 3.2: The address_space in detail

```c
struct address_space {
    struct inode       *host;      // "I cache pages for THIS inode"
    struct xarray      i_pages;    // ★ THE MAP: page_index → struct page
    atomic_t           nrpages;    // how many pages are cached right now
    
    struct address_space_operations *a_ops;  // how to read/write pages
    
    // Writeback tracking:
    unsigned long      flags;       // AS_HAS_DIRTY (has dirty pages?)
    spinlock_t         i_lock;      // protects modifications
    struct list_head   i_dirty_list; // list of dirty pages
};
```

The most important field is `i_pages` — an **XArray** (a radix tree)
that maps page indices to actual memory pages.


## 3.3: The XArray — The Actual Lookup Structure

The XArray is a tree that maps an integer key (page index) to a pointer
(struct page). Think of it like a HashMap but optimized for integer keys
and sparse populations:

```
XArray for database.db's address_space:

            ┌─────────────┐
            │ XArray root  │
            └──────┬──────┘
                   │
        ┌──────────┴──────────┐
        ▼                     ▼
  ┌───────────┐         ┌───────────┐
  │ Entries    │         │ Entries    │
  │ 0-63      │         │ 64-127    │
  └───────────┘         └───────────┘
   │   │             │
   ▼   ▼             ▼
  [0] [1]  [3]  [6]  [9]     ← only CACHED pages have entries
   │   │    │    │    │
   ▼   ▼    ▼    ▼    ▼
  pg   pg   pg   pg   pg     ← struct page (actual memory)

Lookup: i_pages[2] → NULL (page 2 is NOT cached)
Lookup: i_pages[3] → struct page at 0xFFFF888001230000 (CACHED!)
Lookup: i_pages[7] → NULL (page 7 is NOT cached)
```

**Why a tree and not an array?** Because files can be HUGE (terabytes)
but only a tiny fraction of their pages are cached. A flat array for a
1TB file would need 256 million entries. The XArray only allocates nodes
for indices that actually have cached pages.

**Lookup cost:** O(log N) where N is the file size in pages, but the tree
is very shallow (4-6 levels even for huge files). In practice, the lookup
is effectively O(1) because the tree nodes themselves are cached in CPU
cache lines.


---

# Chapter 4: What struct page Is — The Physical Memory Page

When you look up a page index in the XArray and find an entry, you get a
`struct page`. This represents ONE 4KB chunk of physical RAM:

```c
struct page {
    unsigned long flags;       // PG_dirty, PG_uptodate, PG_locked, etc.
    struct address_space *mapping;  // ★ back-pointer to the address_space!
    pgoff_t index;             // ★ my page index within the file
    atomic_t _refcount;        // reference count
    atomic_t _mapcount;        // how many page tables map this page
    struct list_head lru;      // position in the LRU list (for eviction)
    void *virtual;             // kernel virtual address of this page
};
```

**Notice the back-pointers!** The struct page knows:
- `mapping`: which file (address_space) I belong to
- `index`: which page number within that file I am

So from a struct page, you can go BACK to the file:
```
struct page → page->mapping → address_space → address_space->host → inode
```

And from an inode, you can find any cached page:
```
inode → inode->i_mapping → address_space → i_pages[index] → struct page
```

**The navigation works in both directions.**


## 4.1: The page flags — what state is this page in?

```
PG_locked    → Someone is reading/writing this page right now. Wait.
PG_uptodate  → The page content matches what's on disk (valid data).
PG_dirty     → The page has been MODIFIED in memory but NOT written to disk.
PG_referenced → The page was accessed recently (used by LRU for eviction).
PG_active    → The page is on the "active" LRU list (hot, don't evict).
PG_writeback → The page is currently being written to disk.
PG_reclaim   → The page is selected for eviction.
```

A page's lifecycle through flags:

```
Just allocated:     flags = 0 (nothing set)
After disk read:    flags = PG_uptodate (data is valid)
After modification: flags = PG_uptodate | PG_dirty (modified but valid)
During disk write:  flags = PG_uptodate | PG_dirty | PG_writeback
After disk write:   flags = PG_uptodate (clean again — matches disk)
```


---

# Chapter 5: How f_mapping Connects Everything

Now let's go back to the original line:

```c
f->f_mapping = inode->i_mapping
```

When you `open()` a file, the kernel creates a `struct file` and copies
the inode's `i_mapping` pointer into the file's `f_mapping` field:

```
struct file (for your open fd=3):

  f_mapping = inode->i_mapping ─────┐
  f_inode = inode 4521              │
  f_pos = 8192                      │
  f_op = ext4_file_operations       │
                                    │
                                    ▼
  ┌─ address_space for inode 4521 ──────────────────────┐
  │                                                      │
  │  host = inode 4521                                   │
  │  nrpages = 5                                         │
  │                                                      │
  │  i_pages (XArray):                                   │
  │    [0] → struct page (physical addr 0x50000)         │
  │    [1] → struct page (physical addr 0x82000)         │
  │    [2] → (empty — not cached)                        │
  │    [3] → struct page (physical addr 0xA1000) DIRTY   │
  │    [6] → struct page (physical addr 0x33000)         │
  │    [9] → struct page (physical addr 0xC5000) DIRTY   │
  │                                                      │
  │  a_ops = ext4_aops                                   │
  │    .readpage = ext4_readpage                          │
  │    .writepage = ext4_writepage                        │
  │    .write_begin = ext4_write_begin                    │
  │    .write_end = ext4_write_end                        │
  │                                                      │
  └──────────────────────────────────────────────────────┘
```

**Why is f_mapping a separate field?** Why not just use `f->f_inode->i_mapping`
every time?

Two reasons:
1. **Speed**: One fewer pointer dereference. Instead of going
   `file → inode → mapping`, the kernel goes `file → mapping` directly.
   This is called on every read and write, millions of times per second.
   One pointer dereference saved = measurable performance gain.

2. **Abstraction**: Some special files (like block devices) can have
   `f_mapping` point somewhere different than `f_inode->i_mapping`.
   Having a separate field allows this flexibility.


---

# Chapter 6: Walking Through a Write — Using the Mapping

Let's trace what happens when you write 5 bytes at file offset 8192,
focusing specifically on how the mapping is used:

```
write(fd=3, "Hello", 5) at file offset 8192

Step 1: WHICH PAGE?
  file_offset = 8192
  page_index = 8192 / 4096 = 2
  offset_within_page = 8192 % 4096 = 0
  
  "I need page 2 of this file."

Step 2: IS PAGE 2 IN THE CACHE?
  mapping = f->f_mapping    (get the address_space)
  page = xa_load(&mapping->i_pages, 2)   (look up index 2 in the XArray)
  
  ┌─ XArray lookup for index 2 ─────────────────────┐
  │                                                   │
  │  i_pages[0] → page (exists)                       │
  │  i_pages[1] → page (exists)                       │
  │  i_pages[2] → NULL ← NOT CACHED!                  │
  │  i_pages[3] → page (exists)                       │
  │                                                   │
  └───────────────────────────────────────────────────┘
  
  Result: NULL → page 2 is NOT in the cache. CACHE MISS.

Step 3: ALLOCATE A NEW PAGE IN MEMORY
  new_page = alloc_page(GFP_KERNEL)
  
  The kernel asks the physical memory allocator for a free 4KB page.
  → Gets physical page at address 0xD7000
  → Kernel virtual address: 0xFFFF888000D7000
  
  new_page->mapping = f->f_mapping   (back-pointer: "I belong to this file")
  new_page->index = 2                (back-pointer: "I am page 2")

Step 4: DO WE NEED TO READ THE OLD DATA?
  We're writing 5 bytes at offset 0 within this 4096-byte page.
  That means bytes 5-4095 of this page are NOT being written.
  
  Do we care about the old bytes 5-4095?
  → If the file is 10000 bytes long, page 2 covers bytes 8192-12287.
    Bytes 8197-10000 contain file data we must preserve!
  → So YES: we must read page 2 from disk first.
  
  The kernel calls:
    mapping->a_ops->readpage(file, new_page)
    → ext4_readpage()
    → ext4 looks up the disk block for page 2 (using the extent tree)
    → Submits a disk read: block 50432 → new_page
    → Disk read completes: new_page now has the old data
    → new_page->flags |= PG_uptodate ("I have valid data")

Step 5: ADD THE PAGE TO THE CACHE
  xa_store(&mapping->i_pages, 2, new_page)
  mapping->nrpages++    (5 → 6)
  
  ┌─ XArray AFTER adding page 2 ───────────────────┐
  │                                                   │
  │  i_pages[0] → page (exists)                       │
  │  i_pages[1] → page (exists)                       │
  │  i_pages[2] → new_page ← NOW CACHED!              │
  │  i_pages[3] → page (exists)                       │
  │  i_pages[6] → page (exists)                       │
  │  i_pages[9] → page (exists)                       │
  │                                                   │
  │  nrpages = 6                                       │
  └───────────────────────────────────────────────────┘

Step 6: WRITE THE DATA
  Copy "Hello" into the page at offset 0:
    memcpy(page_address(new_page) + 0, "Hello", 5)
  
  The page now has:
    bytes 0-4: "Hello" (new data)
    bytes 5-4095: (old data from disk, preserved)

Step 7: MARK THE PAGE DIRTY
  new_page->flags |= PG_dirty
  
  "This page has been modified. It's different from what's on disk.
   It MUST be written to disk before it can be evicted."
```


---

# Chapter 7: Walking Through a Read — Using the Mapping

Now let's read the same page back:

```
read(fd=3, buf, 10) at file offset 8192

Step 1: WHICH PAGE?
  page_index = 8192 / 4096 = 2

Step 2: IS PAGE 2 IN THE CACHE?
  mapping = f->f_mapping
  page = xa_load(&mapping->i_pages, 2)
  
  Result: page at 0xFFFF888000D7000 → CACHE HIT!
  
  (We put it there during the write in Chapter 6.)

Step 3: COPY DATA TO USER BUFFER
  copy_to_user(user_buf, page_address(page) + 0, 10)
  
  Copies 10 bytes from the cached page to your buffer.
  NO DISK I/O. Entirely from RAM. ~200 nanoseconds.

DONE! The mapping made this instant — one XArray lookup, one memcpy.
```

Now read a page that's NOT cached:

```
read(fd=3, buf, 4096) at file offset 16384

Step 1: WHICH PAGE?
  page_index = 16384 / 4096 = 4

Step 2: IS PAGE 4 IN THE CACHE?
  mapping = f->f_mapping
  page = xa_load(&mapping->i_pages, 4)
  
  Result: NULL → NOT CACHED. CACHE MISS.

Step 3: ALLOCATE AND READ FROM DISK
  new_page = alloc_page(GFP_KERNEL)
  mapping->a_ops->readpage(file, new_page)
  → ext4 reads from disk → ~100,000 nanoseconds (slow!)
  
  xa_store(&mapping->i_pages, 4, new_page)  (add to cache)
  mapping->nrpages++

Step 4: COPY TO USER BUFFER
  copy_to_user(user_buf, page_address(new_page), 4096)

DONE. But it was 500x slower than the cache hit case!
```


---

# Chapter 8: Multiple File Descriptors, ONE Mapping

Here's something important: if two processes open the same file,
they get DIFFERENT struct files but the SAME mapping:

```
Process A: fd 3 → struct file (A) → f_mapping ──┐
                   f_pos = 0                      │
                                                  │
Process B: fd 5 → struct file (B) → f_mapping ──┤
                   f_pos = 5000                   │
                                                  │
                                                  ▼
                                    ┌──────────────────────────┐
                                    │ address_space             │
                                    │ (ONE instance per inode)  │
                                    │                          │
                                    │ i_pages:                 │
                                    │   [0] → page             │
                                    │   [1] → page             │
                                    │   [2] → page             │
                                    │   [3] → page (DIRTY)     │
                                    │                          │
                                    │ host = inode 4521        │
                                    └──────────────────────────┘
```

**Both processes share the SAME cached pages.** If Process A writes to
page 2, Process B immediately sees the change (because they're looking
at the same physical memory page through the same mapping).

This is how:
- Two processes can share a file efficiently (only one copy in RAM)
- mmap works (the mapped memory IS the page cache pages)
- Cache consistency is maintained (one truth, one source)

```
Process A writes "Hello" to page 2:
  → Modifies the page in the shared mapping
  → Page is marked dirty

Process B reads page 2:
  → Looks up page 2 in the SAME mapping
  → Gets the SAME physical page
  → Sees "Hello" immediately!
  → No disk I/O needed (it's already in the cache)
```


---

# Chapter 9: The Relationship Chain — Complete Picture

```
                    ONE PER OPEN() CALL
                    ┌────────────────────────────────────┐
  fd=3 ───────────► │ struct file                         │
                    │   f_pos = 8192                      │
  fd=7 ───────────► │   f_flags = O_RDWR                  │  (different fd,
  (dup'd or         │   f_mapping ────────────┐           │   same file)
   forked)          │   f_inode ──────┐       │           │
                    └────────────────│───────│───────────┘
                                     │       │
                    ONE PER FILE     │       │
                    ┌────────────────▼──┐    │
                    │ struct inode       │    │
                    │   i_ino = 4521    │    │
                    │   i_size = 40960  │    │
                    │   i_mapping ──────│────┤  (same pointer!)
                    │                   │    │
                    │   i_data: ◄───────│────┘
                    │   ┌───────────────▼──────────────────┐
                    │   │ struct address_space              │
                    │   │ (THE MAPPING / PAGE CACHE)        │
                    │   │                                   │
                    │   │ ONE PER INODE                     │
                    │   │                                   │
                    │   │ host = inode 4521 (back-pointer)  │
                    │   │ nrpages = 5                       │
                    │   │                                   │
                    │   │ i_pages (XArray):                 │
                    │   │   [0] → struct page ──┐           │
                    │   │   [1] → struct page   │           │
                    │   │   [3] → struct page   │           │
                    │   │   [6] → struct page   │           │
                    │   │   [9] → struct page   │           │
                    │   │                       │           │
                    │   │ a_ops:                │           │
                    │   │   .readpage           │           │
                    │   │   .writepage          │           │
                    │   │   .write_begin        │           │
                    │   │   .write_end          │           │
                    │   └───────────────────────│───────────┘
                    └───────────────────────────│────────────┘
                                                │
                    ONE PER CACHED PAGE          │
                    ┌───────────────────────────▼───────────┐
                    │ struct page                             │
                    │                                        │
                    │ mapping = address_space (back-pointer)  │
                    │ index = 0 (which page of the file)      │
                    │ flags = PG_uptodate | PG_dirty           │
                    │ _refcount = 2 (two users)               │
                    │                                        │
                    │ THE ACTUAL 4096 BYTES OF DATA           │
                    │ ARE AT THE PHYSICAL ADDRESS              │
                    │ THAT THIS struct page REPRESENTS        │
                    │                                        │
                    └────────────────────────────────────────┘
```

```
MULTIPLICITY:

  Many fd's ──────► One struct file (or many if separate opens)
  Many struct files ► One inode
  One inode ──────► One address_space (mapping)
  One address_space ► Many struct pages (the cached pages)
  One struct page ──► One address_space (back-pointer)
```


---

# Chapter 10: Connection to Your Storage Engine

The kernel's address_space is doing the EXACT same thing as your buffer pool:

```
KERNEL                              YOUR STORAGE ENGINE
────────────────────                ─────────────────────────────
address_space                   ←→  BufferPool
  (one per inode/file)              (one per database)

i_pages (XArray)                ←→  page_table (HashMap)
  maps: page_index → struct page    maps: page_id → frame_index

struct page                     ←→  Frame
  holds 4KB of cached file data     holds 4KB of cached page data

PG_dirty flag                   ←→  frame.dirty flag
  "modified, needs writeback"       "modified, needs flushing"

nrpages                         ←→  frames.len() (occupied count)

a_ops->readpage()               ←→  file.read_at() in fetch_page()
  "load this page from disk"        "load this page from disk"

a_ops->writepage()              ←→  file.write_at() in flush_page()
  "write this dirty page to disk"   "write this dirty page to disk"

LRU active/inactive lists       ←→  LRU-K replacer
  kernel's eviction policy          your eviction policy

page->mapping (back-pointer)    ←→  frame.page_id
  "which file does this page        "which database page is in
   belong to?"                       this frame?"

page->index (page number)       ←→  (same — the page_id IS the index)
```

**The key difference:** The kernel's mapping uses a generic LRU and gives
you no control over eviction or write ordering. Your buffer pool gives you
full control — which is why databases build their own cache on top of
(or instead of) the kernel's page cache.


---

# Chapter 11: Why This Matters for fsync()

When you call `fsync(fd)`, the kernel needs to find ALL dirty pages for
your file and flush them to disk. The mapping makes this efficient:

```
fsync(fd=3):

Step 1: Get the mapping
  mapping = f->f_mapping

Step 2: Find all dirty pages
  Walk the XArray looking for pages with PG_dirty set:
  
  i_pages[0] → page (clean) → skip
  i_pages[1] → page (clean) → skip
  i_pages[2] → page (DIRTY) → add to flush list
  i_pages[3] → page (DIRTY) → add to flush list
  i_pages[6] → page (clean) → skip
  i_pages[9] → page (DIRTY) → add to flush list

Step 3: Flush the dirty pages
  For each dirty page:
    mapping->a_ops->writepage(page)
    → ext4 translates page index to disk block
    → submits write I/O to the block layer
    → waits for completion
    → clears PG_dirty flag

Step 4: Send FLUSH CACHE to the drive
  (ensures drive writes its volatile cache to permanent media)

Step 5: Return to user space
  "Your data is now on disk."
```

**Without the mapping, fsync would have to scan ALL pages in the entire
system looking for dirty pages belonging to your file.** With the mapping,
it's a targeted walk of just YOUR file's XArray — only visiting pages that
actually exist in the cache.


---

# Summary

```
f->f_mapping = inode->i_mapping means:

  "This file descriptor's page cache is the SAME page cache
   as the inode's page cache."

The mapping (address_space) is:
  → A per-file data structure
  → Contains an XArray mapping: page_number → physical_memory_page
  → Tells the kernel which pages of THIS file are currently in RAM
  → Shared by ALL file descriptors pointing to the same inode
  → Used by read() to find cached data (cache hit = fast)
  → Used by write() to find/create the target page
  → Used by fsync() to find all dirty pages that need flushing
  → The kernel's equivalent of your buffer pool's page_table HashMap
```
