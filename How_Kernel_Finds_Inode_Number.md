# How Does the Kernel Find the Inode Number?

## From "/data/database.db" to Inode 4521 — Every Step

---

# Chapter 1: The Question

You write this code:

```rust
let file = File::open("/data/database.db")?;
```

And somehow the kernel finds inode 4521. But HOW? You gave it a
STRING (a path), not a number. Where does the inode number come from?

**The short answer:** The inode number is stored INSIDE the parent
directory. Directories are just files that contain a list of
(name → inode number) pairs. The kernel reads directories to find
inode numbers.

**The long answer:** Let's trace every step.


---

# Chapter 2: A Directory Is a File Containing a Table

This is the key insight that unlocks everything: **a directory is NOT
a special magical thing. It's a regular file whose CONTENT is a table
of (name, inode number) pairs.**

When you run `ls /data/`, you're reading the CONTENTS of the directory
file "/data/". Those contents look like this:

```
The file "/data/" (which is itself an inode, say inode 500):

Its data blocks contain:

  ┌────────────────────────────────────────────────────────┐
  │ Directory entries (stored in the data blocks):          │
  │                                                        │
  │  Name              Inode Number    Type                │
  │  ──────────────    ────────────    ────                │
  │  "."               500             directory           │
  │  ".."              2               directory           │
  │  "database.db"     4521            regular file   ★    │
  │  "backup.db"       4522            regular file        │
  │  "logs"            4600            directory           │
  │  "config.yaml"     4700            regular file        │
  │                                                        │
  └────────────────────────────────────────────────────────┘
```

**That's it.** The inode number 4521 is literally written in the
parent directory's data. The directory "/data/" is a file that says:
"the name 'database.db' corresponds to inode 4521."

**Analogy:** A directory is like a CONTACTS LIST on your phone.
Each entry has a name ("Mohamed") and a phone number (4521).
The contacts list doesn't contain Mohamed himself — it just has
his number. You use the number to call him (read the inode).


---

# Chapter 3: The On-Disk Format of a Directory Entry

On ext4, each directory entry is a small struct stored in the
directory's data blocks:

```c
// ext4 directory entry (on disk):

struct ext4_dir_entry_2 {
    __le32  inode;          // 4 bytes: the inode number (e.g., 4521)
    __le16  rec_len;        // 2 bytes: total size of this entry
    __u8    name_len;       // 1 byte: length of the name string
    __u8    file_type;      // 1 byte: file type (1=file, 2=dir, 7=symlink)
    char    name[255];      // up to 255 bytes: the filename
};
```

For the entry "database.db" → inode 4521:

```
Bytes on disk inside the directory's data block:

  Offset  Hex                              Meaning
  ──────  ───────────────────────────      ─────────────────
  0x00    A9 11 00 00                      inode = 0x000011A9 = 4521
  0x04    20 00                            rec_len = 32 (total entry size)
  0x06    0B                               name_len = 11 ("database.db")
  0x07    01                               file_type = 1 (regular file)
  0x08    64 61 74 61 62 61 73 65 2E 64 62 name = "database.db"
  0x13    00                               padding to align to 4 bytes
```

**The inode number is just a 4-byte integer sitting in a directory
entry.** Nothing magical. When the kernel needs to look up
"database.db", it reads the directory's data blocks and scans
the entries until it finds a matching name.


---

# Chapter 4: The Complete Path Walk — Step by Step

When you open "/data/database.db", the kernel performs **path
resolution** — walking the path component by component, left to right.

### Starting Point: The Root Directory

Every filesystem has a known starting point: **inode 2 is always the
root directory.** This is hardcoded — the kernel doesn't need to look
it up. It's like knowing "the building's front door is always at
street level."

```
The kernel KNOWS: inode 2 = root directory "/"
This is defined by the ext4 specification. Always inode 2.
```

### Step 1: Read the Root Directory to Find "data"

