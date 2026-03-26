# The Complete I/O Pipeline: VFS → Page Cache → Block I/O

## Every Step Connected — Inputs, Outputs, and Data Flow

---

# How to Read This Guide

Every step shows:
- **WHO**: Which kernel component is running
- **INPUT**: What it receives from the previous step
- **WHAT IT DOES**: The actual work
- **OUTPUT**: What it passes to the next step
- **WHERE DATA LIVES**: Physical location of your data at this moment

We trace TWO complete operations:
1. **write()** — your program writes data (Chapters 1-10)
2. **fsync()** — your program forces data to disk (Chapters 11-20)

These are the two syscalls your storage engine uses most.

---

# THE WRITE PATH: write(fd, "Hello", 5)

## Chapter 1: Your Program (User Space)

```
WHO:    Your Rust program
INPUT:  fd=3, buffer="Hello" at address 0x7FFE0040, count=5
ACTION: Executes the SYSCALL instruction

┌──────────────────────────────────────────────────────────┐
│  Your code:                                              │
│    let data = b"Hello";                                  │
│    file.write_all(data)?;                                │
│                                                          │
│  Rust std library translates to:                         │
│    libc::write(fd=3, buf=0x7FFE0040, count=5)            │
│                                                          │
│  libc translates to:                                     │
│    mov rax, 1          ; syscall number 1 = SYS_write    │
│    mov rdi, 3          ; fd                              │
│    mov rsi, 0x7FFE0040 ; buffer pointer                  │
│    mov rdx, 5          ; byte count                      │
│    syscall             ; enter kernel                    │
└──────────────────────────────────────────────────────────┘

OUTPUT → to kernel: (syscall_number=1, fd=3, buf_ptr=0x7FFE0040, count=5)
DATA LIVES: in your user-space buffer at 0x7FFE0040
```


## Chapter 2: Syscall Entry (Kernel, Privilege Switch)

```
WHO:    CPU hardware + kernel syscall entry code
INPUT:  (syscall_number=1, fd=3, buf_ptr=0x7FFE0040, count=5)

┌──────────────────────────────────────────────────────────┐
│  CPU hardware does (one instruction):                    │
│    1. CPL: 3 → 0 (switch to kernel mode)                 │
│    2. RSP → this process's kernel stack                  │
│    3. RIP → kernel's syscall_entry function              │
│                                                          │
│  Kernel's syscall_entry() does:                          │
│    1. Save all user registers onto kernel stack          │
│    2. Read rax (syscall number = 1 = SYS_write)         │
│    3. Look up syscall table: table[1] = sys_write        │
│    4. Call sys_write(fd=3, buf=0x7FFE0040, count=5)      │
└──────────────────────────────────────────────────────────┘

OUTPUT → to VFS: sys_write(fd=3, buf=0x7FFE0040, count=5)
DATA LIVES: still in your user-space buffer (unchanged)
```


## Chapter 3: VFS Layer — sys_write()

```
WHO:    Virtual File System (VFS) — fs/read_write.c
INPUT:  (fd=3, user_buf=0x7FFE0040, count=5)

┌──────────────────────────────────────────────────────────┐
│  sys_write() does:                                       │
│                                                          │
│  Step 1: Convert fd → struct file                        │
│    struct file *f = current->files->fd_table[3];         │
│                                                          │
│    Result:                                               │
│      f->f_inode = inode 4521 (your database file)        │
│      f->f_pos = 8192 (current file offset)               │
│      f->f_op = ext4_file_operations (function table)     │
│      f->f_mapping = inode->i_mapping (page cache)        │
│                                                          │
│  Step 2: Security check                                  │
│    Can this process write to this file? Check f->f_mode. │
│    → Yes, O_WRONLY or O_RDWR was used when opening.      │
│                                                          │
│  Step 3: Call the filesystem's write method               │
│    f->f_op->write_iter(kiocb, iov_iter)                  │
│                                                          │
│    Building the arguments:                               │
│      kiocb.ki_filp = f          (which file)             │
│      kiocb.ki_pos = 8192        (write at offset 8192)   │
│      iov_iter.buf = 0x7FFE0040  (user buffer address)    │
│      iov_iter.count = 5         (byte count)             │
└──────────────────────────────────────────────────────────┘

OUTPUT → to filesystem: ext4_file_write_iter(kiocb, iov_iter)
  kiocb contains: file pointer, offset=8192
  iov_iter contains: user buffer address, count=5

DATA LIVES: still in your user-space buffer (unchanged)
```

**The key translation here:** fd (an integer) → struct file (kernel object)
→ filesystem operations table → specific filesystem function. This is
the VFS dispatch — the Strategy Pattern in action.


## Chapter 4: Filesystem Layer — ext4_file_write_iter()

