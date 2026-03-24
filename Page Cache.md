# Page Cache, Block I/O Layer & Device Drivers

## The Complete Guide — How Data Travels From Kernel Memory to Physical Disk

---

# Part A: The Page Cache

## Chapter 1: What the Page Cache Is

The page cache is a region of RAM managed by the kernel that holds copies
of file data. When you call `write()`, your data lands HERE — not on disk.
When you call `read()`, the kernel checks HERE first before going to disk.

**Analogy**: Imagine a restaurant kitchen. The refrigerator is the disk
(permanent, large, slow to access). The countertop is the page cache
(temporary, limited space, instant access). When you need an ingredient,
you check the countertop first. If it's there, great — no need to open
the fridge. If not, you grab it from the fridge and put it on the
countertop for next time.

```
Application calls write("Hello"):

  "Hello" → copy to page cache (RAM)     ← THIS is what write() does
            Done. write() returns.

  ...seconds pass...

  Kernel background thread:
    "I see dirty pages in the cache"
    → writes them to disk eventually       ← THIS happens LATER
```

**The terrifying truth**: Between your `write()` returning and the kernel
flushing to disk, your data exists ONLY in volatile RAM. Power cut = data gone.
This gap is where data loss happens. This is why `fsync()` exists.


## Chapter 2: How the Page Cache Is Organized

The page cache is indexed by **(file inode, offset)**. Every file has its
own set of cached pages.

```
Page Cache (simplified view):

  ┌─────────────────────────────────────────────────────────┐
  │ File: /data/database.db (inode 4521)                    │
  │                                                         │
  │   Page offset 0     → [4KB of data] dirty=no            │
  │   Page offset 4096  → [4KB of data] dirty=yes ★         │
  │   Page offset 8192  → [4KB of data] dirty=no            │
  │   (offset 12288 not cached — would need disk read)      │
  │   Page offset 16384 → [4KB of data] dirty=yes ★         │
  │                                                         │
  │ File: /var/log/syslog (inode 7890)                      │
  │                                                         │
  │   Page offset 0     → [4KB of data] dirty=no            │
  │   Page offset 4096  → [4KB of data] dirty=yes ★         │
  └─────────────────────────────────────────────────────────┘

  ★ = dirty pages: modified in memory but NOT yet written to disk
```

The kernel uses a data structure called the **address_space** (one per inode)
to manage each file's cached pages. Inside it, a **radix tree** (or in
newer kernels, an **XArray**) maps file offsets to physical memory pages.

```c
// Kernel structures (simplified)

struct address_space {
    struct inode       *host;      // Which file this cache belongs to
    struct xarray      i_pages;    // Radix tree: offset → page
    unsigned long      nrpages;    // Number of cached pages
    struct address_space_operations *a_ops;  // read/write operations
};
```

**Why a radix tree?** Because files can be terabytes long, but only a tiny
fraction of their pages are actually cached. A radix tree is sparse — it
only allocates nodes for offsets that exist in the cache, unlike an array
that would need an entry for every possible offset.


## Chapter 3: The Read Path — What Happens When You Read

```
Your code: read(fd, buf, 4096)   // Read 4096 bytes

┌───────────────────────────────────────────────────────┐
│ Step 1: VFS calls the filesystem's read_iter()        │
│                                                       │
│ Step 2: Filesystem calculates which page offset       │
│         file offset 8192 → page index 2               │
│                                                       │
│ Step 3: Check the page cache                          │
│         find_get_page(inode->i_mapping, 2)            │
│                                                       │
│         ┌── FOUND in cache (cache hit) ──┐            │
│         │  Copy data from cached page     │            │
│         │  to user buffer. Done!          │            │
│         │  No disk I/O needed.            │            │
│         │  Cost: ~200 nanoseconds         │            │
│         └─────────────────────────────────┘            │
│                                                       │
│         ┌── NOT found (cache miss) ──────┐            │
│         │  Allocate a new page in cache   │            │
│         │  Call filesystem's readpage()   │            │
│         │  → submits I/O to block layer   │            │
│         │  → block layer reads from disk  │            │
│         │  → data arrives in the new page │            │
│         │  Copy to user buffer. Done.     │            │
│         │  Cost: ~100,000+ nanoseconds    │            │
│         └─────────────────────────────────┘            │
└───────────────────────────────────────────────────────┘
```

