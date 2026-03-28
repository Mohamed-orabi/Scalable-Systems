# struct file vs struct page — Why Both Exist

## The Complete Guide to the Two Most Confused Kernel Objects

---

# Chapter 1: The One-Sentence Answer

```
struct file = "Someone opened a file. Track their session."
struct page = "A chunk of file data is sitting in RAM. Track the memory."
```

A `struct file` is about the ACT of using a file.
A `struct page` is about the DATA of a file living in RAM.

They solve completely different problems. Let me show you why.


---

# Chapter 2: The Analogy That Makes It Click

Imagine a library.

**The BOOK on the shelf** = the file on disk (the inode).
There's only ONE book, no matter how many people are reading it.

**A READING SLIP** = `struct file`.
When you check out a book, the librarian creates a slip:
- Your name (which process)
- Which book (which inode)
- Your current page (f_pos — your bookmark)
- Your permissions (can you write in the margins? read-only?)

If 5 people check out the same book, there are 5 reading slips
but still ONE book. Each person has their own bookmark position.

**A PHOTOCOPIED PAGE on the desk** = `struct page`.
When someone actually reads chapter 3, the library photocopies
chapter 3 and puts it on a desk (RAM) so people don't have to
go to the storage room (disk) every time.

The photocopy exists because someone NEEDED that chapter.
Multiple readers can look at the same photocopy simultaneously.
When nobody needs it anymore and the desk is full, the photocopy
gets thrown away (evicted from the page cache).

```
Reading slip (struct file):
  → "WHO is using the book, and WHERE are they in it?"
  → Created when you open(). Destroyed when you close().
  → One per open() call. Personal to you.

Photocopied page (struct page):
  → "A piece of the book's CONTENT is sitting on the desk."
  → Created when someone first reads/writes that region.
  → Shared by ALL readers. Exists as long as RAM allows.
```


---

# Chapter 3: struct file — What It Tracks

A `struct file` is created every time a process calls `open()`.
It represents ONE open session with ONE file.

```c
struct file {
    loff_t          f_pos;      // ★ "Where am I in the file right now?"
                                //    Your personal bookmark. Changes with
                                //    every read() and write().
    
    unsigned int    f_flags;    // ★ "How did I open this file?"
                                //    O_RDONLY? O_WRONLY? O_RDWR? O_APPEND?
                                //    Set at open() time, usually never changes.
    
    fmode_t         f_mode;     // ★ "What operations are allowed?"
                                //    FMODE_READ? FMODE_WRITE?
                                //    Derived from f_flags.
    
    struct inode     *f_inode;  // ★ "Which file am I opened on?"
                                //    Points to the one true inode.
    
    struct address_space *f_mapping; // ★ "Where is this file's page cache?"
                                     //    Points to the inode's page cache.
    
    struct file_operations *f_op;   // ★ "Which filesystem handles I/O?"
                                     //    ext4? XFS? NFS? Function pointers.
    
    const struct cred *f_cred;  // "Who opened this file?"
                                //    uid, gid of the process that called open().
    
    struct mutex     f_pos_lock; // Lock for f_pos (thread safety).
    
    atomic_long_t    f_count;   // Reference count. When it hits 0,
                                // the struct file is freed.
};
```

### Key properties of struct file:

```
CREATED:    when a process calls open()
DESTROYED:  when the last close()/dup reference is released
SCOPE:      one per open() call (personal to the opener)
CONTAINS:   session state (position, flags, permissions)
DOES NOT:   contain any file DATA (not a single byte of content!)
LIVES IN:   kernel heap memory (dynamically allocated)
```


---

# Chapter 4: struct page — What It Tracks

A `struct page` represents ONE 4KB chunk of physical RAM that currently
holds data from a file (or from anonymous memory like your stack/heap).