```
WHO:    ext4 filesystem driver — fs/ext4/file.c
INPUT:  kiocb (file + offset=8192), iov_iter (user_buf + count=5)

┌──────────────────────────────────────────────────────────┐
│  ext4_file_write_iter() does:                            │
│                                                          │
│  Step 1: Calculate which PAGE of the file we're writing  │
│    file_offset = 8192                                    │
│    page_index = 8192 / 4096 = 2                          │
│    offset_within_page = 8192 % 4096 = 0                  │
│                                                          │
│    → We need page index 2 of this file.                  │
│    → We're writing at byte 0 within that page.           │
│    → We're writing 5 bytes.                              │
│                                                          │
│  Step 2: Call generic_perform_write()                    │
│    This is the standard "write data into page cache"     │
│    routine, shared by most filesystems.                  │
│                                                          │
│    It calls two sub-steps:                               │
│      a. write_begin() — prepare the target page          │
│      b. write_end()   — finalize after copying           │
└──────────────────────────────────────────────────────────┘

OUTPUT → to page cache operations: 
  "I need page index 2 of inode 4521, 
   I'm going to write 5 bytes at offset 0 within that page"

DATA LIVES: still in your user-space buffer
```


## Chapter 5: write_begin() — Preparing the Page Cache Page

```
WHO:    ext4's write_begin → page cache (address_space)
INPUT:  inode=4521, page_index=2, offset_in_page=0, length=5

┌──────────────────────────────────────────────────────────┐
│  ext4_write_begin() does:                                │
│                                                          │
│  Step 1: Find or create the page in the page cache       │
│                                                          │
│    Look up: inode->i_mapping (the address_space)          │
│    Search:  address_space->i_pages (radix tree/xarray)    │
│    Key:     page_index = 2                                │
│                                                          │
│    ┌── Page FOUND in cache ─────────────────────────┐    │
│    │  page = existing page at index 2                │    │
│    │  No disk I/O needed.                            │    │
│    │  Lock the page (so nobody else modifies it)     │    │
│    └─────────────────────────────────────────────────┘    │
│                                                          │
│    ┌── Page NOT found ──────────────────────────────┐    │
│    │  Allocate a new 4KB page from kernel memory     │    │
│    │  page = alloc_page(GFP_KERNEL)                  │    │
│    │                                                 │    │
│    │  Do we need to read the EXISTING data from disk?│    │
│    │                                                 │    │
│    │  If we're overwriting the ENTIRE page: No.      │    │
│    │    Just use the empty page.                     │    │
│    │                                                 │    │
│    │  If we're overwriting only PART of the page: Yes│    │
│    │    We must read the existing page from disk     │    │
│    │    first, so we don't lose the other bytes.     │    │
│    │    → THIS triggers a disk read!                 │    │
│    │    → ext4 calls ext4_readpage()                 │    │
│    │    → which calls submit_bio(READ, block_num)    │    │
│    │    → Block I/O layer reads from disk            │    │
│    │    → Page now has the old data from disk        │    │
│    │                                                 │    │
│    │  Insert the page into the radix tree:           │    │
│    │    address_space->i_pages[2] = page             │    │
│    │  Lock the page.                                 │    │
│    └─────────────────────────────────────────────────┘    │
│                                                          │
│  Step 2: Map disk blocks for this file region            │
│    ext4 checks its extent tree (in the inode):           │
│      "File offset 8192-8196 maps to disk block 50432"   │
│    This mapping will be needed later when flushing.      │
│                                                          │
│  Step 3: Journal (ext4-specific)                         │
│    ext4 writes a journal entry:                          │
│      "I'm about to modify the data block for             │
│       inode 4521 at file offset 8192"                    │
│    (This is ext4's own WAL — separate from yours!)       │
└──────────────────────────────────────────────────────────┘

OUTPUT → to the copy step:
  locked_page: a 4KB kernel memory page, 
    either fresh or containing the old data from disk
  page is at kernel virtual address, e.g., 0xFFFF888001230000
  
  The page is LOCKED — no other thread can modify it.

DATA LIVES: 
  - Your "Hello" is still in user buffer 0x7FFE0040
  - The TARGET page is in kernel memory (page cache)
  - The page may contain old data from disk, or zeros
```

**This is the critical decision point**: if you're writing a full 4KB page
(like your storage engine does), no disk read is needed. If you're writing
a partial page (like 5 bytes), the kernel must read the rest from disk first.


## Chapter 6: The Copy — User Buffer → Page Cache

