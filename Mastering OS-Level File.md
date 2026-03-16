# Mastering OS-Level File I/O for Storage Engines

## From Electrons to Syscalls — A Complete Guide

---

# Chapter 1: The Physical Reality

Before touching any code, you need to understand what's physically happening when
your program "writes to a file." Most developers never learn this, and it's why
they write storage engines that lose data.

## 1.1 The Storage Stack (What Actually Happens)

When your Rust program calls `file.write_all(data)`, here's what ACTUALLY happens:

```
Your Rust program
    │
    ▼
┌──────────────────────────┐
│  libc / Rust std          │  write() system call wrapper
└──────────────────────────┘
    │  ← SYSCALL BOUNDARY (user mode → kernel mode)
    ▼
┌──────────────────────────┐
│  VFS (Virtual File System)│  Translates path → inode → filesystem
└──────────────────────────┘
    │
    ▼
┌──────────────────────────┐
│  Page Cache               │  ★ YOUR DATA LANDS HERE ★
│  (kernel memory)          │  This is NOT on disk yet!
└──────────────────────────┘
    │  ← This gap is where data loss happens
    ▼
┌──────────────────────────┐
│  Block I/O Layer          │  Merges, reorders, and schedules I/O requests
└──────────────────────────┘
    │
    ▼
┌──────────────────────────┐
│  Device Driver            │  Speaks the drive's protocol (SATA, NVMe, SCSI)
└──────────────────────────┘
    │
    ▼
┌──────────────────────────┐
│  Drive Controller         │  ★ ANOTHER CACHE ★ (drive's internal RAM)
│  (hardware)               │  STILL not on the physical medium!
└──────────────────────────┘
    │
    ▼
┌──────────────────────────┐
│  Physical Medium          │  Finally: magnetic platters / NAND flash cells
│  (HDD / SSD)             │  Data survives power loss HERE and only here.
└──────────────────────────┘
```

**The critical insight**: There are TWO caches between your program and the
physical disk. A simple `write()` call puts data in the kernel's page cache —
it could sit there for 30 seconds before the kernel decides to flush it.
If power goes out during those 30 seconds, your data is gone.

This is why `fsync()` exists, and why it's the most important syscall for
any database.


## 1.2 The Kernel Page Cache

The kernel maintains a cache of recently accessed disk pages in RAM.
When you read a file, the kernel first checks this cache. When you write,
the data goes to the cache and the page is marked "dirty."

```
Physical RAM
┌─────────────────────────────────────────────┐
│                                             │
│  Your program's memory  ←  malloc, stack    │
│  ─────────────────────                      │
│  Other programs                             │
│  ─────────────────────                      │
│  Kernel page cache      ←  file data lives  │
│  ─────────────────────      here after       │
│  Kernel code + data         write()          │
│                                             │
└─────────────────────────────────────────────┘
```

The page cache is why `read()` after `write()` returns the new data even
before `fsync()` — you're reading from the cache, not from disk.

**How the cache decides to flush** (writeback):
- A background thread (`pdflush` / `writeback`) runs every ~5 seconds
- It writes dirty pages that are older than `dirty_expire_centisecs` (default: 30 seconds)
- It also flushes when dirty pages exceed `dirty_ratio` (default: 20% of RAM)
- You can force it with `fsync()`, `fdatasync()`, or `sync()`

This means: after a `write()`, your data might not reach disk for up to
30 seconds. That's 30 seconds of vulnerability.


## 1.3 What a "Page" Actually Is

The kernel, the filesystem, and your storage engine all think in terms of
**pages** (or "blocks"). A page is a fixed-size chunk — typically 4096 bytes.

Why 4096?
- It's the smallest unit the kernel's page cache manages
- It matches most CPU architectures' virtual memory page size (x86, ARM)
- Most SSDs have a "page" size of 4KB-16KB internally
- Most HDDs have a sector size of 512B or 4KB, and 4KB aligns cleanly