```c
struct page {
    unsigned long    flags;     // ★ "What state is this memory in?"
                                //    PG_dirty: modified, not on disk yet
                                //    PG_uptodate: contains valid data
                                //    PG_locked: someone is using it right now
                                //    PG_writeback: being written to disk
    
    struct address_space *mapping; // ★ "Which file does this data belong to?"
                                   //    Points back to the inode's page cache.
                                   //    NULL if this page isn't file-backed.
    
    pgoff_t          index;     // ★ "Which 4KB chunk of the file am I?"
                                //    If index=2, I hold bytes 8192-12287.
    
    atomic_t         _refcount; // How many users are referencing this page.
                                // When it hits 0, the page can be freed.
    
    struct list_head lru;       // Position in the LRU eviction list.
                                // The kernel uses this to decide which
                                // pages to evict when RAM runs low.
};
```

### Key properties of struct page:

```
CREATED:    when the kernel allocates a physical RAM frame
DESTROYED:  when the RAM frame is freed (returned to the free page pool)
SCOPE:      shared by ALL processes that access this file region
CONTAINS:   metadata ABOUT 4KB of RAM (not the bytes themselves!)
            The actual 4096 bytes are at the physical address
            that this struct page represents.
LIVES IN:   a global array (mem_map[]) — one struct page per physical frame
```

**Important subtlety**: The struct page is NOT the 4096 bytes of data.
It's a small (~64 byte) metadata structure that DESCRIBES a 4096-byte
region of physical RAM. The actual data is at a physical address derived
from the struct page's position in the mem_map array.

```
Physical RAM layout:

  Frame 0:  physical bytes 0x00000 - 0x00FFF  → described by mem_map[0]
  Frame 1:  physical bytes 0x01000 - 0x01FFF  → described by mem_map[1]
  Frame 2:  physical bytes 0x02000 - 0x02FFF  → described by mem_map[2]
  ...
  Frame N:  physical bytes N*4096 - (N+1)*4096-1 → described by mem_map[N]

  To get the actual data from a struct page:
    void *data = page_address(page);
    // This converts the struct page to its kernel virtual address.
    // Now you can read/write the 4096 bytes at that address.
```


---

# Chapter 5: Side-by-Side Comparison

```
                    struct file                 struct page
                    ────────────                ────────────
What it is          An open file session        A cached chunk of RAM

Created when        open() is called            Data is first read from disk
                                                or first written to

Destroyed when      close() (last ref)          RAM is needed (eviction)
                                                or file is deleted

How many            One per open() call         One per 4KB of cached data
                    5 opens = 5 struct files     40KB file fully cached = 10 pages

Belongs to          One process (or shared       The INODE — shared by all
                    via dup/fork)                processes accessing the file

Contains data?      NO! Only metadata about      Points to 4KB of actual data
                    the open session             in physical RAM

Has a position?     YES — f_pos (bookmark)       YES — index (which 4KB chunk)
                    "I'm at byte 8192 in         "I hold bytes 8192-12287
                     the file"                    of the file"

Mutable by user?    YES — read/write/seek        INDIRECTLY — write() modifies
                    change f_pos                  the page's data bytes

Can be dirty?       NO — struct file is never    YES — PG_dirty means the data
                    "dirty"                       was modified but not on disk

Evictable?          NO — exists until close()    YES — kernel can evict to
                                                 free RAM (flush if dirty)

Per-process?        YES — each process gets      NO — shared by all processes
                    its own struct file            accessing the same file

Analogous to        A browser TAB open to        A cached copy of a PORTION
                    a web page                    of that web page in memory
```


---

# Chapter 6: Why Both Are Needed — A Concrete Scenario

Two processes (A and B) both work with database.db (40KB, 10 pages).
Process A opened it for reading at the beginning.
Process B opened it for writing and is at byte 20000.

```
PROCESS A:
  fd 3 → struct file (A) {
      f_pos = 0              ← A is at the beginning
      f_flags = O_RDONLY     ← A can only read
      f_inode = inode 4521
      f_mapping = inode 4521's address_space
  }

PROCESS B:
  fd 7 → struct file (B) {
      f_pos = 20000          ← B is at byte 20000
      f_flags = O_RDWR       ← B can read and write
      f_inode = inode 4521   ← SAME inode!
      f_mapping = inode 4521's address_space  ← SAME mapping!
  }
```

Now process A reads 4096 bytes (page 0 of the file):