```
WHO:    generic_perform_write() internals
INPUT:  locked_page (kernel memory), user_buf=0x7FFE0040, 
        offset_in_page=0, length=5

┌──────────────────────────────────────────────────────────┐
│  The actual data copy:                                   │
│                                                          │
│  Step 1: Get a writable pointer to the page              │
│    kaddr = kmap_local_page(page)                         │
│    → kaddr = 0xFFFF888001230000 (kernel virtual address) │
│                                                          │
│  Step 2: Copy from user space to kernel page cache       │
│    copy_from_user(kaddr + 0, user_buf, 5)                │
│                                                          │
│    What this does:                                       │
│      Source: 0x7FFE0040 (your buffer in user space)      │
│      Dest:   0xFFFF888001230000 (page cache in kernel)   │
│      Count:  5 bytes                                     │
│                                                          │
│    Before copy:                                          │
│      page cache page: [old old old old old old ...]      │
│      user buffer:     [H e l l o]                        │
│                                                          │
│    After copy:                                           │
│      page cache page: [H e l l o old old old ...]        │
│      user buffer:     [H e l l o] (unchanged)            │
│                                                          │
│  Step 3: Unmap                                           │
│    kunmap_local(kaddr)                                   │
│                                                          │
│  ★ THIS IS THE MOMENT YOUR DATA ENTERS KERNEL MEMORY ★  │
│  ★ It's now in the page cache, but NOT on disk yet.  ★   │
└──────────────────────────────────────────────────────────┘

OUTPUT → to write_end: 
  The page cache page now contains your data.
  bytes_copied = 5

DATA LIVES:
  ✓ In user buffer (0x7FFE0040) — still there, unchanged
  ★ In page cache (0xFFFF888001230000) — NEW copy here
  ✗ NOT on disk yet
```


## Chapter 7: write_end() — Marking the Page Dirty

```
WHO:    ext4_write_end() → page cache infrastructure
INPUT:  the page we just copied into, bytes_copied=5

┌──────────────────────────────────────────────────────────┐
│  ext4_write_end() does:                                  │
│                                                          │
│  Step 1: Mark the page as DIRTY                          │
│    set_page_dirty(page)                                  │
│                                                          │
│    What this does internally:                            │
│      page->flags |= PG_dirty                             │
│                                                          │
│      The page is also added to the inode's dirty list:   │
│        inode->i_mapping->dirty_pages (a linked list)     │
│                                                          │
│      AND to the filesystem's dirty inode list:           │
│        The superblock tracks all inodes with dirty pages │
│        sb->s_dirty_inodes                                │
│                                                          │
│    Dirty tracking is THREE levels deep:                  │
│      Level 1: The page itself (PG_dirty flag)            │
│      Level 2: The inode's dirty page list                │
│      Level 3: The filesystem's dirty inode list          │
│                                                          │
│    This hierarchy lets the kernel efficiently answer:    │
│      "Which pages are dirty?" → check the page flag      │
│      "Which files have dirty pages?" → check the list    │
│      "Which filesystems need flushing?" → check the list │
│                                                          │
│  Step 2: Update the inode metadata                       │
│    inode->i_mtime = current_time()  (modification time)  │
│    inode->i_size = max(i_size, offset + bytes_written)   │
│    mark_inode_dirty(inode)                                │
│                                                          │
│  Step 3: Unlock the page                                 │
│    unlock_page(page)                                     │
│    → Other threads can now access this page              │
│                                                          │
│  Step 4: Update the file position                        │
│    file->f_pos += 5  (advance file offset from 8192 to   │
│                        8197 for next write)               │
└──────────────────────────────────────────────────────────┘

OUTPUT → back up the call chain:
  returns bytes_written = 5

DATA LIVES:
  ★ In page cache — marked DIRTY
  ✗ NOT on disk
  The kernel knows: "inode 4521, page index 2, is dirty and needs flushing"
```


## Chapter 8: Returning to User Space

```
WHO:    syscall return path
INPUT:  bytes_written = 5

┌──────────────────────────────────────────────────────────┐
│  The return path (unwinding the call chain):             │
│                                                          │
│  ext4_write_end() returns 5                              │
│    ↑                                                     │
│  generic_perform_write() returns 5                       │
│    ↑                                                     │
│  ext4_file_write_iter() returns 5                        │
│    ↑                                                     │
│  vfs_write() returns 5                                   │
│    ↑                                                     │
│  sys_write() returns 5                                   │
│    ↑                                                     │
│  syscall_entry():                                        │
│    1. Put return value (5) in rax register               │
│    2. Restore all user registers from kernel stack       │
│    3. Execute SYSRETQ instruction:                       │
│       - CPL: 0 → 3 (back to user mode)                  │
│       - RIP → your code (right after the syscall)        │
│       - RSP → your user stack                            │
│                                                          │
│  Your Rust code:                                         │
│    write_all() checks: 5 == 5? Yes, all bytes written.   │
│    Returns Ok(()).                                        │
└──────────────────────────────────────────────────────────┘

OUTPUT → to your program: Ok(())  (write succeeded)

DATA LIVES:
  ★ In page cache (kernel memory) — DIRTY
  ✗ NOT on disk
  Your program thinks "data is written." It IS — but only to RAM.
```