**The difference: 200 ns vs 100,000 ns.** A cache hit is 500x faster
than a cache miss. This is why the page cache is so important — and
why your storage engine has its own buffer pool (to have even MORE
control over what stays cached).


## Chapter 4: The Write Path — What Happens When You Write

```
Your code: write(fd, "Hello World", 11)

┌───────────────────────────────────────────────────────┐
│ Step 1: VFS calls the filesystem's write_iter()       │
│                                                       │
│ Step 2: Filesystem calls write_begin()                │
│         → Find or create the target page in the cache │
│         → If the page doesn't exist yet, may need to  │
│           read the existing page from disk first       │
│           (because we might be overwriting only PART   │
│           of a 4KB page)                               │
│                                                       │
│ Step 3: Copy "Hello World" into the cached page       │
│         memcpy(page + offset, "Hello World", 11)      │
│                                                       │
│ Step 4: Filesystem calls write_end()                  │
│         → Mark the page as DIRTY                       │
│         → Update the inode's modification time         │
│         → Return to user. write() is now complete.     │
│                                                       │
│         ★ DATA IS IN RAM ONLY. NOT ON DISK YET. ★     │
│                                                       │
│ Step 5: (LATER) Kernel writeback thread notices        │
│         dirty pages and flushes them to disk.          │
│         Or your program calls fsync() to force it.     │
└───────────────────────────────────────────────────────┘
```

**The key insight**: `write()` returns after step 4. Your data is in the
page cache but NOT on disk. This is why `write()` is fast (~hundreds of
nanoseconds) — it's just a memory copy. The slow disk write happens later.

### The Dirty Page Problem

A dirty page is a page cache entry that has been modified in memory but
not yet written to disk. The kernel tracks dirty pages carefully:

```
Page states:

  CLEAN:  cache matches disk. Can be evicted freely.
          (Just throw away the cached copy — disk has the same data)

  DIRTY:  cache is DIFFERENT from disk. MUST write before evicting.
          (If we evict without writing, the modifications are LOST)

  Transition: CLEAN → DIRTY  (when write() modifies a page)
  Transition: DIRTY → CLEAN  (when writeback writes it to disk)
```


## Chapter 5: Writeback — How Dirty Pages Reach Disk

The kernel has background threads that periodically flush dirty pages.
This process is called **writeback**.

### Who Triggers Writeback?

```
1. BACKGROUND WRITEBACK (automatic, periodic)
   
   The kernel thread "kworker/writeback" wakes up every 5 seconds
   (configurable via /proc/sys/vm/dirty_writeback_centisecs = 500)
   
   It looks for pages that have been dirty for more than 30 seconds
   (configurable via /proc/sys/vm/dirty_expire_centisecs = 3000)
   
   These "expired" dirty pages get written to disk.


2. THRESHOLD WRITEBACK (when too much memory is dirty)
   
   /proc/sys/vm/dirty_background_ratio = 10 (default: 10% of RAM)
   → When dirty pages exceed 10% of RAM, background writeback starts.
   
   /proc/sys/vm/dirty_ratio = 20 (default: 20% of RAM)
   → When dirty pages exceed 20% of RAM, ALL writing processes are
     BLOCKED until dirty pages drop below the threshold.
     This is called "throttling" and it makes write() suddenly slow.

   Example with 16GB RAM:
     dirty_background_ratio = 10% → writeback starts at 1.6GB dirty
     dirty_ratio = 20%            → processes blocked at 3.2GB dirty


3. EXPLICIT SYNC (your program asks for it)
   
   fsync(fd)      → Flush all dirty pages for this file, wait for completion
   fdatasync(fd)  → Same but skip metadata unless file size changed
   sync()         → Flush ALL dirty pages for ALL files (the nuclear option)
   
   These are the ONLY ways to GUARANTEE data is on disk.
```

### The Writeback Flow

```
Writeback thread wakes up:

  For each dirty inode:
    1. Collect all dirty pages for this inode
    2. Sort them by file offset (for sequential I/O)
    3. Call the filesystem's writepages() operation
       → Filesystem maps file offsets to disk block numbers
       → Filesystem creates I/O requests (bio structures)
       → Submits them to the Block I/O Layer
    4. Wait for I/O completion
    5. Mark pages as CLEAN

  The same flow happens when you call fsync(), but:
    - Only for the specific file you're syncing
    - fsync() BLOCKS until ALL I/O is complete
    - fsync() also flushes the drive's hardware cache
```