```
Process A: read(fd=3, buf, 4096)

Step 1: Kernel reads A's struct file
  f_pos = 0 → page index = 0/4096 = 0
  f_mapping → inode 4521's address_space

Step 2: Kernel checks the address_space (page cache)
  xa_load(&mapping->i_pages, 0) → NULL (not cached!)

Step 3: CACHE MISS — allocate a struct page
  new_page = alloc_page()
  new_page->mapping = address_space   (back-pointer)
  new_page->index = 0                 (page 0 of the file)
  new_page->flags = 0                 (not yet valid)

Step 4: Read from disk into the struct page's memory
  ext4_readpage(file, new_page)
  → disk read → 4096 bytes loaded into new_page's RAM
  new_page->flags |= PG_uptodate     (data is now valid)

Step 5: Add to the address_space
  xa_store(&mapping->i_pages, 0, new_page)

Step 6: Copy to user buffer
  copy_to_user(buf, page_address(new_page), 4096)

Step 7: Update A's struct file
  A's f_pos = 0 + 4096 = 4096
```

Now process B reads the SAME page 0:

```
Process B: pread(fd=7, buf, 4096, 0)  // read page 0

Step 1: Kernel reads B's struct file
  f_mapping → SAME address_space as A (same inode!)

Step 2: Kernel checks the address_space
  xa_load(&mapping->i_pages, 0) → FOUND! The struct page from A's read!

Step 3: CACHE HIT — no disk I/O needed!
  The struct page allocated during A's read is shared.
  B reads the exact same 4KB of physical RAM.

Step 4: Copy to B's user buffer
  copy_to_user(buf, page_address(page), 4096)

  FAST! No disk read. The struct page was already there.
```

Now process B writes to page 0:

```
Process B: pwrite(fd=7, "Hello", 5, 0)  // write at byte 0

Step 1: Kernel reads B's struct file
  f_flags = O_RDWR → writing is allowed ✓
  f_mapping → address_space

Step 2: Find page 0 in the address_space
  xa_load(&mapping->i_pages, 0) → FOUND! Same struct page.

Step 3: Write into the struct page's memory
  memcpy(page_address(page) + 0, "Hello", 5)

Step 4: Mark the struct page DIRTY
  page->flags |= PG_dirty
  "This page has been modified. Must be written to disk eventually."

Now process A reads page 0 again:

Step 1: xa_load(&mapping->i_pages, 0) → same struct page
Step 2: copy_to_user(buf, page_address(page), 4096)
  → A sees "Hello" + the rest of the original data!
  → A sees B's write IMMEDIATELY, because they share the struct page.
```

**Summary of what exists at this point:**

```
struct file objects: 2
  → A's struct file (f_pos=4096, O_RDONLY)
  → B's struct file (f_pos=20000, O_RDWR)
  → Different positions, different permissions, SAME file

struct page objects: 1 (for page 0)
  → Shared by both A and B
  → flags = PG_uptodate | PG_dirty (B wrote to it)
  → mapping = inode 4521's address_space
  → index = 0

inode: 1
  → inode 4521 (database.db)

address_space: 1
  → Belongs to inode 4521
  → i_pages[0] → the shared struct page
```


---

# Chapter 7: Multiplicity — How Many of Each?