## Chapter 9: The Waiting Period — Data in Limbo

```
WHO:    Nobody is doing anything with your data right now
STATUS: Your 5 bytes are sitting in a dirty page cache page

┌──────────────────────────────────────────────────────────┐
│                                                          │
│  YOUR DATA IS IN THE "DANGER ZONE"                       │
│                                                          │
│  The dirty page sits in the page cache.                  │
│  The kernel WILL eventually flush it, but not yet.       │
│                                                          │
│  What could flush it:                                    │
│                                                          │
│  1. Background writeback (5-30 seconds from now)         │
│     → kworker thread wakes up, sees dirty pages          │
│     → writes them to disk in the background              │
│                                                          │
│  2. Memory pressure (unpredictable)                      │
│     → System runs low on RAM                             │
│     → Kernel needs to free pages                         │
│     → Must flush dirty pages before evicting them        │
│                                                          │
│  3. YOUR fsync() call (you decide when)                  │
│     → Forces immediate flush to disk                     │
│     → Blocks until complete                              │
│     → THE ONLY RELIABLE OPTION for databases             │
│                                                          │
│  What could LOSE it:                                     │
│                                                          │
│  ✗ Power failure → RAM loses all content → data gone     │
│  ✗ Kernel panic  → no orderly shutdown → data gone       │
│  ✓ Process crash → data survives! (kernel still has it)  │
│  ✓ Signal/kill   → data survives! (kernel still has it)  │
│                                                          │
└──────────────────────────────────────────────────────────┘

DATA LIVES: page cache ONLY. Volatile RAM. Not durable.
```


## Chapter 10: Summary of the Write Path

```
COMPLETE WRITE PATH — DATA FLOW:

User buffer          Page cache page       Disk
(0x7FFE0040)         (kernel memory)       (physical media)
     │                     │                    │
     │  ① syscall          │                    │
     │  (privilege flip)   │                    │
     │         │           │                    │
     │         ▼           │                    │
     │  ② VFS: fd→file     │                    │
     │  ③ VFS: dispatch    │                    │
     │         │           │                    │
     │         ▼           │                    │
     │  ④ ext4: compute    │                    │
     │    page_index=2     │                    │
     │         │           │                    │
     │         ▼           │                    │
     │  ⑤ write_begin:     │                    │
     │    find/create page─┼──→ ★ page created  │
     │    (may read disk   │    or found in     │
     │     for partial     │    cache           │
     │     page writes)    │                    │
     │         │           │                    │
     │         ▼           │                    │
     ├─────────────────────┤                    │
     │  ⑥ copy_from_user:  │                    │
     │  "Hello" copied ───►│ "Hello" now HERE   │
     │  from user buf      │ in page cache      │
     ├─────────────────────┤                    │
     │         │           │                    │
     │         ▼           │                    │
     │  ⑦ write_end:       │                    │
     │    mark DIRTY ──────┼──→ ★ page is DIRTY │
     │    update inode      │                    │
     │    unlock page       │                    │
     │         │           │                    │
     │         ▼           │                    │
     │  ⑧ return to user   │                    │
     │    write() = 5      │                    │
     │                     │                    │
     │                     │    NOT HERE YET!   │
     │                     │    ═══════════════ │
     │                     │    Need fsync()    │
     │                     │    to get here     │
     │                     │         ↓          │
     │                     │    (see Part 2)    │
```


---

# THE FSYNC PATH: fsync(fd)

Your data is in the page cache, marked dirty. Now you call `fsync()` to
force it to the physical disk.


## Chapter 11: fsync() Enters the Kernel

```
WHO:    Your program → syscall entry
INPUT:  fd=3

┌──────────────────────────────────────────────────────────┐
│  Your code:                                              │
│    file.sync_all()?;     // Rust                         │
│    → libc::fsync(3)      // C                            │
│    → syscall(74, 3)      // syscall number 74 = fsync    │
│                                                          │
│  CPU: privilege flip, same as write path.                │
│                                                          │
│  Kernel:                                                 │
│    sys_fsync(fd=3)                                       │
│    → convert fd to struct file (same as write path)      │
│    → call vfs_fsync(file, datasync=0)                    │
│      datasync=0 means: flush data AND metadata           │
│      (fdatasync would pass datasync=1)                   │
└──────────────────────────────────────────────────────────┘

OUTPUT → to VFS: vfs_fsync(file, datasync=0)
DATA LIVES: still only in page cache (dirty)
```


## Chapter 12: VFS Dispatches to Filesystem's fsync