## Chapter 6: Page Cache Eviction — When RAM Runs Low

The page cache can use ALL available RAM (after programs and the kernel take
what they need). When RAM runs low, the kernel must evict cache pages.

```
Memory usage:

  ┌──────────────────────────────────────────┐
  │  Program memory (can't evict — in use)    │ 
  │  Kernel memory (can't evict — needed)     │
  │  Page cache (CAN evict — it's a cache!)   │ ★
  │  Free memory                              │
  └──────────────────────────────────────────┘
  
  When free memory is low:
    1. Evict CLEAN cache pages first (free, no I/O needed)
    2. Flush DIRTY cache pages to disk, then evict them
    3. If still not enough, swap out program memory to disk
```

The kernel uses a **two-list LRU** (active list + inactive list) to decide
which pages to evict:

```
Active list:   Pages accessed recently and more than once (HOT)
Inactive list: Pages accessed once or not recently (COLD)

New page → enters inactive list
Accessed again → promoted to active list
Active list too full → demoted to inactive list
Evict from → bottom of inactive list

This is similar to your LRU-K! The kernel uses K=2 effectively:
pages must be accessed twice to stay on the active list.
```

**You can see the page cache in action:**

```bash
$ free -h
              total   used   free   shared  buff/cache  available
Mem:           16G    4.2G   1.1G    320M       10.7G      11.2G

# 10.7 GB is being used by the page cache (buff/cache)!
# "available" = free + reclaimable cache = what programs can actually use
```

```bash
$ cat /proc/meminfo | grep -E "Cached|Dirty|Writeback"
Cached:         10485760 kB    # 10 GB in page cache
Dirty:              4096 kB    # 4 KB waiting to be written to disk
Writeback:             0 kB    # 0 KB currently being written to disk
```


## Chapter 7: Page Cache and Your Storage Engine

```
The Linux I/O stack with page cache:

  Your program: write(fd, data, 4096)
       │
       ▼
  ┌─────────────────────────────┐
  │ Page Cache                   │  Your data lands HERE (RAM)
  │                              │  write() returns immediately
  │ Dirty page: file X, offset Y│
  └──────────────┬──────────────┘
                 │
        (later, via writeback or fsync)
                 │
                 ▼
  ┌─────────────────────────────┐
  │ Block I/O Layer              │  (covered in Part B)
  └─────────────────────────────┘
                 │
                 ▼
  ┌─────────────────────────────┐
  │ Device Driver                │  (covered in Part C)
  └─────────────────────────────┘
                 │
                 ▼
            Physical Disk
```

**Why your buffer pool exists alongside the page cache:**

You might wonder: the kernel already has a page cache, why do you build
ANOTHER cache (the buffer pool) in your storage engine?

```
Page Cache (kernel):
  ✗ Generic LRU eviction — doesn't know B+ tree nodes are hotter than leaf data
  ✗ Kernel decides when to flush dirty pages — you can't control write ordering
  ✗ No concept of WAL ordering — might flush a data page before the WAL
  ✗ No pin/unpin — can evict any page at any time

Your Buffer Pool:
  ✓ LRU-K eviction — you know internal nodes are hot, scanned leaves are cold
  ✓ YOU decide when to flush — WAL pages before data pages, always
  ✓ Pin/unpin — guarantees a page stays in memory while you're using it
  ✓ Dirty page tracking tied to your WAL's LSN
```

In fact, many production databases open files with `O_DIRECT` to BYPASS
the page cache entirely, because the database's own buffer pool is smarter
for its specific access patterns. Without `O_DIRECT`, data is cached
TWICE — once in your buffer pool, once in the page cache — wasting RAM.


---

# Part B: The Block I/O Layer

## Chapter 8: What the Block I/O Layer Does

The page cache knows about files and offsets. But the disk doesn't know
what a "file" is — it only knows about **block numbers** (numbered 4KB
chunks on the physical device).

The block I/O layer sits between the filesystem and the device driver.
Its job is to translate file-level I/O requests into optimized block-level
I/O requests.