When you read 1 byte from a file, the kernel actually reads a full 4KB page
from disk into the page cache, then copies your 1 byte from the cached page.
This is why sequential reads are fast (you pay for 4KB, get 4KB of prefetched
data) and random reads of scattered single bytes are slow (you pay for 4KB
each time but only use 1 byte).

**For your storage engine**: You'll read and write in 4KB page units because:
1. It aligns with the kernel's I/O granularity — no wasted reads
2. It aligns with the disk's I/O granularity — no read-modify-write cycles
3. It makes crash recovery reasoning simpler — a page write is atomic on
   most hardware (either the whole 4KB lands or none of it does)


## 1.4 Atomicity of Writes

This is the scariest part. When you write 4KB to disk, can you guarantee
it either fully lands or doesn't land at all?

**HDD**: A sector write (512B) is atomic. A 4KB write spanning 8 sectors
is NOT guaranteed atomic — you could get 6 sectors written and 2 not.
However, most modern HDDs with 4KB physical sectors make the 4KB write
atomic at the hardware level.

**SSD**: Most NVMe SSDs guarantee that a 4KB write aligned to a 4KB
boundary is atomic. Intel's documentation explicitly states this.
But it's not universal — some cheaper SSDs don't guarantee it.

**What this means for your storage engine**: NEVER assume a multi-page write
is atomic. If you're writing a B+ tree node split (modifying 3 pages),
you MUST use the WAL to make the multi-page operation recoverable.
Any single page might be a "torn write" — partially old, partially new.

This is why your WAL stores CRC checksums — to detect torn pages.


---

# Chapter 2: The System Calls

There are exactly 7 syscalls you need to master for a storage engine.
Let's go through each one in detail.


## 2.1 open() — Getting a File Descriptor

```c
int fd = open("database.db", O_RDWR | O_CREAT, 0644);
```