```
WHO:    VFS layer — fs/sync.c
INPUT:  struct file *f, datasync=0

┌──────────────────────────────────────────────────────────┐
│  vfs_fsync() does:                                       │
│                                                          │
│  Step 1: Call the filesystem's fsync operation            │
│    f->f_op->fsync(f, start=0, end=LLONG_MAX, datasync=0)│
│                                                          │
│    For ext4, this calls ext4_sync_file()                 │
│                                                          │
│    start=0, end=LLONG_MAX means: flush the ENTIRE file   │
│    (fsync always flushes all dirty pages for this file)   │
└──────────────────────────────────────────────────────────┘

OUTPUT → to ext4: ext4_sync_file(file, start=0, end=MAX, datasync=0)
DATA LIVES: still only in page cache (dirty)
```


## Chapter 13: Filesystem Flushes Dirty Pages

```
WHO:    ext4_sync_file() — fs/ext4/fsync.c
INPUT:  file (inode 4521), flush range = entire file

┌──────────────────────────────────────────────────────────┐
│  ext4_sync_file() does:                                  │
│                                                          │
│  Step 1: Flush all dirty PAGES to disk                   │
│    filemap_write_and_wait_range(inode->i_mapping, 0, MAX)│
│                                                          │
│    This function does two things:                        │
│      a. WRITE: submit all dirty pages for I/O            │
│      b. WAIT:  block until all I/O completes             │
│                                                          │
│    Let's trace filemap_write_and_wait_range():           │
│                                                          │
│    Step 1a: Find all dirty pages                         │
│      Walk inode->i_mapping->i_pages (radix tree)         │
│      Collect all pages with PG_dirty flag set            │
│                                                          │
│      Found: page at index 2 (our "Hello" page)           │
│      (In a real database, there might be hundreds of     │
│       dirty pages — they're all collected here)          │
│                                                          │
│    Step 1b: Call the filesystem's writepages()            │
│      This is where ext4 translates file pages            │
│      to disk blocks.                                     │
│                                                          │
│      → See Chapter 14 for the details                    │
│                                                          │
│  Step 2: Flush the ext4 journal                          │
│    jbd2_complete_transaction()                           │
│    (Make sure ext4's own metadata journal is on disk)    │
│                                                          │
│  Step 3: Flush the disk's hardware write cache           │
│    blkdev_issue_flush(sb->s_bdev)                        │
│    → See Chapter 18 for the details                      │
└──────────────────────────────────────────────────────────┘

OUTPUT → to writepages: list of dirty pages to flush
DATA LIVES: still in page cache (dirty), about to be written to disk
```


## Chapter 14: writepages() — File Offsets → Disk Blocks

```
WHO:    ext4_writepages() — fs/ext4/inode.c
INPUT:  list of dirty pages for inode 4521

┌──────────────────────────────────────────────────────────┐
│  ext4_writepages() does FOR EACH dirty page:             │
│                                                          │
│  Step 1: Map file offset to disk block number            │
│                                                          │
│    Dirty page: index=2, so file offset = 2 × 4096 = 8192│
│                                                          │
│    ext4 reads the inode's EXTENT TREE:                   │
│      The extent tree (stored in the inode on disk)       │
│      maps file logical blocks → physical disk blocks.    │
│                                                          │
│      Extent: logical blocks 0-99 → physical blocks       │
│              50430-50529                                  │
│                                                          │
│      Our file offset 8192 = logical block 2              │
│      Logical block 2 → physical block 50432              │
│                                                          │
│  Step 2: Create a bio (Block I/O request)                │
│                                                          │
│    bio = bio_alloc()                                     │
│    bio->bi_bdev = /dev/sda (block device)                │
│    bio->bi_sector = 50432 × 8 = 403456                   │
│      (disk sectors are 512 bytes, so 4096/512 = 8        │
│       sectors per page, starting at sector 403456)       │
│    bio->bi_opf = REQ_OP_WRITE                            │
│    bio->bi_io_vec[0]:                                    │
│      .bv_page = our dirty page (the one with "Hello")    │
│      .bv_len = 4096                                      │
│      .bv_offset = 0                                      │
│                                                          │
│  Step 3: Try to MERGE with adjacent bios                 │
│    If we also have dirty pages at index 1 and index 3,   │
│    they map to physical blocks 50431 and 50433.          │
│    These are adjacent on disk! Merge into one big bio:   │
│                                                          │
│    Before: bio1(blk 50431), bio2(blk 50432), bio3(50433) │
│    After:  one bio: blocks 50431-50433, 12KB, 3 pages    │
│                                                          │
│  Step 4: Submit the bio to the Block I/O layer           │
│    submit_bio(bio)                                       │
└──────────────────────────────────────────────────────────┘

OUTPUT → to Block I/O layer:
  bio structure:
    device: /dev/sda
    operation: WRITE
    starting sector: 403456
    pages: [the dirty page containing "Hello"]
    size: 4096 bytes (or more if merged)

DATA LIVES: 
  ★ In page cache (source of the write)
  → Being submitted to the block layer
```

