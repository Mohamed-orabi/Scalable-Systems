# Pages on Disk vs Inodes — Who Holds What?

## The Complete Guide to the Most Confused Relationship in Filesystems

---

# Chapter 1: The Confusion

People think: "The inode holds my file data." **It doesn't.**
People think: "Pages on disk ARE my file data." **They are, but they
don't know which file they belong to.**

The truth:

```
The INODE is the librarian's index card.
  → It knows ABOUT the book (size, author, date, which shelves).
  → It contains almost NO actual book content.

The DATA BLOCKS (pages on disk) are the actual shelves holding the book.
  → They contain the actual book pages (your file's bytes).
  → They have NO idea which book they belong to.

The inode POINTS TO the data blocks.
The data blocks DON'T point back to the inode.
```


---

# Chapter 2: What the Inode Actually Stores

Let's look at an ext4 inode on disk. It's exactly 256 bytes:

```
ext4 inode for "database.db" (inode 4521):

256 bytes total — that's it. ONE QUARTER of a kilobyte.

  ┌──────────────────────────────────────────────────────────┐
  │                                                          │
  │  METADATA (information ABOUT the file):                  │
  │                                                          │
  │    i_mode = 0100644        "regular file, rw-r--r--"     │ 2 bytes
  │    i_uid = 1000            "owned by user 1000"          │ 2 bytes
  │    i_gid = 1000            "group 1000"                  │ 2 bytes
  │    i_size = 1,000,000      "file is 1MB"                 │ 8 bytes
  │    i_atime = 1709251200    "last accessed March 1"       │ 4 bytes
  │    i_mtime = 1709251200    "last modified March 1"       │ 4 bytes
  │    i_ctime = 1709200000    "metadata last changed"       │ 4 bytes
  │    i_links_count = 1       "1 filename points to me"     │ 2 bytes
  │    i_blocks = 2048         "using 2048 × 512B = 1MB"     │ 4 bytes
  │    i_flags = 0x80000       "using extents"               │ 4 bytes
  │                                                          │
  │  THE MAP (where is my data on disk):                     │
  │                                                          │
  │    i_block[0..14] (60 bytes) = EXTENT TREE               │ 60 bytes
  │    ┌──────────────────────────────────────────────────┐  │
  │    │ "My data bytes 0-409599 are at disk blocks       │  │
  │    │  50000-50099 (contiguous, 100 blocks)"           │  │
  │    │                                                  │  │
  │    │ "My data bytes 409600-999999 are at disk blocks  │  │
  │    │  80000-80143 (contiguous, 144 blocks)"           │  │
  │    └──────────────────────────────────────────────────┘  │
  │                                                          │
  │  THAT'S IT. No file data. No bytes of "Hello World."     │
  │  The inode is a 256-byte INDEX CARD.                     │
  │                                                          │
  └──────────────────────────────────────────────────────────┘
```

**The inode is 256 bytes. The file is 1,000,000 bytes.** The inode
is a tiny label that says WHERE the million bytes are, not the bytes
themselves. Like a treasure map — the map is small, the treasure is big.


---

# Chapter 3: What the Data Blocks Actually Store

The data blocks (pages on disk) hold the ACTUAL file content — your
bytes, your records, your "Hello World":

```
Disk layout for database.db (1MB file):

  Disk block 50000: bytes 0-4095 of the file
    ┌──────────────────────────────────────────────┐
    │ 48 65 6C 6C 6F 20 57 6F 72 6C 64 00 00 ...  │
    │ "H  e  l  l  o     W  o  r  l  d"            │
    │                                              │
    │ This IS your actual file data.               │
    │ 4096 bytes of content.                       │
    │ It has NO header saying "I belong to inode   │
    │ 4521" — it's just raw bytes.                 │
    └──────────────────────────────────────────────┘

  Disk block 50001: bytes 4096-8191 of the file
    ┌──────────────────────────────────────────────┐
    │ More file data... records... whatever you     │
    │ wrote to this part of the file.              │
    └──────────────────────────────────────────────┘

  Disk block 50002: bytes 8192-12287 of the file
    ┌──────────────────────────────────────────────┐
    │ More file data...                            │
    └──────────────────────────────────────────────┘

  ...

  Disk block 50099: bytes 405504-409599 of the file
    ┌──────────────────────────────────────────────┐
    │ Last block of the first extent.              │
    └──────────────────────────────────────────────┘

  ── GAP: blocks 50100-79999 belong to OTHER files ──

  Disk block 80000: bytes 409600-413695 of the file
    ┌──────────────────────────────────────────────┐
    │ File data continues here (second extent).    │
    └──────────────────────────────────────────────┘

  ...

  Disk block 80143: bytes 995328-999423 of the file
    ┌──────────────────────────────────────────────┐
    │ Last bytes of the file.                      │
    └──────────────────────────────────────────────┘
```