```
Filesystem says:  "Write page at file offset 8192 of inode 4521"
                        │
                        ▼
Block I/O layer:  "That maps to disk block 50432. I'll create a
                   request to write 4KB at block 50432."
                        │
                   "Oh wait, there's also a write to block 50433
                    and 50434 queued up. Let me MERGE them into
                    one big 12KB write — that's way faster."
                        │
                   "And there's a read of block 10000 queued too.
                    The disk head is currently near block 50000,
                    so I'll do the writes first (they're close),
                    then seek to block 10000 for the read."
                        │
                        ▼
Device driver:    "Here's a single optimized I/O command"
```

The block I/O layer does three critical things:
1. **Translating**: file offsets → disk block numbers
2. **Merging**: combining adjacent requests into larger ones
3. **Scheduling**: reordering requests for optimal disk performance


## Chapter 9: The bio Structure — I/O Building Block

The fundamental data structure in the block I/O layer is the **bio**
(block I/O). It describes a single I/O operation:

```c
struct bio {
    struct block_device *bi_bdev;    // Which disk device
    sector_t            bi_sector;   // Starting sector on disk
    unsigned int        bi_size;     // Total bytes to transfer
    unsigned int        bi_opf;      // READ or WRITE + flags
    struct bio_vec      *bi_io_vec;  // Array of memory segments
    // ...
};

struct bio_vec {
    struct page  *bv_page;    // Which memory page
    unsigned int  bv_len;     // How many bytes from this page
    unsigned int  bv_offset;  // Offset within the page
};
```

**Analogy**: A bio is like a shipping order. It says:
- "Ship TO warehouse sector 50432" (bi_sector)
- "The cargo is in these boxes:" (bi_io_vec)
  - "Box 1: page at address 0x1234, 4096 bytes"
  - "Box 2: page at address 0x5678, 4096 bytes"
- "This is a WRITE operation" (bi_opf)

A single bio can reference multiple memory pages (scatter-gather I/O),
allowing the kernel to write several pages to contiguous disk blocks
in one I/O operation.


## Chapter 10: Request Merging — Combining Adjacent I/O

When multiple bios target adjacent disk blocks, the block layer merges
them into a single request:

```
Three separate bios arrive:
  bio 1: WRITE block 100, 4KB
  bio 2: WRITE block 101, 4KB  
  bio 3: WRITE block 102, 4KB

Block layer merges them:
  merged request: WRITE blocks 100-102, 12KB (one command to the disk!)

Without merging: 3 commands → 3 disk head seeks → slow
With merging:    1 command  → 1 disk head seek  → 3x faster
```

**Front merge**: new bio is placed BEFORE an existing request
**Back merge**: new bio is placed AFTER an existing request (most common)

```
Existing request: blocks 100-102
New bio: block 103

→ BACK MERGE: request becomes blocks 100-103

Existing request: blocks 100-102  
New bio: block 99

→ FRONT MERGE: request becomes blocks 99-102
```

**For your storage engine**: This is why sequential writes (like WAL appends)
are faster than random writes (like flushing B+ tree pages). The WAL writes
blocks 1000, 1001, 1002 in sequence — the block layer merges them into one
big write. B+ tree flushes might write blocks 5, 892, 34, 7651 — no merging
possible, each becomes a separate disk command.


## Chapter 11: I/O Schedulers — Ordering Requests

The I/O scheduler decides the ORDER in which merged requests are sent to
the disk. Different schedulers optimize for different goals.

### For HDDs (spinning disks): Elevator Algorithms

An HDD has a physical read/write head that must MOVE to the right track.
Seeking is slow (~8ms). The scheduler minimizes total seek distance.

```
Pending requests (block numbers): 10, 500, 12, 498, 15, 502

Naive order (FIFO): 10 → 500 → 12 → 498 → 15 → 502
  Head movement: 490 + 488 + 486 + 483 + 487 = 2434 blocks of seeking

Elevator order (C-LOOK): 10 → 12 → 15 → 498 → 500 → 502
  Head movement: 2 + 3 + 483 + 2 + 2 = 492 blocks of seeking
  5x LESS seeking!
```

**Analogy**: An elevator doesn't go 1→8→2→7→3. It goes 1→2→3→7→8.
It sweeps in one direction, serving all requests along the way, then
reverses. The I/O scheduler does the same with disk head movement.

### For SSDs (no moving parts): Simple Queuing

SSDs have no seek penalty — any block can be read in the same time.
The scheduler for SSDs is much simpler:

```
mq-deadline (Multi-Queue Deadline):
  - Each request has a deadline (reads: 500ms, writes: 5000ms)
  - Requests are served roughly in order
  - But expired deadlines get priority (no request starved forever)
  - Reads prioritized over writes (reads are often blocking a program)

none (No-op):
  - Just pass requests through in order
  - Let the SSD's internal controller do the optimization
  - Modern NVMe SSDs have very smart internal schedulers
```

**You can see which scheduler your system uses:**

```bash
$ cat /sys/block/sda/queue/scheduler
[mq-deadline] kyber bfq none

# [mq-deadline] is currently active
# You can change it:
$ echo "none" > /sys/block/sda/queue/scheduler
```

### For NVMe SSDs: Multi-Queue

Modern NVMe SSDs support multiple hardware queues (typically 64 or more).
The kernel creates one software queue per CPU core and maps them to
hardware queues. This means multiple CPU cores can submit I/O
simultaneously without contending for a single lock.

```
Old model (SATA HDD):
  All CPUs → one queue → one disk command at a time

New model (NVMe SSD):
  CPU 0 → queue 0 ──→ NVMe hardware queue 0
  CPU 1 → queue 1 ──→ NVMe hardware queue 1
  CPU 2 → queue 2 ──→ NVMe hardware queue 2
  CPU 3 → queue 3 ──→ NVMe hardware queue 3
  
  4 commands in flight simultaneously!
```

**For your storage engine**: This is why NVMe SSDs can handle thousands of
concurrent `pread()` and `pwrite()` calls — the multi-queue architecture
parallelizes at the hardware level. Your buffer pool's concurrent page
fetches benefit directly from this.


## Chapter 12: Pluggable I/O Schedulers

Linux uses a pluggable scheduler architecture, similar to how the VFS
uses pluggable filesystem drivers:

```c
struct elevator_type {
    const char *elevator_name;          // "mq-deadline", "bfq", "kyber"
    
    struct elevator_mq_ops {
        // Called when a new request arrives
        void (*insert_requests)(struct blk_mq_hw_ctx *,
                                struct list_head *, bool);
        
        // Called to get the next request to dispatch to the driver
        struct request *(*dispatch_request)(struct blk_mq_hw_ctx *);
        
        // Merge check
        bool (*allow_merge)(struct request *, struct bio *);
        
        // ...
    } ops;
};
```

**This is the same Strategy Pattern** you saw in the VFS (filesystem
operations tables) and in your storage engine (pluggable eviction policies
in the buffer pool). The kernel uses it everywhere.


## Chapter 13: The Request Queue

Each block device has a request queue where I/O requests wait to be
dispatched to the device driver:

```
Application writes → page cache dirty page
                         │
                    (writeback)
                         │
                         ▼
                 ┌────────────────────────┐
                 │ bio (single I/O op)     │
                 └──────────┬─────────────┘
                            │
                    (try to merge)
                            │
                            ▼
                 ┌────────────────────────┐
                 │ Request Queue           │
                 │                        │
                 │ req 1: READ  blk 100   │
                 │ req 2: WRITE blk 500-503│ ← merged from 4 bios
                 │ req 3: READ  blk 102   │
                 │ req 4: WRITE blk 800   │
                 └──────────┬─────────────┘
                            │
                    (I/O scheduler reorders)
                            │
                            ▼
                 ┌────────────────────────┐
                 │ Dispatch to driver:     │
                 │ req 1: READ  blk 100   │ ← close together, do first
                 │ req 3: READ  blk 102   │
                 │ req 2: WRITE blk 500-503│ ← then sweep here
                 │ req 4: WRITE blk 800   │ ← then here
                 └────────────────────────┘
```


---

# Part C: Device Drivers

## Chapter 14: What a Device Driver Does

The device driver is the translator between the kernel's generic block
I/O requests and the specific hardware protocol of the disk.

**Analogy**: The block I/O layer writes a letter in English: "Please read
block 50432." The device driver translates it into the specific language
the disk speaks — SATA commands for a SATA drive, NVMe commands for an
NVMe drive, SCSI commands for a SAS drive. The disk doesn't understand
English (kernel requests) — it only understands its own protocol.