**This is the bridge between "file world" and "disk world."** Before this
step, everything was in terms of files and offsets. After this step,
everything is in terms of disk sectors and physical blocks.


## Chapter 15: Block I/O Layer — Scheduling

```
WHO:    Block I/O layer — block/blk-mq.c
INPUT:  bio (device=/dev/sda, op=WRITE, sector=403456, 4096 bytes)

┌──────────────────────────────────────────────────────────┐
│  The block layer does:                                   │
│                                                          │
│  Step 1: Find the request queue for this device          │
│    /dev/sda → its request queue (blk_mq_hw_ctx)          │
│                                                          │
│  Step 2: Try to MERGE with existing requests             │
│    Check the queue: is there already a request for       │
│    adjacent sectors?                                     │
│                                                          │
│    Existing request: WRITE sectors 403448-403455          │
│    Our bio: WRITE sectors 403456-403463                  │
│    → BACK MERGE! Extend the existing request to          │
│      sectors 403448-403463 (two pages combined)          │
│                                                          │
│    If no merge possible: create a new request             │
│      struct request *rq = blk_mq_alloc_request()         │
│      rq->bio = our bio                                   │
│      rq->__sector = 403456                               │
│      rq->__data_len = 4096                               │
│                                                          │
│  Step 3: I/O scheduler decides ordering                  │
│    mq-deadline scheduler:                                │
│      - Assign a deadline to this request                 │
│        (writes: 5 second deadline)                       │
│      - Insert into sorted position                       │
│        (sorted by sector number for sequential access)   │
│                                                          │
│  Step 4: Dispatch to the device driver                   │
│    The scheduler picks the next request(s) to send:      │
│      - Priority: expired requests first                  │
│      - Then: batch nearby requests together              │
│      - Respect device queue depth (how many commands     │
│        the drive can handle simultaneously)              │
│                                                          │
│    blk_mq_dispatch_rq_list(hctx, &requests)              │
│    → calls the device driver's queue_rq() function       │
└──────────────────────────────────────────────────────────┘

OUTPUT → to device driver:
  struct request:
    sector: 403456 (or merged range)
    size: 4096 bytes (or more if merged)
    operation: WRITE
    pages: pointer to the page cache pages

DATA LIVES:
  ★ In page cache (the page data to be written)
  → request is in the device driver's hands now
```


## Chapter 16: Device Driver — Building the Hardware Command

```
WHO:    NVMe device driver — drivers/nvme/host/pci.c
        (or AHCI/SATA driver if it's a SATA drive)
INPUT:  struct request (sector=403456, size=4096, WRITE)

┌──────────────────────────────────────────────────────────┐
│  NVMe driver's nvme_queue_rq() does:                     │
│                                                          │
│  Step 1: Map the page to a DMA address                   │
│    The page is at kernel virtual address 0xFFFF8880...   │
│    But the NVMe controller needs a PHYSICAL address      │
│    (it accesses RAM directly via PCIe, bypassing the CPU │
│     and its virtual memory system).                      │
│                                                          │
│    dma_addr = dma_map_page(page)                         │
│    → e.g., dma_addr = 0x1A3000 (physical address)        │
│                                                          │
│  Step 2: Build the NVMe command (64 bytes)               │
│    ┌──────────────────────────────────────────┐          │
│    │ NVMe Write Command:                      │          │
│    │   Opcode: 0x01 (Write)                   │          │
│    │   NSID: 1 (namespace ID)                 │          │
│    │   PRP1: 0x1A3000 (physical addr of page) │          │
│    │   SLBA: 50432 (starting logical block)   │          │
│    │   NLB: 0 (number of blocks - 1 = 0 → 1) │          │
│    └──────────────────────────────────────────┘          │
│                                                          │
│  Step 3: Place the command in the Submission Queue       │
│    nvme_sq[tail] = command                               │
│    tail = (tail + 1) % queue_size                        │
│                                                          │
│  Step 4: Ring the doorbell                               │
│    Write the new tail value to the doorbell register:    │
│    writel(tail, nvme_db_addr)                            │
│                                                          │
│    This is a single MMIO write to a PCIe register.       │
│    The NVMe controller sees it and starts processing.    │
└──────────────────────────────────────────────────────────┘

OUTPUT → to NVMe hardware:
  A 64-byte command in the submission queue.
  The doorbell has been rung.
  The SSD controller is now reading the command.

DATA LIVES:
  ★ In page cache (the page data)
  → The SSD will DMA-read this data from RAM
```


## Chapter 17: Hardware — SSD Writes to NAND Flash