```
Path so far: "/"
Remaining:   "data/database.db"
Current inode: 2 (root directory)

ACTION: Read inode 2's data blocks.

  Step 1a: Find inode 2 on disk.
    Inode table starts at block 4.
    Inode 2 is at: block 4 + ((2-1) / 16) = block 4
    Offset within block: ((2-1) % 16) × 256 = 256 bytes
    Read 256 bytes → got inode 2.

  Step 1b: Read inode 2's extent tree.
    The extent tree says: root directory's data is at disk block 68.

  Step 1c: Read disk block 68 (root directory's data).
    Contents:
      ┌─────────────────────────────────────────┐
      │  "."       → inode 2    (self)          │
      │  ".."      → inode 2    (parent = self) │
      │  "bin"     → inode 100  (directory)     │
      │  "etc"     → inode 200  (directory)     │
      │  "home"    → inode 300  (directory)     │
      │  "data"    → inode 500  (directory) ★   │
      │  "tmp"     → inode 600  (directory)     │
      └─────────────────────────────────────────┘

  Step 1d: Scan entries for name = "data".
    Found! "data" → inode 500.

RESULT: "data" is inode 500.
```

### Step 2: Read the "/data" Directory to Find "database.db"

```
Path so far: "/data/"
Remaining:   "database.db"
Current inode: 500 (the "/data" directory)

ACTION: Read inode 500's data blocks.

  Step 2a: Find inode 500 on disk.
    Block = 4 + ((500-1) / 16) = block 35
    Offset = ((500-1) % 16) × 256 = 768 bytes
    Read 256 bytes → got inode 500.

  Step 2b: Read inode 500's extent tree.
    Data is at disk block 5042.

  Step 2c: Read disk block 5042 ("/data" directory's data).
    Contents:
      ┌─────────────────────────────────────────────┐
      │  "."              → inode 500   (self)       │
      │  ".."             → inode 2     (parent = /) │
      │  "database.db"    → inode 4521  ★ FOUND!     │
      │  "backup.db"      → inode 4522               │
      │  "logs"           → inode 4600               │
      │  "config.yaml"    → inode 4700               │
      └─────────────────────────────────────────────┘

  Step 2d: Scan entries for name = "database.db".
    Found! "database.db" → inode 4521.

RESULT: "database.db" is inode 4521.
```

### Step 3: Read Inode 4521 (the Actual File)

```
Path fully resolved: "/data/database.db" = inode 4521

ACTION: Read inode 4521 from the inode table.

  Block = 4 + ((4521-1) / 16) = block 286
  Offset = ((4521-1) % 16) × 256 = 1280 bytes
  Read 256 bytes → got inode 4521.

  Inode 4521 tells us:
    i_mode = 0100644 (regular file)
    i_size = 1000000 (1MB)
    i_uid = 1000
    Extents: blocks 50000-50099, 80000-80143

NOW the kernel can create struct file and start reading/writing.
```


---

# Chapter 5: The Dentry Cache — Why This Isn't Slow

You might think: "Reading directories from disk for every path
component? That's 2-3 disk reads just to open a file!"

Yes — the FIRST time. After that, the kernel caches the results in
the **dentry cache (dcache)**:

```
FIRST open("/data/database.db"):
  Read root dir from disk → find "data" → inode 500
  Read /data dir from disk → find "database.db" → inode 4521
  Read inode 4521 from disk
  3 disk reads. Slow (~300 microseconds).

  But now the dcache remembers:
    dcache["/" + "data"] → inode 500
    dcache["/data" + "database.db"] → inode 4521

SECOND open("/data/database.db"):
  dcache["/" + "data"] → 500 (cache hit, no disk read!)
  dcache["/data" + "database.db"] → 4521 (cache hit!)
  inode cache[4521] → cached inode (cache hit!)
  0 disk reads. Fast (~200 nanoseconds).
```

**The dcache is a hash table in kernel memory:**

```
Key: (parent_inode_number, name_string)
Value: child_inode_number + dentry struct

  (2, "data")         → 500
  (500, "database.db") → 4521
  (500, "backup.db")   → 4522
  (2, "etc")           → 200
  (200, "hostname")    → 2001
  ...
```

After the first access, path resolution is purely in-memory hash table
lookups. No disk I/O at all. This is why `open()` is fast in practice
even though the path walk sounds expensive.


---

# Chapter 6: Large Directories — How ext4 Finds Names Fast

What if a directory has 100,000 files? Scanning every entry sequentially
would be terrible. ext4 solves this with an **HTree** — a hash-based
B-tree inside the directory:

```
Small directory (≤ ~200 entries):
  LINEAR SCAN — just read entries one by one.
  
  Data block: [entry1][entry2][entry3]...[entry200]
  
  To find "database.db": scan from start until found.
  200 entries × ~20 bytes each = one 4KB block. Fast enough.


Large directory (> ~200 entries):
  HTREE — a 2-level hash tree.
  
  Step 1: Hash the filename.
    hash("database.db") = 0x4A3B2C1D
  
  Step 2: Look up the hash in the HTree root.
    HTree root (in the directory's first data block):
      Hashes 0x00000000-0x3FFFFFFF → data block 5043
      Hashes 0x40000000-0x7FFFFFFF → data block 5044  ★
      Hashes 0x80000000-0xBFFFFFFF → data block 5045
      Hashes 0xC0000000-0xFFFFFFFF → data block 5046
    
    0x4A3B2C1D falls in range 0x40000000-0x7FFFFFFF → block 5044.
  
  Step 3: Read block 5044 and scan for "database.db".
    This block contains only ~250 entries (those with matching hashes).
    Much faster than scanning 100,000!
  
  Total: 1 block read (HTree root, likely cached) + 1 block read (leaf).
  O(1) lookup even in directories with millions of files.
```

**This is the same concept as your B+ tree!**
- Your B+ tree: hash/compare the KEY → walk the tree → find the VALUE.
- ext4's HTree: hash the FILENAME → walk the tree → find the INODE NUMBER.
- Both solve the same problem: fast lookup in a large sorted/hashed collection.


---

# Chapter 7: When a New File Is Created

When you create a new file, the kernel must:
1. Allocate a new inode number
2. Add a directory entry in the parent directory

```
File::create("/data/newfile.txt"):

Step 1: Path resolution
  Walk "/data/" → inode 500 (the parent directory).
  Try to find "newfile.txt" in /data → NOT FOUND.
  (This is expected — we're creating a new file.)

Step 2: Allocate a new inode number
  Read the INODE BITMAP (a 4KB block with one bit per inode):
  
  ┌──────────────────────────────────────────────────────┐
  │ Inode bitmap (one bit per inode):                     │
  │                                                      │
  │ Bit 0:    1 (inode 1 used — bad blocks file)         │
  │ Bit 1:    1 (inode 2 used — root directory)          │
  │ ...                                                  │
  │ Bit 4520: 1 (inode 4521 used — database.db)          │
  │ Bit 4521: 1 (inode 4522 used — backup.db)            │
  │ Bit 4522: 0 ← FIRST FREE! Allocate inode 4523.       │
  │ ...                                                  │
  └──────────────────────────────────────────────────────┘
  
  Set bit 4522 to 1. New inode number = 4523.

Step 3: Initialize the new inode
  Find inode 4523's position in the inode table:
    Block = 4 + ((4523-1) / 16) = block 286
    Offset = ((4523-1) % 16) × 256 = 1792 bytes
  
  Write 256 bytes of inode data:
    i_mode = 0100644 (regular file)
    i_size = 0 (empty file)
    i_uid = 1000 (your user)
    i_links_count = 1 (one name points to this inode)
    i_block = empty extent tree (no data yet)

Step 4: Add directory entry in parent
  Read /data directory's data blocks (inode 500's data at block 5042).
  Append a new entry:
  
  ┌─────────────────────────────────────────────────────┐
  │ ... existing entries ...                             │
  │ "newfile.txt"    → inode 4523   type=regular file    │ ← NEW
  └─────────────────────────────────────────────────────┘
  
  If the directory's data block is full, allocate a new block.
  If using HTree, insert into the correct hash bucket.

Step 5: Update the parent directory's inode
  inode 500 (the /data directory):
    i_mtime = now (directory was modified)
    i_size may increase (if a new data block was allocated)

NOW: "newfile.txt" exists. Its inode number (4523) is stored in the
     /data directory's data blocks. Anyone looking up "newfile.txt"
     will find inode 4523 by reading that directory.
```


---

# Chapter 8: When a File Is Deleted