```
Block I/O layer:
  "Read 4KB from block 50432"         ← generic request

SATA device driver translates to:
  Command: READ DMA EXT
  LBA: 50432
  Sector count: 8                      ← 8 × 512 = 4096 bytes
  → Send via AHCI register interface   ← SATA-specific

NVMe device driver translates to:
  Opcode: 0x02 (NVMe Read)
  NSID: 1
  SLBA: 50432
  NLB: 7                               ← (count - 1)
  → Submit to NVMe Submission Queue     ← NVMe-specific
```


## Chapter 15: How a SATA Driver Works

SATA (Serial ATA) is the most common interface for consumer SSDs and HDDs.
The driver communicates through a hardware interface called **AHCI**
(Advanced Host Controller Interface).

```
SATA/AHCI Communication:

CPU ←──── PCIe bus ────→ AHCI Controller ←── SATA cable ──→ Drive

Step 1: Driver builds a Command FIS (Frame Information Structure)
        in a special memory region called the Command Table.

        Command Table (in DMA-accessible memory):
        ┌────────────────────────────────┐
        │ Command FIS:                    │
        │   FIS type: 0x27 (H2D Register)│
        │   Command: 0x25 (READ DMA EXT) │
        │   LBA: 50432                    │
        │   Sector count: 8              │
        ├────────────────────────────────┤
        │ PRDT (Physical Region Desc.):   │
        │   Base address: 0x1A3000       │  ← where to put the data in RAM
        │   Byte count: 4096             │
        └────────────────────────────────┘

Step 2: Driver writes to the AHCI port's Command Issue register.
        "Hey controller, slot 0 has a command ready. Go!"

Step 3: AHCI controller reads the Command Table via DMA.
        Controller sends the command to the drive over SATA.

Step 4: Drive processes the command, reads the data from its media.

Step 5: Drive sends the data back over SATA to the AHCI controller.

Step 6: AHCI controller writes the data to RAM via DMA at the
        address specified in the PRDT (0x1A3000).

Step 7: AHCI controller triggers an INTERRUPT.
        CPU: "Oh, the I/O is done!"

Step 8: Driver's interrupt handler runs:
        - Marks the request as complete
        - Wakes up any process waiting for this I/O (e.g., fsync)
        - Tells the block layer: "request done"
```

**DMA (Direct Memory Access)** is crucial here: the disk controller reads
and writes RAM directly, WITHOUT the CPU copying data byte by byte. The CPU
just sets up the command and gets interrupted when it's done. This frees
the CPU to do other work while the disk transfer happens.


## Chapter 16: How an NVMe Driver Works

NVMe (Non-Volatile Memory Express) is the modern interface designed
specifically for SSDs. It's fundamentally different from SATA:

```
SATA:  1 command queue,  max 32 commands in flight
NVMe:  65,535 queues,    max 65,536 commands per queue!
```

NVMe uses **Submission Queues** and **Completion Queues** in host memory:

```
NVMe Communication:

CPU ←──── PCIe bus ────→ NVMe Controller (on the SSD itself)

┌─────────────────────────────────────────────────────┐
│  Host Memory (RAM)                                   │
│                                                     │
│  Submission Queue (SQ) — ring buffer, CPU writes:    │
│  ┌─────┬─────┬─────┬─────┬─────┬─────┐             │
│  │CMD 0│CMD 1│CMD 2│     │     │     │             │
│  └─────┴─────┴─────┴─────┴─────┴─────┘             │
│   ↑ tail                                             │
│                                                     │
│  Completion Queue (CQ) — ring buffer, SSD writes:    │
│  ┌─────┬─────┬─────┬─────┬─────┬─────┐             │
│  │CMP 0│CMP 1│     │     │     │     │             │
│  └─────┴─────┴─────┴─────┴─────┴─────┘             │
│   ↑ head                                             │
└─────────────────────────────────────────────────────┘

Step 1: Driver builds an NVMe command (16 bytes):
        ┌────────────────────────────────┐
        │ Opcode: 0x02 (Read)            │
        │ NSID: 1 (namespace)            │
        │ PRP1: 0x1A3000 (data dest.)    │
        │ SLBA: 50432 (start block)      │
        │ NLB: 7 (8 blocks - 1)         │
        └────────────────────────────────┘

Step 2: Driver places the command in the Submission Queue.
        Driver rings the "doorbell" — writes to an MMIO register
        telling the SSD: "There's a new command at tail position X"

Step 3: SSD reads the command from RAM via PCIe DMA.
        SSD processes the command internally.
        (SSD has its own internal controller, flash translation layer,
         wear leveling, garbage collection, etc.)

Step 4: SSD writes the result data to RAM at PRP1 address (DMA).

Step 5: SSD writes a Completion Entry to the Completion Queue.
        SSD triggers an MSI-X interrupt (one per queue — no sharing).

Step 6: Driver's interrupt handler:
        - Reads the Completion Queue entry
        - Marks the request as done
        - Notifies the block layer
```