```
WHO:    NVMe SSD controller (hardware, not software)
INPUT:  NVMe Write command from the submission queue

┌──────────────────────────────────────────────────────────┐
│  The SSD hardware does:                                  │
│                                                          │
│  Step 1: Read the command from the Submission Queue      │
│    SSD's controller reads the 64-byte command via PCIe   │
│    DMA from host RAM.                                    │
│                                                          │
│  Step 2: DMA-read the data from host RAM                 │
│    The PRP1 field says: data is at physical 0x1A3000     │
│    SSD controller reads 4096 bytes from that address     │
│    via PCIe DMA — directly from RAM, CPU not involved.   │
│                                                          │
│  Step 3: Write to the SSD's internal write cache         │
│    The data goes into the SSD's DRAM write buffer.       │
│    ★ NOT on NAND flash yet! ★                            │
│                                                          │
│  Step 4: Post a Completion Entry                         │
│    SSD writes a completion entry to the Completion Queue │
│    in host RAM (via DMA):                                │
│    ┌──────────────────────────────────┐                  │
│    │ Completion entry:                 │                  │
│    │   Command ID: matches the write   │                  │
│    │   Status: SUCCESS                 │                  │
│    │   SQ Head: updated position       │                  │
│    └──────────────────────────────────┘                  │
│                                                          │
│  Step 5: Trigger an interrupt (MSI-X)                    │
│    SSD sends an interrupt to the specific CPU core       │
│    that submitted the command.                           │
│                                                          │
│  ★ AT THIS POINT:                                        │
│    - Data is in SSD's volatile DRAM cache                │
│    - NOT yet on NAND flash                               │
│    - Power loss NOW would lose the data!                 │
│    - That's why fsync() sends FLUSH CACHE next           │
└──────────────────────────────────────────────────────────┘

OUTPUT → to driver (via interrupt):
  Completion entry: command succeeded.
  Data is in SSD's write cache (volatile!)

DATA LIVES:
  ★ In page cache (kernel RAM) — still there as cache
  ★ In SSD's DRAM write cache — volatile!
  ✗ NOT on NAND flash yet
```


## Chapter 18: The FLUSH CACHE Command — Final Step

```
WHO:    Back in ext4_sync_file(), after all writes complete
INPUT:  All write I/Os have completed (data in drive's cache)

┌──────────────────────────────────────────────────────────┐
│  ext4_sync_file() continues:                             │
│                                                          │
│  Step 1: All dirty page writes are complete              │
│    filemap_write_and_wait() has returned.                │
│    Every dirty page has been submitted and the driver    │
│    has received completion interrupts for all of them.   │
│                                                          │
│  Step 2: Send FLUSH CACHE command                        │
│    blkdev_issue_flush(sb->s_bdev, GFP_KERNEL)            │
│                                                          │
│    This creates a special bio:                           │
│      bio->bi_opf = REQ_OP_FLUSH                          │
│      (no data — just a command to the drive)             │
│                                                          │
│    The bio goes through the block layer → driver:        │
│                                                          │
│    NVMe driver sends:                                    │
│    ┌──────────────────────────────────────────┐          │
│    │ NVMe Flush Command:                      │          │
│    │   Opcode: 0x00 (Flush)                   │          │
│    │   NSID: 1                                │          │
│    │   "Write your volatile cache to NAND!"   │          │
│    └──────────────────────────────────────────┘          │
│                                                          │
│  Step 3: SSD receives the Flush command                  │
│    The SSD controller:                                   │
│      a. Writes ALL data from DRAM cache to NAND flash    │
│      b. This is the slow part (~50-200 microseconds)     │
│      c. Posts a completion entry                         │
│      d. Triggers interrupt                               │
│                                                          │
│  Step 4: Driver receives flush completion                │
│    blkdev_issue_flush() returns.                         │
│    ext4_sync_file() returns.                             │
│    vfs_fsync() returns.                                  │
│    sys_fsync() returns.                                  │
│    SYSRETQ — back to user space.                         │
│    file.sync_all() returns Ok(()).                        │
│                                                          │
│  ★ NOW your data is on NAND flash. Durable. ★            │
│  ★ Power failure from this point on: data survives. ★    │
└──────────────────────────────────────────────────────────┘

OUTPUT → to your program: Ok(())
DATA LIVES:
  ✓ In page cache (kernel RAM) — still cached for fast re-reads
  ✓ In SSD's NAND flash — PERMANENT, survives power loss
```


## Chapter 19: Mark Pages Clean