**Key observation**: The data blocks have NO metadata about which file
they belong to. Block 50000 is just 4096 bytes of raw data. If you looked
at it in isolation, you'd have no idea it's part of database.db. Only the
inode knows the connection.


---

# Chapter 4: The Analogy — A Warehouse

```
THE INODE is the INVENTORY CARD in the manager's filing cabinet:

  ┌────────────────────────────────────────────────┐
  │ Item: "Database.db"                             │
  │ Size: 1,000,000 bytes                          │
  │ Owner: Employee #1000                          │
  │ Created: March 1, 2025                         │
  │                                                │
  │ Storage locations:                             │
  │   Shelves 50000-50099 (first 400KB)            │
  │   Shelves 80000-80143 (remaining 600KB)        │
  │                                                │
  │ Total shelves used: 244                        │
  └────────────────────────────────────────────────┘

  The card is SMALL (256 bytes).
  It tells you WHERE the stuff is, not WHAT the stuff is.


THE DATA BLOCKS are the actual SHELVES in the warehouse:

  Shelf 50000: [box containing bytes 0-4095 of the item]
  Shelf 50001: [box containing bytes 4096-8191]
  Shelf 50002: [box containing bytes 8192-12287]
  ...

  The shelves are BIG (4096 bytes each, 244 of them = 1MB).
  They hold the actual stuff, but have NO label saying
  "I belong to the Database.db inventory card."


To find the stuff:
  1. Find the inventory card (inode) in the filing cabinet
  2. Read the storage locations from the card
  3. Go to those shelves and get the boxes

To find which card a shelf belongs to:
  ??? You CAN'T. You'd have to search every card until you
  find one that mentions this shelf number. (The filesystem
  doesn't need to do this in normal operation.)
```


---

# Chapter 5: The Extent Tree — The Inode's Map

The 60 bytes of `i_block` inside the inode contain an **extent tree** —
a compact data structure that maps logical file offsets to physical disk
block numbers.

## 5.1: What an Extent Is

An extent says: "A CONTIGUOUS run of blocks."

```
Extent 1:
  logical_start = 0       "Starting at file offset 0"
  physical_start = 50000  "The data is at disk block 50000"
  length = 100            "100 contiguous blocks (400KB)"

  This ONE extent maps 100 blocks (409,600 bytes) with just 12 bytes of metadata!

  File offset 0      → disk block 50000
  File offset 4096   → disk block 50001
  File offset 8192   → disk block 50002
  ...
  File offset 405504 → disk block 50099

  To find any byte in this range:
    disk_block = 50000 + (file_offset / 4096)
```

Compare this to the old block-pointer system (ext2/ext3):
```
Old way (ext2): one pointer per block
  block[0] = 50000
  block[1] = 50001
  block[2] = 50002
  ...
  block[99] = 50099
  → 100 pointers × 4 bytes = 400 bytes of metadata!

New way (ext4 extents): one extent for contiguous blocks
  extent = {start=0, physical=50000, length=100}
  → 12 bytes of metadata for the same 100 blocks!
```

## 5.2: The Extent Tree Layout Inside the Inode

The 60 bytes of `i_block` can hold a small extent tree directly:

```
i_block (60 bytes) for a small file with 2 extents:

  ┌────────────────────────────────────────────────────────┐
  │ Extent Header (12 bytes):                               │
  │   magic = 0xF30A                                       │
  │   entries = 2          (we have 2 extents)              │
  │   max = 4              (room for up to 4 extents here)  │
  │   depth = 0            (this is a leaf — no subtrees)   │
  │   generation = 7                                        │
  ├────────────────────────────────────────────────────────┤
  │ Extent 1 (12 bytes):                                    │
  │   logical_block = 0                                     │
  │   length = 100                                          │
  │   physical_block_hi = 0                                 │
  │   physical_block_lo = 50000                             │
  │   → "File blocks 0-99 are at disk blocks 50000-50099"  │
  ├────────────────────────────────────────────────────────┤
  │ Extent 2 (12 bytes):                                    │
  │   logical_block = 100                                   │
  │   length = 144                                          │
  │   physical_block_hi = 0                                 │
  │   physical_block_lo = 80000                             │
  │   → "File blocks 100-243 at disk blocks 80000-80143"   │
  ├────────────────────────────────────────────────────────┤
  │ (space for 2 more extents — currently unused)           │
  └────────────────────────────────────────────────────────┘

  12 (header) + 2 × 12 (extents) = 36 bytes used out of 60.
  Room for 2 more extents without leaving the inode.
```

## 5.3: What Happens When the Inode's Space Runs Out

If a file is very fragmented (many non-contiguous extents), 4 extents
won't be enough. The extent tree GROWS into a multi-level tree — just
like a B+ tree:

```
Small file (≤4 extents): everything fits inside the 60-byte i_block

  inode.i_block:
    [header][extent 1][extent 2][extent 3][extent 4]


Large/fragmented file (>4 extents): the tree grows external nodes

  inode.i_block:
    [header][index 1 → block 90000][index 2 → block 90001]
    
    The inode now holds INDEX ENTRIES instead of extents.
    Each index entry points to a disk block containing more extents.
    
  Disk block 90000 (a leaf block of the extent tree):
    [header][extent 1][extent 2]...[extent 340]
    
    A 4KB block can hold ~340 extents!
    
  Disk block 90001 (another leaf block):
    [header][extent 341][extent 342]...

  This is a B-TREE of extents.
  The root is in the inode (60 bytes).
  The leaves are in separate disk blocks.
  This scales to files of any size with any fragmentation level.
```

**This is directly analogous to your B+ tree!**
- Your B+ tree maps: key → RID (record location)
- ext4's extent tree maps: file_block_number → disk_block_number
- Both are balanced trees stored in pages
- Both start small (fits in one node) and grow as needed


---

# Chapter 6: The Full Picture — Where Everything Lives on Disk

```
AN ext4 DISK LAYOUT:

  Block 0:     Superblock (filesystem metadata)
  Block 1:     Group descriptor table
  Block 2:     Block bitmap ("which blocks are free?")
  Block 3:     Inode bitmap ("which inodes are free?")
  Blocks 4-67: INODE TABLE (where ALL inodes live)
  
  Block 4 contains inodes 1-16    (256 bytes each, 16 per 4KB block)
  Block 5 contains inodes 17-32
  ...
  Block 67 contains inodes 1009-1024
  
  Blocks 68+:  DATA BLOCKS (where file contents live)
  
  Block 68:    data belonging to some file
  Block 69:    data belonging to some other file
  ...
  Block 50000: data belonging to database.db (first extent)
  Block 50001: data belonging to database.db
  ...
  Block 50099: data belonging to database.db (end of first extent)
  Block 50100: data belonging to yet another file
  ...
  Block 80000: data belonging to database.db (second extent)
  ...
  Block 80143: data belonging to database.db (end of second extent)
  Block 80144: data belonging to some other file
  ...
  Block 90000: extent tree leaf for a very fragmented file
```

**Notice**: The inodes and data blocks live in COMPLETELY SEPARATE regions
of the disk. The inode table is a dense, predictable array (inode N is at
a calculable position). The data blocks are scattered wherever the allocator
placed them.


---

# Chapter 7: The Three "Pages" That Confuse Everyone

The word "page" means THREE different things in this context. This is
the core of the confusion:

```
PAGE TYPE 1: DISK BLOCK (also called "disk page")
  ─────────────────────────────────────────────
  WHERE:    On the physical disk (SSD/HDD)
  SIZE:     4096 bytes (same as a page)
  WHAT:     A fixed-size chunk of the disk
  CONTAINS: Raw file data bytes, OR inode table entries,
            OR extent tree nodes, OR free space bitmaps
  WHO MANAGES: The filesystem (ext4) and block I/O layer
  
  Example: disk block 50000 contains bytes 0-4095 of database.db
  
  Analogy: A SHELF in the warehouse.


PAGE TYPE 2: PAGE CACHE PAGE (struct page in RAM)
  ─────────────────────────────────────────────
  WHERE:    In kernel RAM (physical memory)
  SIZE:     4096 bytes (matches disk block size)
  WHAT:     A CACHED COPY of a disk block
  CONTAINS: Same bytes as the corresponding disk block
            (or modified bytes if dirty)
  WHO MANAGES: The kernel's page cache (address_space)
  
  Example: struct page in the page cache holding a copy of disk block 50000
  
  Analogy: A PHOTOCOPY of the shelf's contents, placed on the DESK (RAM)
           for quick access.


PAGE TYPE 3: YOUR STORAGE ENGINE'S PAGE
  ─────────────────────────────────────────────
  WHERE:    Inside the database file's data blocks
  SIZE:     4096 bytes (matches everything)
  WHAT:     A page within YOUR database's file format
  CONTAINS: Your slotted page header, slot array, records —
            structured data that YOUR code manages
  WHO MANAGES: Your storage engine code
  
  Example: page 2 of your database file (at file offset 8192) contains
           a slotted page with 15 bank account records
  
  Analogy: A LABELED BOX on the shelf, with your own internal organization.
```

**How they relate:**

```
Your database page (type 3) is stored in a disk block (type 1),
which may be cached as a page cache page (type 2).

They're the SAME 4096 bytes seen from different levels:

  Your code sees:       "A slotted page with records"
  The kernel sees:      "A cached page for inode 4521, offset 8192"
  The disk sees:        "4096 bytes at block 50002"

  SAME BYTES. THREE PERSPECTIVES.

  ┌─────────────────────────────────────────────────────┐
  │ Your storage engine:                                │
  │   "Page 2 of my database. Slotted page. 15 records.│
  │    Slot 0 points to offset 4063. Slot 1 at 4030."  │
  │                                                     │
  │ The kernel page cache:                              │
  │   "struct page for inode 4521, index 2.             │
  │    dirty=true. uptodate=true. In the XArray."       │
  │                                                     │
  │ The disk:                                           │
  │   "Block 50002. 4096 bytes.                         │
  │    I don't know what's in here. Just bytes."        │
  │                                                     │
  │ ALL THREE ARE THE SAME 4096 BYTES.                  │
  └─────────────────────────────────────────────────────┘
```


---

# Chapter 8: The Inode Does NOT Contain File Data (With One Exception)

The inode is PURELY metadata + a map to data blocks. It contains zero
bytes of your "Hello World" file content.

**EXCEPT**: ext4 has a feature called **inline data** for very tiny files:

```
Normal file (any size):
  Inode: [metadata][extent tree pointing to data blocks]
  Data blocks: [actual file bytes]
  
  The inode just POINTS to the data.

Tiny file (≤60 bytes):
  Inode: [metadata][actual file bytes crammed into i_block!]
  No data blocks needed!
  
  The 60-byte i_block space that normally holds the extent tree
  instead holds the actual file content directly.

  Example: a file containing just "Hello" (5 bytes)
  
  inode.i_block = "Hello\0\0\0\0\0..." (5 bytes of data + padding)
  
  No extent tree. No data blocks. Everything fits in the inode.
  This saves one disk block (4KB) for a 5-byte file!
```

For your database file (which is megabytes or gigabytes), inline data
never applies. The inode ALWAYS just contains the extent tree.


---

# Chapter 9: How a Read Traverses From Inode to Data

Let's trace reading bytes 8192-8196 ("Hello") from database.db:

```
Step 1: FIND THE INODE
  We know the inode number: 4521.
  Inode table block = 4 + (4521 - 1) / 16 = block 286
  Offset within block = ((4521 - 1) % 16) * 256 = byte 1280
  
  Read disk block 286, extract 256 bytes at offset 1280.
  Now we have the inode in memory.

Step 2: PARSE THE EXTENT TREE
  Read the inode's i_block (60 bytes).
  Extent header says: 2 entries, depth=0 (leaf).
  
  Extent 1: logical_start=0, length=100, physical=50000
  Extent 2: logical_start=100, length=144, physical=80000

Step 3: MAP FILE OFFSET → DISK BLOCK
  We want file offset 8192.
  Logical block = 8192 / 4096 = 2.
  
  Which extent? Extent 1 covers logical blocks 0-99.
  Block 2 is in extent 1.
  
  Physical block = 50000 + (2 - 0) = 50002.

Step 4: READ THE DISK BLOCK
  Read disk block 50002 → 4096 bytes into a page cache page.

Step 5: EXTRACT THE BYTES
  Offset within page = 8192 % 4096 = 0.
  Read 5 bytes starting at offset 0: "Hello".

Done! We went:
  inode → extent tree → physical block number → disk read → bytes

The inode didn't HOLD "Hello". It told us WHERE "Hello" lives.
```