```
std::fs::remove_file("/data/database.db"):

Step 1: Path resolution
  Walk "/data/" → inode 500.
  Find "database.db" in /data → inode 4521.

Step 2: Remove the directory entry
  Read /data directory's data blocks.
  Find the entry for "database.db".
  Mark it as deleted (set inode number to 0, or reclaim the space).
  
  BEFORE:
    "database.db"  → inode 4521
  
  AFTER:
    (entry removed or zeroed out)

Step 3: Decrement the inode's link count
  inode 4521: i_links_count = 1 → 0
  
  "Nobody points to me anymore."

Step 4: If links_count == 0 AND no process has it open:
  FREE the inode:
    Clear inode 4521 in the inode table.
    Clear bit 4520 in the inode bitmap (inode now free for reuse).
  
  FREE the data blocks:
    Return blocks 50000-50099 and 80000-80143 to the block bitmap.
    (These blocks can now be allocated to other files.)
  
  The file is GONE. The inode number 4521 can be reused for a future file.

Step 5: If links_count == 0 BUT a process still has it open:
  DON'T free anything yet!
  The process can still read/write through its fd.
  When the last fd is closed → THEN free the inode and blocks.
  (This is the "ghost file" scenario.)
```


---

# Chapter 9: Hard Links — Two Names, One Inode

Hard links reveal how this system works:

```
std::os::unix::fs::hard_link("/data/database.db", "/data/db_link"):

Step 1: Resolve "/data/database.db" → inode 4521.

Step 2: Add a NEW directory entry in /data:
  "db_link" → inode 4521   ← SAME inode number!

Step 3: Increment inode 4521's link count:
  i_links_count = 1 → 2

NOW:
  /data directory contains:
    "database.db"  → inode 4521
    "db_link"      → inode 4521    ← SAME inode!
    
  Both names point to the SAME inode.
  The SAME data blocks.
  The SAME file content.
  
  The inode doesn't know or care about either name.
  It just knows: "2 things point to me."

If you delete "database.db":
  i_links_count = 2 → 1
  The inode and data blocks are NOT freed (link count > 0).
  "db_link" still works perfectly.
  
If you then delete "db_link":
  i_links_count = 1 → 0
  NOW the inode and data blocks are freed.
```

**This proves that the filename is NOT the file.** The inode is the file.
Filenames are just entries in directory tables that point to inodes.
A file can have zero names (deleted but open), one name (normal), or
many names (hard links). The inode doesn't change.


---

# Chapter 10: The Inode Number Is a Physical Address (Sort Of)

The inode number ISN'T just an arbitrary ID. It's a **computable
physical location** on disk:

```
Given inode number N, you can calculate EXACTLY where it is on disk:

  inodes_per_block = 4096 / 256 = 16
  inode_table_start = block 4 (for block group 0)
  
  block_number = inode_table_start + (N - 1) / inodes_per_block
  byte_offset = ((N - 1) % inodes_per_block) * 256

Example: inode 4521
  block = 4 + (4521 - 1) / 16 = 4 + 282 = block 286
  offset = ((4521 - 1) % 16) × 256 = (8 × 256) = 2048 bytes

  Inode 4521 is at: disk block 286, byte offset 2048.
  
This is O(1) — pure arithmetic, no searching.
The inode number IS an index into the inode table.
```

**Analogy:** An inode number is like a seat number in a theater.
"Seat 4521" tells you EXACTLY which row and which seat — you don't
need to search. Row = 4521 / seats_per_row. Position = 4521 % seats_per_row.

**This is why inodes are fast:** Given an inode number, the kernel
computes the disk location with simple math and reads it directly.
No searching, no tree walking, no hash lookups. O(1).

**The expensive part is finding the inode number** — that requires
reading directories. But once you HAVE the number, reading the
inode itself is instant.


---

# Chapter 11: The Complete Chain — From String to Bytes