What this does:
1. Kernel looks up the path in the directory tree
2. Finds or creates the inode (the file's metadata structure)
3. Creates a "file description" in kernel memory (tracks current offset, flags)
4. Returns an integer — the file descriptor (fd) — which is just an index
   into the process's file descriptor table

The fd is a HANDLE, not the file itself. Multiple fds can point to the
same file. Each has its own offset position.

**Important flags for storage engines:**

| Flag | Effect | When to use |
|------|--------|-------------|
| O_RDWR | Read + write access | Always (you're a database) |
| O_CREAT | Create file if doesn't exist | On database creation |
| O_DIRECT | Bypass the page cache | Advanced: when YOU manage caching |
| O_DSYNC | Every write is immediately durable | Alternative to fsync() per write |
| O_SYNC | Like O_DSYNC but also updates metadata | Rarely needed |

**O_DIRECT deserves special attention**: It tells the kernel to bypass the
page cache entirely — reads go directly from disk to your buffer, writes go
directly from your buffer to disk. This means:
- You must allocate page-aligned buffers (use `posix_memalign`)
- You must read/write in multiples of the sector size (usually 512B or 4KB)
- You lose the kernel's readahead optimization
- But you get predictable I/O performance and no double-caching

Most storage engines start WITHOUT O_DIRECT (let the kernel cache help you)
and add it later as an optimization when they have their own buffer pool.
Your ferrisdb should start without it.


## 2.2 read() and write() — The Basics

```c
ssize_t bytes_read = read(fd, buffer, count);
ssize_t bytes_written = write(fd, buffer, count);
```

These read/write from the file descriptor's CURRENT OFFSET, then advance
the offset by the number of bytes transferred.

**The problem for storage engines**: The offset is shared state. If two
threads share a file descriptor and both call write(), their offsets
interleave unpredictably. Thread A seeks to page 5, Thread B seeks to
page 10, Thread A writes — but now the offset is at page 10's position!

This is why pread/pwrite exist.


## 2.3 pread() and pwrite() — Positioned I/O (★ CRITICAL ★)

```c
ssize_t pread(int fd, void *buf, size_t count, off_t offset);
ssize_t pwrite(int fd, const void *buf, size_t count, off_t offset);
```

These are like read/write BUT:
- You specify the offset explicitly as a parameter
- They do NOT change the file descriptor's current offset
- They are ATOMIC with respect to the offset — no race conditions
- They are THREAD-SAFE — multiple threads can pread/pwrite on the same fd

**This is the I/O primitive your storage engine uses for everything.**

Example: reading page 42 from a database file:
```c
// Page 42 starts at byte offset 42 * 4096 = 172032
char buf[4096];
ssize_t n = pread(fd, buf, 4096, 42 * 4096);
```

No seeking, no offset management, no locks needed for concurrent reads.

**In Rust**, pread/pwrite are available through:
- `std::os::unix::fs::FileExt::read_at()` and `write_at()`
- The `nix` crate: `nix::sys::uio::pread()` and `pwrite()`
- The `libc` crate for raw syscall access

The `read_at`/`write_at` API is the cleanest and what you should use.


## 2.4 fsync() and fdatasync() — Making Data Durable (★ MOST CRITICAL ★)

```c
int fsync(int fd);      // Flush data AND metadata
int fdatasync(int fd);  // Flush data only (faster)
```

This is THE most important syscall for any database. It forces the kernel
to flush all dirty pages for this file from the page cache to the physical
disk. It also asks the drive controller to flush its internal write cache.

**fsync() does NOT return until the data is on the physical medium.**

(Well, almost — some drive controllers lie about this. Enterprise SSDs
have battery-backed caches and are honest. Consumer SSDs might not be.
There's nothing you can do about lying hardware except use enterprise drives.)

**fsync vs fdatasync:**
- `fsync()` flushes data AND file metadata (size, timestamps, permissions)
- `fdatasync()` flushes data only, UNLESS the file size changed
- For a database that writes full pages (file size doesn't change after
  initial allocation), `fdatasync()` is sufficient and ~2x faster

**When to call fsync:**
- After flushing the WAL (to make committed transactions durable)
- After a checkpoint (to make checkpointed pages durable)
- After writing the meta page (to make structural changes durable)
- NEVER after every single page write (way too slow — this is why the
  WAL exists: batch many page modifications, then one fsync on the WAL)

**The cost of fsync:**

| Storage | fsync latency | fsyncs/second |
|---------|---------------|---------------|
| HDD 7200rpm | ~8ms | ~125 |
| SATA SSD | ~0.5ms | ~2,000 |
| NVMe SSD | ~0.05ms | ~20,000 |
| Battery-backed RAID | ~0.01ms | ~100,000 |

A single HDD fsync costs 8ms. If you fsync after every single write
operation, you're capped at 125 operations/second. This is why
GROUP COMMIT (batching many transactions into one fsync) is the single
most important performance optimization in database engineering.


## 2.5 lseek() — Moving the File Offset

```c
off_t lseek(int fd, off_t offset, int whence);
// whence: SEEK_SET (absolute), SEEK_CUR (relative), SEEK_END (from end)
```

You mostly won't use this directly because pread/pwrite don't need it.
But it's useful for:
- Finding the file size: `lseek(fd, 0, SEEK_END)` returns the file size
- Extending a file: `lseek(fd, new_size - 1, SEEK_SET)` then `write(fd, "", 1)`

**In Rust**: `file.seek(SeekFrom::End(0))` to get file size, or better:
`file.metadata()?.len()`.


## 2.6 ftruncate() — Setting File Size

```c
int ftruncate(int fd, off_t length);
```

Sets the file to exactly `length` bytes. If extending, new bytes are zero.
If shrinking, excess data is lost.

Useful for pre-allocating your database file:
```c
ftruncate(fd, 1024 * 4096);  // Pre-allocate 1024 pages = 4MB
```

Pre-allocation avoids filesystem fragmentation and metadata updates during
writes (because the file blocks are already allocated).

**In Rust**: `file.set_len(size)`.


## 2.7 mmap() — Memory-Mapped I/O (★ THE BIG DECISION ★)

```c
void *addr = mmap(NULL, length, PROT_READ | PROT_WRITE,
                   MAP_SHARED, fd, offset);
```

This is the most powerful and most dangerous I/O mechanism. It maps a
region of a file directly into your process's virtual address space.
After the mapping, reading memory at `addr` reads from the file, and
writing to `addr` writes to the file — no syscalls needed.

**How mmap works internally:**

```
Virtual Address Space                  Physical RAM
┌────────────────────┐                 ┌────────────────────┐
│ Stack              │                 │                    │
│ ...                │                 │                    │
│ mmap region ───────┼────────────────>│ Page cache page    │
│ (file pages)       │  Page table     │ (file data)        │
│ ...                │  mapping        │                    │
│ Heap               │                 │                    │
│ Code               │                 │                    │
└────────────────────┘                 └────────────────────┘
                                              │
                                              ▼
                                       ┌────────────────────┐
                                       │ Disk               │
                                       └────────────────────┘
```

When you access an mmap'd address:
1. CPU checks the page table → if the page is resident, done (fast path)
2. If not resident → PAGE FAULT (CPU trap to kernel)
3. Kernel reads the page from disk into the page cache
4. Kernel updates the page table to point to the cached page
5. CPU retries the access → succeeds

**Advantages of mmap for a storage engine:**
- Zero-copy: your program reads directly from the page cache, no memcpy
- Automatic caching: the kernel's page cache manages eviction for you
- Simplicity: just read/write memory instead of calling syscalls
- Potential performance: no syscall overhead for cached pages

**Disadvantages (and why many databases DON'T use mmap):**

1. **No control over eviction**: The kernel decides which pages to evict.
   Your B+ tree knows that internal nodes are hotter than leaf nodes, but
   the kernel doesn't. Your custom LRU-K replacer can't help.

2. **No control over write-back**: The kernel can flush dirty mmap'd
   pages to disk at any time, in any order. This BREAKS write-ahead
   logging! If the kernel writes a dirty data page before the WAL record
   that describes the modification is flushed, and then you crash, the
   database is corrupted.

3. **No error handling**: If a disk read fails during a page fault, your
   process gets SIGBUS (killed). With pread(), you get an error return
   that you can handle gracefully.

4. **TLB pressure**: Large mmap regions consume Translation Lookaside
   Buffer entries, which can slow down all memory access.

5. **fsync complexity**: You need `msync()` instead of `fsync()`, and
   the semantics are subtler.

**The recommendation for your storage engine:**

Start with pread/pwrite. They give you:
- Full control over which pages are in memory (your buffer pool)
- Full control over write ordering (WAL before data pages)
- Explicit error handling
- Portable behavior

Later, as an optimization, you can mmap the WAL for read-only recovery
scanning (reading the WAL sequentially is a perfect mmap use case).


---

# Chapter 3: File Descriptors In Depth

## 3.1 What a File Descriptor Actually Is

```
Process's File Descriptor Table (per-process)
┌─────┬──────────────────┐
│ fd  │ Pointer           │
├─────┼──────────────────┤
│  0  │ ──→ stdin desc    │
│  1  │ ──→ stdout desc   │
│  2  │ ──→ stderr desc   │
│  3  │ ──→ database.db   │──→ Open File Description (kernel)
│  4  │ ──→ database.wal  │     ├─ current offset: 172032
│  5  │ ──→ (closed)      │     ├─ flags: O_RDWR
└─────┴──────────────────┘     ├─ inode pointer ──→ Inode
                                │                    ├─ file size
                                │                    ├─ block map
                                │                    └─ permissions
                                └─ page cache pointer
```

Key facts:
- fd is just an integer index (0, 1, 2, 3, ...)
- The kernel allocates the lowest available fd number
- Two processes can have fd 3 pointing to different files
- Two fds in the same process can point to the same file (via dup())
- fork() copies all fds — child inherits parent's open files

## 3.2 The Relationship Between fd, Inode, and Data

```
fd ──→ Open File Description ──→ Inode ──→ Block Map ──→ Disk Blocks
       (kernel, ephemeral)       (persistent)
       - offset                  - size
       - flags                   - timestamps
       - refcount                - permissions
                                 - block pointers
```

The inode is the REAL file. The filename is just a directory entry pointing
to an inode number. Two filenames can point to the same inode (hard links).
When you delete a file, you remove a directory entry — the inode is only
freed when its link count reaches 0 AND no process has it open.

**For your storage engine**: If your process has the database file open and
another process deletes it, YOUR process can still read/write to it through
its fd. The data only disappears when you close the fd. This is why
`lsof` (list open files) is useful for debugging.

## 3.3 Directory Entry and fsync on Directories

Here's a subtlety that trips up many database developers:

When you create a new file, two things happen:
1. The file's data is written (new inode, new blocks)
2. The parent directory is modified (new directory entry added)

If you `fsync()` the file but NOT the directory, and the system crashes,
the file's data is on disk but the directory entry might not be. The file
exists as an orphan inode — it has data but no name. The filesystem's
journal or fsck might recover it, or might not.

**The safe pattern for creating a new database file:**
```rust
// 1. Create and write the file
let file = File::create("database.db")?;
file.write_all(&initial_data)?;
file.sync_all()?;  // fsync the file

// 2. Also fsync the parent directory
let dir = File::open(".")?;  // Open the directory
dir.sync_all()?;              // fsync the directory entry
```

You only need to do this at file creation time, not on every write.


---

# Chapter 4: Understanding I/O Patterns

## 4.1 Sequential vs Random I/O

The performance difference between sequential and random I/O is dramatic:

**HDD (spinning disk):**

```
Sequential read:  150 MB/s
Random read:      0.5 MB/s (300x slower!)

Why: The disk head must physically MOVE to a new track for each random read.
Each seek takes ~8ms. In 1 second you can do 125 seeks × 4KB = 0.5 MB.
```

**SSD (NAND flash):**

```
Sequential read:  3,500 MB/s (NVMe)
Random read:      500 MB/s  (7x slower)

Why: No moving parts, but the flash controller can batch and parallelize
sequential reads across multiple NAND channels. Random reads don't batch well.
```

**What this means for your storage engine:**
- The WAL is append-only → sequential writes → FAST on any storage
- B+ tree traversals are ~random reads → this is where the buffer pool matters
- Full table scans are sequential → fast even without caching
- Index range scans follow leaf page linked lists → semi-sequential → fast

## 4.2 Read-Ahead (Prefetching)

When you read sequentially, the kernel detects the pattern and starts
reading AHEAD of your requests. By the time your program asks for page N+1,
it's already in the page cache.

You can control this:
```c
posix_fadvise(fd, offset, length, POSIX_FADV_SEQUENTIAL);  // Hint: sequential
posix_fadvise(fd, offset, length, POSIX_FADV_RANDOM);      // Hint: random
posix_fadvise(fd, offset, length, POSIX_FADV_DONTNEED);    // Evict from cache
```

**For your storage engine:**
- Use `FADV_SEQUENTIAL` when scanning the WAL for recovery
- Use `FADV_RANDOM` on the main database file (B+ tree access is random)
- Use `FADV_DONTNEED` after a checkpoint to free cache memory


## 4.3 Write Amplification

When you write 1 byte to the middle of a page:
1. Kernel reads the full 4KB page from disk (if not cached)
2. Modifies 1 byte in the cached copy
3. Eventually writes the full 4KB page back to disk

You wrote 1 byte, but 4KB was read and 4KB was written. That's 8192x
amplification.

**For SSDs, it's even worse**: SSDs can't overwrite in place. They must
erase a whole "erase block" (typically 256KB-4MB) before writing. Writing
one 4KB page might trigger:
1. Read the entire erase block (256KB)
2. Erase the block (slow)
3. Write back the block with your 1 byte changed (256KB)

This is "write amplification" and it's why SSDs wear out.

**Your storage engine avoids this by**:
- Always writing full 4KB pages (no partial page writes)
- Using the WAL for sequential writes (SSD-friendly)
- Batching modifications in the buffer pool before flushing


---

# Chapter 5: The fsync Guarantee Chain

This is the most important chapter. Understanding this chain is what separates
a toy database from a real one.

## 5.1 What Exactly Does fsync Guarantee?

After `fsync(fd)` returns 0 (success):

1. All data written to `fd` via `write()`/`pwrite()` is on stable storage
2. All metadata necessary to retrieve that data is on stable storage
3. The drive's internal write cache has been flushed

"Stable storage" means: survives power loss, kernel panic, process crash.

**What fsync does NOT guarantee:**
- Data written to OTHER file descriptors (even if they're the same file!)
  → Actually, on Linux, fsync on any fd to a file flushes all data for
  that inode. But POSIX doesn't guarantee this. Be explicit.
- Writes that happened after the fsync call started
- That the filesystem's journal is flushed (it is on ext4, might not be on others)

## 5.2 The Write-Ahead Logging Protocol

This is the core protocol of every ACID database:

```
RULE 1 (WAL): Before a dirty page can be flushed to disk,
              all WAL records for that page must be flushed first.

RULE 2 (COMMIT): A transaction is committed if and only if
                  its COMMIT record is in the flushed WAL.

RULE 3 (RECOVERY): On startup, replay the WAL:
                    - REDO all changes from committed transactions
                    - UNDO all changes from uncommitted transactions
```

In practice:

```
Time ──────────────────────────────────────────────────────────►

Transaction 1:
  BEGIN ──→ UPDATE page 5 ──→ UPDATE page 8 ──→ COMMIT
    │              │                │               │
    ▼              ▼                ▼               ▼
   WAL:     [BEGIN T1]    [UPDATE T1,P5]  [UPDATE T1,P8]  [COMMIT T1]
                                                               │
                                                          fsync(WAL) ←── THIS is the commit point
                                                               │
                                                          Now safe to tell user "committed"
                                                               │
                                                          Pages 5, 8 can be flushed
                                                          to disk lazily (checkpoint)
```

If we crash BEFORE `fsync(WAL)`: Transaction 1 never committed. On recovery,
we see no COMMIT record, so we undo any partial changes. No data corruption.

If we crash AFTER `fsync(WAL)`: Transaction 1 committed. On recovery, we
redo the changes to pages 5 and 8 from the WAL records. Data is preserved.

## 5.3 The Ordering Problem

Consider this dangerous sequence:

```
1. write(WAL, "UPDATE page 5: old=X, new=Y")
2. write(DATA_FILE, page 5 with new value Y)  ← WRONG! Haven't fsynced WAL yet!
3. fsync(WAL)
4. *** CRASH between step 2 and step 3 ***
```

After the crash:
- The WAL doesn't have the UPDATE record (fsync didn't finish)
- But page 5 on disk has the NEW value Y
- Recovery can't undo the change because there's no WAL record for it
- DATABASE CORRUPTED

**The correct sequence:**

```
1. write(WAL, "UPDATE page 5: old=X, new=Y")
2. fsync(WAL)                                  ← WAL durable FIRST
3. THEN write(DATA_FILE, page 5 with new value Y)  ← NOW safe
4. fsync(DATA_FILE)                            ← Optional, can defer to checkpoint
```

This ordering is enforced by your buffer pool: it checks that a page's
LSN is ≤ the WAL's flushed LSN before writing the page to disk.

## 5.4 Group Commit — The Key Performance Optimization

Naive approach: one fsync per transaction commit.
Result: On HDD, ~125 transactions/second. Terrible.

Group commit: batch multiple transactions' COMMIT records, one fsync.

```
Time ──────────────────────────────────────────────────────────►

T1: BEGIN ... UPDATE ... COMMIT ──┐
T2: BEGIN ... UPDATE ... COMMIT ──┤── All in WAL buffer
T3: BEGIN ... UPDATE ... COMMIT ──┘
                                   │
                                   ▼
                              fsync(WAL)  ← ONE fsync for 3 transactions!
                                   │
                                   ▼
                              All 3 transactions are now committed
```

Result: Same 125 fsyncs/second on HDD, but each fsync commits ~10-100
transactions. Effective throughput: 1,250 - 12,500 transactions/second.

This is the SINGLE most important optimization in your WAL implementation.


---

# Chapter 6: mmap Deep Dive

## 6.1 How mmap Actually Works

When you call mmap(), NO data is read from disk. The kernel just sets up
the virtual memory mapping:

```
mmap(NULL, 4096 * 100, PROT_READ, MAP_SHARED, fd, 0);

Result: 100 virtual pages mapped, ALL marked "not present" in the page table.
```

When you first access byte 0:
1. CPU → page fault (page not present)
2. Kernel → reads page from disk into page cache
3. Kernel → updates page table (virtual page → physical page)
4. CPU → retries access → succeeds
5. Future accesses to bytes 0-4095 → NO page fault (already mapped)

When you access byte 4096 (next page):
1. CPU → page fault again (different virtual page)
2. Same process as above

This is called "demand paging" — pages are loaded on demand, not upfront.

## 6.2 mmap and the Page Cache

mmap'd pages and read/write pages share the SAME page cache:

```
┌──────────────────────────────┐
│     Kernel Page Cache         │
│                              │
│  Page 0: ←── mmap access     │  ← SAME physical page
│          ←── pread() also    │
│                              │
│  Page 1: ←── pread() first   │
│          ←── mmap sees it    │
│                              │
└──────────────────────────────┘
```

This means: if you mmap a file AND also pread from it, they see the
same data. There's no consistency problem between the two methods
(on Linux with MAP_SHARED — other OSes may differ).

## 6.3 msync() — mmap's fsync

For mmap'd regions, you use `msync()` instead of `fsync()`:

```c
msync(addr, length, MS_SYNC);   // Synchronous — blocks until durable
msync(addr, length, MS_ASYNC);  // Asynchronous — just schedules flush
```

**The subtlety**: msync only flushes the pages in the specified range.
If you've modified pages all over the file, you need to msync each
modified range (or msync the entire mapped region, which is slow).

## 6.4 When to Use mmap in a Storage Engine

Use mmap for:
- Read-only access to the WAL during recovery (sequential scan)
- Read-only access to immutable files (old WAL segments)
- Memory-mapped scratch space for sort operations

Do NOT use mmap for:
- The primary data file (you need write ordering control)
- The active WAL (you need precise fsync control)
- Anything where you need to control eviction policy


---

# Chapter 7: Putting It All Together

## 7.1 Your Storage Engine's I/O Pattern

```
Database file (data pages):
- pread()  to load pages into buffer pool
- pwrite() to flush dirty pages from buffer pool
- fdatasync() at checkpoint time
- posix_fadvise(FADV_RANDOM) — access pattern is random

WAL file:
- write() (append only, sequential)
- fsync() at commit time (or group commit)
- posix_fadvise(FADV_SEQUENTIAL) — access pattern is sequential

Recovery:
- mmap() the WAL read-only for fast sequential scan
- pread() data pages to verify/redo
- fsync() the data file after recovery completes
```

## 7.2 The Checklist for Correct Durability

Before you ship your storage engine, verify:

□ WAL records are fsynced BEFORE data pages are written to disk
□ COMMIT record fsync is the actual commit point
□ CRC on every WAL record detects torn writes
□ Recovery replays the WAL correctly (redo committed, undo uncommitted)
□ New file creation also fsyncs the parent directory
□ Buffer pool checks page LSN ≤ flushed WAL LSN before writing
□ Group commit batches multiple transactions per fsync
□ Pre-allocate the database file to avoid metadata updates during writes
□ Use fdatasync() instead of fsync() where metadata flush isn't needed
□ Handle fsync() errors correctly (on Linux, a failed fsync means
  the dirty pages are DROPPED from the page cache — re-writing and
  re-syncing won't help, you must abort the transaction)

---


# Syscall Cheat Sheet for Storage Engines

## Quick Reference: C → Rust Mapping

| C Syscall | Rust std | Rust (nix crate) | What it does |
|-----------|----------|-------------------|--------------|
| `open()` | `File::open()` / `OpenOptions` | `nix::fcntl::open()` | Get a file descriptor |
| `read()` | `file.read()` | `nix::unistd::read()` | Read at current offset |
| `write()` | `file.write()` | `nix::unistd::write()` | Write at current offset |
| `pread()` | `file.read_at()` ★ | `nix::sys::uio::pread()` | Read at specific offset |
| `pwrite()` | `file.write_at()` ★ | `nix::sys::uio::pwrite()` | Write at specific offset |
| `lseek()` | `file.seek()` | `nix::unistd::lseek()` | Move current offset |
| `fsync()` | `file.sync_all()` ★ | `nix::unistd::fsync()` | Flush data + metadata |
| `fdatasync()` | `file.sync_data()` ★ | `nix::unistd::fdatasync()` | Flush data only |
| `ftruncate()` | `file.set_len()` | `nix::unistd::ftruncate()` | Set file size |
| `mmap()` | use `memmap2` crate | `nix::sys::mman::mmap()` | Map file to memory |
| `munmap()` | (auto on drop) | `nix::sys::mman::munmap()` | Unmap file |
| `msync()` | — | `nix::sys::mman::msync()` | Flush mmap'd pages |
| `fadvise()` | — | `nix::fcntl::posix_fadvise()` | Hint access pattern |
| `fallocate()` | — | `nix::fcntl::fallocate()` | Pre-allocate space |

★ = Use these in your storage engine

## The 5 Syscalls You'll Use 99% of the Time

```rust
use std::os::unix::fs::FileExt;  // Provides read_at, write_at

// 1. Positioned read (pread) — thread-safe, no seeking
let mut buf = [0u8; 4096];
file.read_at(&mut buf, page_id as u64 * 4096)?;

// 2. Positioned write (pwrite) — thread-safe, no seeking
file.write_at(&data, page_id as u64 * 4096)?;

// 3. fsync — make writes durable (data + metadata)
file.sync_all()?;  // Use for WAL (file grows)

// 4. fdatasync — make writes durable (data only, faster)
file.sync_data()?;  // Use for data file (pre-allocated, fixed size)

// 5. Set file size (ftruncate) — pre-allocate pages
file.set_len(num_pages as u64 * 4096)?;
```

## Decision Matrix: When to Use What

| Situation | Syscall | Why |
|-----------|---------|-----|
| Read a B+ tree node | `read_at()` | Random access, need offset control |
| Write a dirty page to disk | `write_at()` | Need exact offset, thread-safe |
| Commit a transaction | `sync_all()` on WAL | Must be durable before confirming |
| Checkpoint data pages | `sync_data()` on data file | Data only, file size unchanged |
| Scan WAL for recovery | `mmap()` read-only | Sequential scan, zero-copy |
| Create new database | `set_len()` + `sync_all()` | Pre-allocate + sync directory |
| Tell kernel access pattern | `posix_fadvise()` | Optimize readahead/caching |

## The Golden Rules

1. **NEVER** assume `write()` reaches disk. Always `sync_all()`/`sync_data()`.
2. **ALWAYS** flush WAL before flushing data pages.
3. **ALWAYS** use `read_at()`/`write_at()` (not `read()`/`write()`) for data pages.
4. **ALWAYS** pre-allocate your data file to avoid metadata updates on every write.
5. **NEVER** use `mmap()` for write paths — you lose control over flush ordering.
6. **ALWAYS** CRC your WAL records to detect torn writes.
7. **BATCH** multiple commits into one `sync_all()` (group commit).