---

# Chapter 10: How a Write Goes From Data to Disk Block

Let's trace writing "World" at byte 8192 of database.db:

```
Step 1: FIND THE INODE (same as read)
  Inode 4521 → extent tree.

Step 2: MAP FILE OFFSET → DISK BLOCK (same as read)
  File offset 8192 → logical block 2 → extent 1 → physical block 50002.

Step 3: FETCH OR CREATE THE PAGE CACHE PAGE
  Look up address_space->i_pages[2]:
    → Found? Use it (cache hit).
    → Not found? Allocate new page, read block 50002 from disk.

Step 4: MODIFY THE PAGE IN RAM
  Write "World" at offset 0 within the page.
  Mark the page as DIRTY.

Step 5: (LATER, ON FSYNC) FLUSH TO DISK
  Write the dirty page to disk block 50002.
  Send FLUSH CACHE command to the drive.

The inode was USED (to find block 50002) but NOT MODIFIED
(unless the file size changed — then i_size is updated).
```


---

# Chapter 11: What Gets Modified in Each Scenario

```
SCENARIO: Write 5 bytes to the middle of an existing file

  MODIFIED:
    ✓ One data block (the one containing the written bytes)
    ✓ The page cache page (in RAM, marked dirty)
    ✓ Inode's i_mtime (modification timestamp)
    
  NOT MODIFIED:
    ✗ Inode's extent tree (file didn't grow, no new blocks allocated)
    ✗ Inode's i_size (file size unchanged)
    ✗ Other data blocks (we only touched one)
    ✗ Block bitmap (no new blocks allocated)
    ✗ Inode bitmap (no new inodes)


SCENARIO: Write 5 bytes that EXTEND the file (past current end)

  MODIFIED:
    ✓ One data block (new or existing)
    ✓ The page cache page
    ✓ Inode's i_mtime
    ✓ Inode's i_size (file got bigger!)
    ✓ Inode's i_blocks (more blocks used)
    ✓ Inode's extent tree (new extent or extended existing extent)
    ✓ Block bitmap (new block allocated — marked as "used")
    
  NOT MODIFIED:
    ✗ Inode bitmap (no new inodes)


SCENARIO: Create a new file and write 5 bytes

  MODIFIED:
    ✓ Inode bitmap (new inode allocated — marked as "used")
    ✓ New inode in the inode table (initialized with metadata)
    ✓ New extent in the inode (pointing to the data block)
    ✓ Block bitmap (new data block allocated)
    ✓ Data block (the 5 bytes written)
    ✓ Parent directory's data block (new directory entry added)
    ✓ Parent directory's inode (i_mtime updated)
    ✓ Page cache pages for all of the above
    
    That's a LOT of writes for creating one tiny file!
    This is why databases pre-allocate large files.
```


---

# Chapter 12: Side-by-Side Comparison

```
                        INODE                           DATA BLOCK
                        ─────                           ──────────

What it is              Index card for a file           Actual file content

Size                    256 bytes (fixed)               4096 bytes (fixed)

Where on disk           Inode table (predictable        Anywhere (allocated
                        location, dense array)          by block allocator)

How many per file       Exactly ONE                     One per 4KB of content
                                                        (1MB file = 244 blocks)

Contains file data?     No (except inline ≤60B)         YES — the actual bytes

Contains metadata?      YES — size, owner, times,       NO — just raw data bytes
                        permissions, extent tree

Knows its file?         YES — it IS the file's          NO — a data block has no
                        identity                        idea which file owns it

Finding it              By inode number (O(1) —         Through the inode's extent
                        calculate position in table)    tree (O(log N) lookup)

Created when            File is created (one inode      Data is first written
                        allocated per new file)         (blocks allocated on demand)

Freed when              File deleted AND all fds         File shrinks or is deleted
                        closed AND nlink=0              (blocks returned to free pool)

Modified by write()     Only metadata (mtime, maybe     YES — the actual bytes
                        size and extents)               are modified

Modified by chmod       YES (i_mode changes)            NO (data doesn't change)

Modified by touch       YES (timestamps change)         NO (data doesn't change)

Your engine parallel    Page 0 (meta page) — stores     Pages 1+ (data pages) —
                        metadata about the database     store your actual records
```