### Why NVMe is Faster Than SATA

```
SATA:
  - 1 command queue → serialized commands
  - AHCI register-based → CPU must write multiple registers per command
  - Shared interrupt → handler must check which command completed
  - Max throughput: ~600 MB/s (SATA III limit)

NVMe:
  - 65K queues → massive parallelism
  - Memory-mapped queues → CPU just writes to RAM, rings a doorbell
  - Per-queue MSI-X interrupts → no interrupt sharing overhead
  - Max throughput: ~7,000 MB/s (PCIe 4.0 x4)
  - Latency: ~10-20 microseconds (vs ~100 us for SATA)
```

**For your storage engine**: On NVMe, concurrent `pread()`/`pwrite()` calls
from multiple threads actually get dispatched in parallel to the SSD.
Your buffer pool's design should account for this — flushing multiple
dirty pages concurrently is faster than flushing them one by one.


## Chapter 17: The Drive's Internal Cache

Here's the scary detail: even after the device driver sends a write command
and gets a "success" completion, the data might STILL not be on the
physical media. The drive has its own volatile write cache:

```
Drive internals:

  ┌──────────────────────────────────────────┐
  │ Drive Controller (inside the SSD/HDD)     │
  │                                          │
  │ ┌──────────────────────────────────────┐ │
  │ │ Write Cache (DRAM, volatile!)        │ │ ← data lands HERE first
  │ │ Typically 256MB - 1GB on SSDs        │ │
  │ │                                      │ │
  │ │ Drive can report "write complete"    │ │
  │ │ while data is still in this cache!   │ │
  │ └──────────────────────┬───────────────┘ │
  │                        │                 │
  │                   (later)                │
  │                        ▼                 │
  │ ┌──────────────────────────────────────┐ │
  │ │ NAND Flash / Magnetic Platters       │ │ ← ACTUALLY permanent
  │ │ (non-volatile — survives power loss)  │ │
  │ └──────────────────────────────────────┘ │
  └──────────────────────────────────────────┘
```

**This is another layer where data loss can happen!**

When you call `fsync()`:
1. The kernel flushes dirty pages from the page cache (Part A)
2. The block I/O layer submits write commands (Part B)
3. The device driver sends the commands to the drive (Part C)
4. **Then the kernel sends a FLUSH CACHE command to the drive**
5. The drive writes its volatile cache to the permanent media
6. The drive reports "flush complete"
7. **NOW** `fsync()` returns to your program

Step 4 is critical. Without the FLUSH CACHE command, `fsync()` would be
lying — the data could still be in the drive's volatile write cache.

```
Enterprise SSDs: Have battery-backed or capacitor-backed DRAM cache.
  Even if power fails, the capacitor provides enough energy to flush
  the DRAM cache to NAND. So FLUSH CACHE is almost a no-op.
  Cost: $$$

Consumer SSDs: Have volatile DRAM cache with NO battery backup.
  FLUSH CACHE must actually write to NAND.
  If the drive LIES about flush completion (some cheap drives do),
  fsync() is meaningless and your database can lose data.
  Cost: $

HDDs: Some have volatile write cache (8-256MB).
  Write cache can be disabled via hdparm:
  $ sudo hdparm -W 0 /dev/sda  (disable write cache)
  This makes every write go to platters but is MUCH slower.
```

**For your storage engine**: This is why enterprise databases run on
enterprise SSDs with power-loss protection. Your WAL's fsync guarantee
is only as strong as the drive's FLUSH CACHE honesty.


---

# Part D: The Complete I/O Path

## Chapter 18: From write() to Physical Media

Here is every step, with timing, when your storage engine writes a WAL
record and calls fsync:

```
Your code:
  wal_file.write_all(&record)?;   // Step 1-4
  wal_file.sync_all()?;           // Step 5-12

Timeline:

  Step 1:  write() syscall enters kernel                     ~2 ns
           (privilege flip, no page table switch)

  Step 2:  VFS → filesystem's write_iter()                   ~10 ns
           filesystem calls write_begin()

  Step 3:  Data copied into page cache page                  ~50 ns
           (memcpy from your user buffer to kernel page)

  Step 4:  write_end() marks page as dirty                   ~10 ns
           write() syscall returns to your code
           ─── DATA IS IN PAGE CACHE (RAM) ONLY ───

  Step 5:  sync_all() syscall enters kernel                  ~2 ns

  Step 6:  VFS calls filesystem's fsync()                    ~10 ns

  Step 7:  Filesystem calls filemap_write_and_wait()         ~100 ns
           → finds all dirty pages for this file
           → calls writepages() for each dirty page

  Step 8:  Filesystem creates bio structures                 ~200 ns
           → maps file offsets to disk block numbers
           → submits bios to the block I/O layer

  Step 9:  Block I/O layer:                                  ~500 ns
           → attempts to merge adjacent bios
           → I/O scheduler orders the requests
           → dispatches to device driver

  Step 10: Device driver:                                    ~1000 ns
           → builds hardware command (NVMe/SATA)
           → submits to hardware queue
           → rings the doorbell

  Step 11: Drive hardware:                                   ~20,000 ns (NVMe)
           → reads command from submission queue               ~100,000 ns (SATA SSD)
           → writes data from drive cache to NAND flash        ~8,000,000 ns (HDD)
           → posts completion to completion queue
           → triggers interrupt

  Step 12: Kernel sends FLUSH CACHE command to drive          ~50,000 ns (NVMe)
           → drive flushes volatile cache to media             ~500,000 ns (SATA)
           → completion interrupt
           → fsync() returns to your code
           ─── DATA IS NOW ON PHYSICAL MEDIA ───

  Total fsync latency:
    NVMe SSD:   ~50-100 microseconds
    SATA SSD:   ~500-2000 microseconds
    HDD:        ~8,000-16,000 microseconds
```


## Chapter 19: The Gap Where Data Loss Happens

```
Time ──────────────────────────────────────────────────────────────►

write()                                       fsync()
returns                                       returns
  │                                             │
  │◄─────── DATA ONLY IN RAM ──────────────────►│
  │         (page cache)                        │
  │                                             │
  │  POWER FAILURE HERE = DATA LOST             │  DATA SAFE
  │  KERNEL PANIC HERE = DATA LOST              │  (on physical media)
  │  PROCESS CRASH HERE = data survives         │
  │  (kernel keeps the page cache alive)        │
```

Notice the subtle difference:
- **Process crash**: data survives! The page cache is in kernel memory, which
  persists even after your process dies. The kernel's writeback thread will
  eventually flush the dirty pages. (But you can't rely on this for database
  correctness — only fsync gives you a guarantee.)
- **Kernel panic / power failure**: data is lost. The page cache is volatile RAM.
  Only fsync before the failure would have saved it.


## Chapter 20: Connection to Your Storage Engine

```
Storage Engine Layer          OS Layer That Handles It
────────────────────          ────────────────────────
WAL append (write)        →   Page Cache (buffered in RAM)
WAL flush (fsync)         →   Page Cache → Block I/O → Driver → Disk
Data page write           →   Page Cache (buffered, flushed at checkpoint)
Data page read            →   Page Cache hit (fast) or Block I/O miss (slow)
Pre-allocate file         →   Filesystem allocates blocks, Block I/O writes
Buffer pool evict dirty   →   Page Cache → Block I/O → Driver → Disk
O_DIRECT bypass           →   Skip page cache, go directly to Block I/O

Your WAL protocol ensures:
  1. WAL record written to page cache (write)
  2. WAL record flushed to disk (fsync) ← passes through ALL three layers
  3. ONLY THEN can the data page be flushed
  4. This ordering survives power failure because fsync waits for
     the ENTIRE stack (page cache → block I/O → driver → hardware flush)
```

The three layers — Page Cache, Block I/O Layer, and Device Driver — are
the infrastructure that makes your `fsync()` call meaningful. Without
understanding them, you can't reason about durability. With this
understanding, you know exactly what "data is on disk" really means,
and why the gap between `write()` and `fsync()` is where your WAL
protocol earns its keep.