```
open("/data/database.db", O_RDWR)

  "/data/database.db"                      ← a STRING (path)
         │
    ┌────┴────────────────────┐
    │ PATH RESOLUTION          │
    │                         │
    │ Start: inode 2 (root)    │
    │   Read root dir data     │
    │   Find "data" → 500      │  ← inode number from directory entry
    │                         │
    │ Now: inode 500 (/data)   │
    │   Read /data dir data    │
    │   Find "database.db"     │
    │   → 4521                 │  ← inode number from directory entry
    └────┬────────────────────┘
         │
         ▼
    inode number = 4521                     ← a NUMBER (32-bit integer)
         │
    ┌────┴────────────────────┐
    │ INODE TABLE LOOKUP       │
    │                         │
    │ block = 4 + 4520/16     │
    │       = block 286       │
    │ offset = (4520%16)×256  │
    │        = 2048 bytes     │
    │                         │
    │ Read disk block 286,    │
    │ extract 256 bytes at    │
    │ offset 2048.            │
    └────┬────────────────────┘
         │
         ▼
    struct inode (in memory)                ← a KERNEL OBJECT (256 bytes)
    ┌──────────────────────┐
    │ i_size = 1000000     │
    │ i_mode = 0100644     │
    │ extents:             │
    │  [0-99] → 50000      │
    │  [100-243] → 80000   │
    └────┬─────────────────┘
         │
    ┌────┴────────────────────┐
    │ EXTENT TREE LOOKUP       │
    │                         │
    │ "Give me byte 8192"     │
    │ page_index = 2          │
    │ extent 1: [0-99]→50000  │
    │ block = 50000 + 2       │
    │       = 50002           │
    └────┬────────────────────┘
         │
         ▼
    disk block 50002                        ← a PHYSICAL LOCATION on disk
         │
    ┌────┴────────────────────┐
    │ DISK READ                │
    │                         │
    │ Read 4096 bytes from    │
    │ disk block 50002.       │
    └────┬────────────────────┘
         │
         ▼
    "Hello World..."                        ← YOUR ACTUAL DATA (bytes)


  THE CHAIN:
    path string → directory lookup → inode number
    → inode table (arithmetic) → inode struct
    → extent tree → physical block number
    → disk read → your data bytes

  EACH ARROW is a different lookup mechanism:
    path → inode number:  directory scan or HTree hash lookup
    inode number → inode: arithmetic (O(1), instant)
    inode → block number: extent tree (B-tree, O(log N))
    block number → data:  disk read (physical I/O)
```


---

# Chapter 12: Connection to Your Storage Engine

```
ext4's lookup chain:                Your FerrisDB's lookup chain:
──────────────────                  ──────────────────────────────

Filename "database.db"          ←→  Key "account_1001"

Directory (name → inode number) ←→  B+ tree (key → RID)
  "database.db" → 4521             "account_1001" → RID(7, 15)

Inode number (4521)             ←→  Page ID (7)
  Computed position in table        Computed position in file
  O(1) arithmetic                   page_offset = 7 × 4096

Extent tree (logical → physical)←→  (not needed — your pages ARE at
  file block 2 → disk block 50002    their computed offsets in the file)
                                     ext4 handles the physical mapping
                                     transparently underneath.

Disk block read                 ←→  page.read_at(7 × 4096)
                                    (ext4 translates this to a disk block
                                     using ITS extent tree — your code
                                     doesn't see this translation)

The inode number is ext4's equivalent of your page_id.
The directory is ext4's equivalent of your B+ tree.
```

**The profound realization:** When your storage engine does
`file.read_at(&mut buf, page_id * 4096)`, ext4 is doing its OWN
lookup chain underneath: file offset → extent tree → physical disk
block. Your page_id goes through TWO levels of translation:

```
Your page_id = 7
  → Your code: offset = 7 × 4096 = 28672
    → ext4: file offset 28672 → logical block 7
      → ext4 extent tree: logical 7 → physical block 50007
        → disk: read block 50007 → your 4096 bytes

Two mapping layers, invisible to each other.
Your code only knows about page IDs.
ext4 only knows about file offsets and disk blocks.
The disk only knows about block numbers.
```


---

# Summary

```
Q: "How do we know the inode number?"
A: It's stored in the PARENT DIRECTORY's data.
   Directories are files containing (name → inode number) tables.

Q: "How does the kernel find the parent directory?"
A: By walking the path from left to right, starting at inode 2 (root).
   Each directory lookup gives the inode number of the next component.

Q: "Isn't reading directories from disk slow?"
A: Only the first time. The dentry cache (dcache) keeps results in RAM.
   Subsequent lookups are hash table hits — nanoseconds, no disk I/O.

Q: "How does the kernel find the inode once it has the number?"
A: Pure arithmetic. Inode N is at a computable position in the inode table.
   O(1) lookup — no searching needed.

Q: "What about large directories with 100,000 files?"
A: ext4 uses an HTree (hash tree) — O(1) lookup by hashing the filename.
   Same concept as your B+ tree.

THE CHAIN:
  path string → directory → inode number → inode table → inode → extent tree → disk block → data
  
  Each step uses a different lookup mechanism,
  each optimized for its specific job.
```