---

# Chapter 13: The Complete Relationship

```
┌──────────────────────────────────────────────────────────────────┐
│                         DISK LAYOUT                              │
│                                                                  │
│  Inode Table                    Data Blocks                      │
│  ┌─────────────────┐           ┌──────────────────────────┐     │
│  │ Inode 1: /       │           │ Block 68: dir entries    │     │
│  │ Inode 2: /home   │           │ Block 69: dir entries    │     │
│  │ ...              │           │ ...                      │     │
│  │ Inode 4521:      │           │ Block 50000: db page 0   │     │
│  │  i_size=1MB      │──────────►│ Block 50001: db page 1   │     │
│  │  extents:        │ "my data  │ Block 50002: db page 2   │     │
│  │   [0-99]→50000   │  is at    │ ...                      │     │
│  │   [100-243]→80000│  these    │ Block 50099: db page 99  │     │
│  │                  │  blocks"  │ ...                      │     │
│  │ Inode 4522:      │           │ Block 80000: db page 100 │     │
│  │  i_size=500      │──────┐    │ Block 80001: db page 101 │     │
│  │  extents:        │      │    │ ...                      │     │
│  │   [0-0]→72000    │      │    │ Block 80143: db page 243 │     │
│  │                  │      │    │ ...                      │     │
│  └─────────────────┘      └───►│ Block 72000: other file   │     │
│                                 └──────────────────────────┘     │
│                                                                  │
│  Inodes are SMALL (256B)        Data blocks are the BULK         │
│  and in a DENSE TABLE.          of disk usage, SCATTERED         │
│  They're the MAP.               wherever allocated.              │
│                                 They're the TERRITORY.            │
│                                                                  │
│  ★ The arrows go ONE WAY: inode → data blocks.                   │
│  ★ Data blocks do NOT point back to their inode.                 │
│  ★ The inode IS the file's identity. The blocks are anonymous.   │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```


---

# Chapter 14: Connection to Your Storage Engine

```
ext4 Filesystem                     Your FerrisDB Storage Engine
──────────────────                  ───────────────────────────────

Inode                           ←→  Page 0 (meta page)
  Stores: file size, extents,       Stores: page count, free list,
  permissions, timestamps           root B+ tree page, version
  256 bytes of metadata             4096 bytes of metadata
  Points to data blocks             Points to data pages

Extent tree (inside inode)      ←→  B+ tree root (in a page)
  Maps: file offset → block         Maps: key → RID
  A B-tree stored partially         A B+ tree stored in pages
  in the inode, partially in 
  separate blocks

Data blocks                     ←→  Data pages (heap pages)
  Hold: raw file content            Hold: slotted pages with records
  4096 bytes each                   4096 bytes each
  Don't know their inode            Don't know their B+ tree key
  Located via extent tree           Located via B+ tree or free space map

Block bitmap                    ←→  Free page list
  Tracks: which blocks are free     Tracks: which pages are free
  1 bit per block                   Linked list of free pages

Inode bitmap                    ←→  (no equivalent — you have one "file")
  Tracks: which inodes are free

Your database file is INSIDE ext4:
  ext4 sees your database as ONE file with ONE inode.
  ext4's extent tree maps your file's pages to disk blocks.
  YOUR page manager manages the pages INSIDE the file.
  TWO levels of page management: ext4's and yours.

  ext4 level:  inode 4521 → extent tree → disk blocks
  Your level:  meta page → B+ tree → heap pages (inside those disk blocks)
```

**The profound insight:** Your storage engine is a filesystem within a
filesystem. ext4 manages disk blocks for your database file. Your engine
manages pages WITHIN that file. Both use trees (ext4's extent tree, your
B+ tree) to map logical addresses to physical locations. Both have free
space tracking (block bitmap, free page list). Both have metadata
structures (inode, meta page). The patterns are identical — just at
different levels of the stack.