```
SCENARIO: 3 processes open the same 40KB file (10 pages).
          They collectively access pages 0, 2, 3, 7.

  inodes:                1   (one file = one inode)
  address_spaces:        1   (one per inode)
  struct files:          3   (one per open() call)
  struct pages:          4   (one per cached 4KB chunk)

  ┌─────────────────────────────────────────────────────────┐
  │                                                         │
  │  Process A: fd 3 → struct file (f_pos=0, RDONLY)        │
  │  Process B: fd 7 → struct file (f_pos=20000, RDWR)     │
  │  Process C: fd 4 → struct file (f_pos=8192, RDONLY)    │
  │       │         │         │                             │
  │       │         │         │                             │
  │       └─────────┴─────────┘                             │
  │                 │                                       │
  │                 ▼    all three f_mapping point here      │
  │           ┌──────────────────────────────────┐          │
  │           │ address_space (inode 4521)        │          │
  │           │                                  │          │
  │           │ i_pages:                         │          │
  │           │   [0] → struct page (0-4095)     │          │
  │           │   [2] → struct page (8192-12287) │          │
  │           │   [3] → struct page (12288-16383)│          │
  │           │   [7] → struct page (28672-32767)│          │
  │           │                                  │          │
  │           │ (pages 1,4,5,6,8,9 not cached)   │          │
  │           └──────────────────────────────────┘          │
  │                 │                                       │
  │                 ▼                                       │
  │           ┌──────────────────────────────────┐          │
  │           │ inode 4521                        │          │
  │           │   i_size = 40960                  │          │
  │           │   i_mapping = &i_data (above)     │          │
  │           └──────────────────────────────────┘          │
  │                                                         │
  └─────────────────────────────────────────────────────────┘
```

**If Process C closes its fd:** One struct file is destroyed. But the
struct pages and address_space are UNAFFECTED — they belong to the inode,
not to any process.

**If ALL processes close their fds:** All struct files are destroyed.
The struct pages STILL EXIST in the page cache (for future opens — the
kernel is smart about keeping cached data). They'll only be evicted when
RAM is needed.

**If the file is deleted (rm):** The inode's link count drops to 0. If no
process has it open (no struct files), the inode is freed, the address_space
is freed, all struct pages are freed, and the disk blocks are released.


---

# Chapter 8: Lifecycle Comparison

```
STRUCT FILE lifecycle:

  open() ──────► CREATED
                   │
                   │ (process reads, writes, seeks — f_pos changes)
                   │ (maybe dup() — another fd, same struct file, f_count++)
                   │
  close() ─────► f_count-- 
                   │
                   ├─ f_count > 0? Keep it (another fd still references it)
                   │
                   └─ f_count = 0? DESTROYED. Gone forever.
                      The inode and struct pages are NOT affected.


STRUCT PAGE lifecycle:

  First access ──► ALLOCATED from free page pool
                     │
                     │ (kernel reads file data from disk into this page)
                     │ page->flags |= PG_uptodate
                     │
                     │ (cached in address_space for fast re-access)
                     │
                     ├─ Read by processes → refcount++, then refcount--
                     │
                     ├─ Written to → flags |= PG_dirty
                     │                "I've been modified!"
                     │
                     ├─ Flushed (writeback or fsync) → flags &= ~PG_dirty
                     │                "I'm clean again — matches disk."
                     │
                     ├─ Stays in cache (even if nobody is using it!)
                     │  (keeping it cached makes future accesses fast)
                     │
                     └─ RAM pressure → EVICTED
                        ├─ Clean? Just throw it away (disk has the data).
                        └─ Dirty? Write to disk FIRST, then throw away.
                           FREED — returned to the free page pool.
```


---

# Chapter 9: What Information Lives WHERE?

People get confused about which object stores which information.
Here's the definitive answer:

```
"What file am I working with?"
  → struct file → f_inode → inode number, file size, permissions

"Where am I in the file?"
  → struct file → f_pos (your personal cursor)

"Can I write to this file?"
  → struct file → f_mode (FMODE_WRITE?)

"How was this file opened?"
  → struct file → f_flags (O_RDONLY? O_APPEND?)

"Which filesystem handles this?"
  → struct file → f_op (ext4_file_operations?)

"Where is the page cache?"
  → struct file → f_mapping → address_space

"Is page 5 of the file in RAM?"
  → address_space → i_pages[5] → struct page (or NULL)

"Has page 5 been modified?"
  → struct page → flags & PG_dirty

"Which file does this RAM chunk belong to?"
  → struct page → mapping → address_space → host → inode

"What byte range does this RAM chunk hold?"
  → struct page → index
  → bytes: index * 4096 to (index + 1) * 4096 - 1

"Where is the actual data in physical RAM?"
  → struct page → page_address(page) → kernel virtual address
  → page_to_pfn(page) * 4096 → physical address

"How big is the file?"
  → inode → i_size (NOT in struct file, NOT in struct page)

"How many pages of this file are cached?"
  → address_space → nrpages
```