```
WHO:    Page cache infrastructure (during filemap_write_and_wait)
INPUT:  Pages that have been successfully written to disk

┌──────────────────────────────────────────────────────────┐
│  After the I/O completion interrupt arrives:             │
│                                                          │
│  For each page that was written:                         │
│    1. Clear the PG_dirty flag on the page                │
│       page->flags &= ~PG_dirty                           │
│                                                          │
│    2. Set PG_uptodate flag (cache matches disk)          │
│       page->flags |= PG_uptodate                         │
│                                                          │
│    3. Remove from the inode's dirty page list            │
│       If no more dirty pages for this inode,             │
│       remove the inode from the filesystem's dirty list  │
│                                                          │
│  The page is now CLEAN:                                  │
│    - It's still in the page cache (for fast reads)       │
│    - But it matches what's on disk exactly               │
│    - If memory runs low, it can be evicted freely        │
│      (no need to write it first — disk has the data)     │
└──────────────────────────────────────────────────────────┘

DATA LIVES:
  ✓ In page cache (CLEAN — matches disk)
  ✓ On NAND flash (permanent)
  The page stays cached for fast future reads.
```


## Chapter 20: Summary — The Complete fsync Path

```
COMPLETE FSYNC PATH — DATA FLOW:

Page cache page        Block I/O         Device Driver      SSD Hardware
(kernel memory)        Layer             (NVMe/SATA)        (controller)
      │                  │                    │                  │
  ⑪ fsync() enters      │                    │                  │
  ⑫ VFS dispatches      │                    │                  │
  ⑬ ext4 collects       │                    │                  │
     dirty pages         │                    │                  │
      │                  │                    │                  │
  ⑭ writepages():       │                    │                  │
     file offset →       │                    │                  │
     disk block          │                    │                  │
     creates bio ───────►│                    │                  │
      │                  │                    │                  │
      │              ⑮ merge adjacent        │                  │
      │                 schedule order        │                  │
      │                 dispatch ────────────►│                  │
      │                  │                    │                  │
      │                  │               ⑯ build NVMe          │
      │                  │                  command              │
      │                  │                  DMA map              │
      │                  │                  ring doorbell ──────►│
      │                  │                    │                  │
      │                  │                    │             ⑰ DMA read
      │◄─────────────────┼────────────────────┼──────────── from RAM
      │  (SSD reads      │                    │              write to
      │   page data      │                    │              DRAM cache
      │   via DMA)       │                    │                  │
      │                  │                    │             interrupt
      │                  │                    │◄─────────────────┤
      │                  │                    │                  │
      │                  │               ⑱ FLUSH CACHE         │
      │                  │                  command ────────────►│
      │                  │                    │                  │
      │                  │                    │             DRAM → NAND
      │                  │                    │             (the real
      │                  │                    │              write!)
      │                  │                    │                  │
      │                  │                    │             interrupt
      │                  │                    │◄─────────────────┤
      │                  │                    │                  │
  ⑲ mark page CLEAN     │                    │                  │
      │                  │                    │                  │
  ⑳ fsync() returns     │                    │                  │
     to your program     │                    │                  │
      │                  │                    │                  │
      ▼                  ▼                    ▼                  ▼

  DATA IS NOW DURABLE. Power failure safe.
```


---

# Quick Reference: The Transformations at Each Boundary

```
Boundary 1: User space → Kernel (syscall)
  INPUT:  fd (integer), buffer pointer, byte count
  OUTPUT: struct file, inode, file offset
  TRANSFORM: fd number → kernel file object

Boundary 2: VFS → Filesystem (dispatch)
  INPUT:  struct file, iov_iter (user buffer)
  OUTPUT: filesystem-specific write call
  TRANSFORM: generic VFS call → ext4/xfs/btrfs specific function

Boundary 3: Filesystem → Page Cache (write_begin)
  INPUT:  inode, file offset, byte count
  OUTPUT: locked page in kernel memory
  TRANSFORM: file offset → page cache page index

Boundary 4: User buffer → Page cache (copy)
  INPUT:  user buffer address, page cache page
  OUTPUT: data copied into kernel memory
  TRANSFORM: user virtual address → kernel page content

Boundary 5: Page cache → bio (writepages)
  INPUT:  dirty page, inode extent tree
  OUTPUT: bio structure with disk sector number
  TRANSFORM: file page index → physical disk block number
  ★ THIS IS WHERE "FILE WORLD" BECOMES "DISK WORLD" ★

Boundary 6: bio → request queue (block layer)
  INPUT:  bio (device, sector, pages)
  OUTPUT: merged and scheduled request
  TRANSFORM: individual I/O ops → optimized batched requests

Boundary 7: request → hardware command (driver)
  INPUT:  struct request (sector, size, pages)
  OUTPUT: NVMe/SATA command + DMA address
  TRANSFORM: kernel data structures → hardware register/queue format

Boundary 8: hardware command → physical media (SSD)
  INPUT:  NVMe command + data in host RAM
  OUTPUT: data on NAND flash (after FLUSH CACHE)
  TRANSFORM: volatile DRAM cache → permanent NAND storage
```