---

# Chapter 10: Connection to Your Storage Engine

```
KERNEL                           YOUR STORAGE ENGINE
──────────────                   ─────────────────────────────

struct file                  ←→  Database cursor / Transaction handle
  f_pos (where in the file)      cursor position (which key we're at)
  f_flags (read/write mode)      transaction access mode
  f_mapping (→ page cache)       reference to the buffer pool
  PERSONAL (one per session)     PERSONAL (one per transaction)

struct page                  ←→  Buffer pool Frame
  4KB of cached file data         4KB of cached page data
  PG_dirty flag                   frame.dirty flag
  mapping (back to inode)         frame.page_id (which page)
  index (which chunk)             (the page_id IS the index)
  LRU list (eviction order)       LRU-K tracking
  SHARED (all processes)          SHARED (all transactions read same frame)

address_space                ←→  BufferPool
  i_pages (page index → page)    page_table (page_id → frame_index)
  nrpages (how many cached)       occupied_frames count
  a_ops (read/write methods)      file I/O methods
  ONE per file                     ONE per database

inode                        ←→  The database file header (page 0)
  i_size (file size)              total_pages * PAGE_SIZE
  i_ino (identity)                the file path
  i_mapping (→ page cache)        → buffer pool
  ONE per file                     ONE per database
```

**The critical parallel:**
- `struct file` is like opening a transaction or creating a cursor.
  Each session has its own state (position, permissions) but shares
  the underlying data.
- `struct page` is like a frame in your buffer pool.
  It holds cached data, can be dirty, can be evicted, and is shared
  by everyone accessing the same database.


---

# Chapter 11: The Question That Ties It All Together

```
"Why can't struct file just contain the data directly?"

Because:
  1. Two processes opening the same file need DIFFERENT cursors
     but the SAME cached data. If data lived in struct file,
     you'd have two copies of the same data — wasting RAM and
     creating consistency problems.

  2. Cached data should survive close(). If you close and reopen
     a file, the data should still be in cache. If data lived in
     struct file, closing would throw away the cache.

  3. Data is 4KB chunks that can be dirty, locked, evicted.
     Session state is position, flags, permissions.
     These are fundamentally different concerns with different
     lifetimes and different sharing rules.

"Why can't struct page track the cursor position?"

Because:
  1. Multiple processes read the same page at different positions.
     Process A is reading byte 100 of page 0.
     Process B is reading byte 3000 of page 0.
     The page can't have two cursors.

  2. The cursor advances with every read/write. Pages don't move.
     The cursor is per-SESSION. The page is per-DATA-CHUNK.

  3. A page might not be associated with any open file.
     Cached pages can exist long after all fds are closed.
     They have no concept of "current position."

SEPARATION OF CONCERNS:
  struct file = SESSION state   (who, where, what mode)
  struct page = DATA state      (what bytes, dirty?, cached?)

Same reason your storage engine separates:
  Transaction = session state   (which transaction, isolation level)
  Frame       = data state      (which page, dirty?, pinned?)
```


---

# Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  struct file = "Someone is using this file"                     │
│    → Created by open(), destroyed by close()                    │
│    → Holds: cursor position, access mode, permissions           │
│    → One per open() call, PERSONAL to the session               │
│    → Contains ZERO bytes of file data                           │
│                                                                 │
│  struct page = "A piece of file data is sitting in RAM"         │
│    → Created on first access, destroyed on eviction             │
│    → Holds: metadata about 4KB of physical RAM                  │
│    → One per cached 4KB chunk, SHARED by all sessions           │
│    → Points to 4096 bytes of actual file content                │
│                                                                 │
│  They work TOGETHER:                                            │
│    struct file (via f_mapping) → address_space → struct page    │
│    "Use my session to find my file's cache to find the data"    │
│                                                                 │
│  They exist SEPARATELY because:                                 │
│    Session state ≠ data state                                   │
│    Per-process ≠ shared                                         │
│    Lives until close() ≠ lives until eviction                   │
│    Many sessions, one cache                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
