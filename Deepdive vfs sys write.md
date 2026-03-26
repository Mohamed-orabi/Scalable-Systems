# Deep Dive: VFS Layer — sys_write()

## Every Data Structure, Every Check, Every Dispatch

---

# Where We Are

The syscall entry code just called `sys_write(fd=3, buf=0x7FFE0040, count=5)`.
We're in kernel mode (ring 0), on the kernel stack. The VFS now takes over.

The VFS has ONE job: translate a generic "write to fd 3" into a SPECIFIC
filesystem function call (like ext4's write function). It does this through
a chain of data structure lookups.

Here's the full chain we'll trace:

```
fd (integer 3)
  → struct file (open file description)
    → struct inode (the actual file identity)
      → struct file_operations (function pointer table)
        → ext4_file_write_iter() (the specific filesystem function)
```

Let's trace every step.


---

# Chapter 1: sys_write() — The Entry Point

```c
// fs/read_write.c — the ACTUAL kernel source (simplified)

SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf,
                size_t, count)
{
    struct fd f = fdget_pos(fd);     // Step 1: fd → struct file
    ssize_t ret = -EBADF;

    if (f.file) {                     // Step 2: did we find a file?
        loff_t pos, *ppos = file_ppos(f.file);
        if (ppos) {
            pos = *ppos;
            ppos = &pos;
        }
        ret = vfs_write(f.file, buf, count, ppos);  // Step 3: VFS dispatch
        if (ret >= 0 && ppos)
            f.file->f_pos = pos;      // Step 4: update file position
        fdput_pos(f);                  // Step 5: release the reference
    }

    return ret;                        // Step 6: return to syscall layer
}
```

That's only 15 lines. But each line hides enormous complexity.
Let me expand every single one.


---

# Chapter 2: fdget_pos(fd) — Finding the File

This is the first and most critical step: converting the integer `fd=3`
into a pointer to the kernel's `struct file` object.

## 2.1: The Process's File Descriptor Table

Every process has a file descriptor table. It's a multi-level structure:

```
current (task_struct for your process)
  │
  │  current->files
  ▼
┌──────────────────────────────────────────────────────┐
│ struct files_struct                                    │
│                                                        │
│   atomic_t count = 1          (reference count)        │
│   struct fdtable *fdt ────────────────┐                │
│                                       │                │
│   // For small fd counts (≤64), the   │                │
│   // fdtable is embedded right here:  │                │
│   struct fdtable fdtab (embedded)     │                │
│   struct file *fd_array[64] ◄─────────┘ (for small     │
│                                          processes)    │
└──────────────────────────────────────────────────────┘

For processes with MORE than 64 open files, fdt points to a
separately allocated fdtable with a larger array.
```

## 2.2: The fdtable Structure

```c
struct fdtable {
    unsigned int max_fds;           // current capacity of the array
    struct file __rcu **fd;         // THE ARRAY: fd number → struct file *
    unsigned long *close_on_exec;   // bitmap: close this fd on exec()?
    unsigned long *open_fds;        // bitmap: is this fd currently open?
    unsigned long *full_fds_bits;   // bitmap: optimization for allocation
};

// For our example:
//   max_fds = 64
//   fd[0] = pointer to struct file for stdin
//   fd[1] = pointer to struct file for stdout
//   fd[2] = pointer to struct file for stderr
//   fd[3] = pointer to struct file for our database file  ★
//   fd[4] = pointer to struct file for our WAL file
//   fd[5] = NULL (not open)
//   fd[6] = NULL
//   ...
```

## 2.3: The Actual Lookup

```c
// fs/file.c (simplified)

struct fd fdget_pos(unsigned int fd) {
    // Step 1: Get the fdtable
    struct files_struct *files = current->files;
    struct fdtable *fdt = files->fdt;
    
    // Step 2: Bounds check
    if (fd >= fdt->max_fds)
        return (struct fd){NULL, 0};    // fd too large → EBADF
    
    // Step 3: Check if the fd is open (bitmap check)
    if (!test_bit(fd, fdt->open_fds))
        return (struct fd){NULL, 0};    // fd not open → EBADF
    
    // Step 4: Read the struct file pointer from the array
    struct file *file = fdt->fd[fd];    // fd[3] → pointer to struct file
    
    if (!file)
        return (struct fd){NULL, 0};    // shouldn't happen if bitmap says open
    
    // Step 5: Take a position lock (for f_pos serialization)
    // Multiple threads sharing an fd need to serialize their f_pos updates.
    // This is a lightweight mutex on the file's position.
    if (file_count(file) > 1)
        mutex_lock(&file->f_pos_lock);
    
    return (struct fd){file, flags};
}
```

**The complete pointer chain:**

```
current                              (register GS → per-CPU data → task_struct)
  → current->files                   (offset 0x6E0 in task_struct)
    → files->fdt                     (pointer to fdtable)
      → fdt->fd                      (pointer to array of struct file *)
        → fd[3]                      (array index 3 → struct file pointer)

Total: 4 pointer dereferences = ~4-8 nanoseconds
All in hot cache lines (file tables are accessed constantly)
```

## 2.4: What We Got — The struct file

```
fd[3] points to:

┌──────────────────────────────────────────────────────────────┐
│ struct file (at kernel address 0xFFFF888004567000)            │
│                                                              │
│ ┌────────────────────────────────────────────────────┐       │
│ │ f_path:                                             │       │
│ │   .mnt = pointer to vfsmount (which filesystem)     │       │
│ │   .dentry = pointer to dentry ("database.db")       │       │
│ │            dentry->d_name = "database.db"           │       │
│ │            dentry->d_parent = dentry for "/data/"   │       │
│ │            dentry->d_inode = pointer to inode 4521  │       │
│ └────────────────────────────────────────────────────┘       │
│                                                              │
│ f_inode = pointer to inode 4521 (shortcut, avoids going      │
│           through dentry every time)                         │
│                                                              │
│ f_op = pointer to ext4_file_operations ★                     │
│        (THIS is the function table that VFS will use)        │
│                                                              │
│ f_mapping = pointer to inode->i_mapping                      │
│             (the page cache / address_space for this file)   │
│                                                              │
│ f_pos = 8192 (current read/write offset in the file)         │
│         "We're at byte 8192 — the next write goes here"      │
│                                                              │
│ f_flags = O_WRONLY | O_CREAT (how the file was opened)       │
│                                                              │
│ f_mode = FMODE_WRITE (derived from f_flags)                  │
│          "This file descriptor allows writing"               │
│                                                              │
│ f_count = 1 (reference count — how many fd's point here)     │
│           If you dup(fd), this becomes 2.                    │
│           When it reaches 0, the struct file is freed.       │
│                                                              │
│ f_pos_lock = mutex (serializes position updates)             │
│                                                              │
│ f_cred = pointer to credentials (uid, gid of who opened it) │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**Where f_op came from:** When you called `open("/data/database.db", O_WRONLY)`,
the VFS did path resolution, found the inode (4521), and saw that the inode
belongs to an ext4 filesystem. The inode has `i_fop = &ext4_file_operations`.
The VFS copied this pointer into the new struct file's `f_op` field.

So `f_op` was set at open() time, and every subsequent read/write/fsync
uses it without re-doing the filesystem lookup. This is the Strategy Pattern
"baked in" at file open time.


---

# Chapter 3: file_ppos() — Getting the File Position

```c
// Back in sys_write():
loff_t pos, *ppos = file_ppos(f.file);
```

```c
// fs/read_write.c

static inline loff_t *file_ppos(struct file *file) {
    return file->f_mode & FMODE_STREAM ? NULL : &file->f_pos;
}
```

This is simple: for normal files, return a pointer to `f_pos` (the current
file offset). For "stream" files (like pipes, sockets), return NULL
(they don't have a seekable position).

For our database file: `f_pos = 8192`. The write will go to byte 8192.

```
Why does VFS manage the position?

  When you call write(fd, buf, 5):
    → VFS reads f_pos = 8192
    → write happens at offset 8192
    → VFS updates f_pos to 8197 (8192 + 5)
  
  Next write(fd, buf, 3):
    → VFS reads f_pos = 8197
    → write happens at offset 8197
    → VFS updates f_pos to 8200

  This is why sequential writes work without seeking.
  The VFS tracks where you are in the file automatically.

  pwrite(fd, buf, 5, offset) BYPASSES this:
    → uses the explicit offset, DOES NOT update f_pos
    → this is why pwrite is better for databases
```


---

# Chapter 4: vfs_write() — The Central Dispatcher

This is the heart of the VFS write path. It performs ALL the checks
before dispatching to the filesystem.

```c
// fs/read_write.c (simplified but complete logic)

ssize_t vfs_write(struct file *file, const char __user *buf,
                  size_t count, loff_t *pos)
{
    ssize_t ret;

    // ═══════════════════════════════════════════════════
    //  CHECK 1: Does this file support writing?
    // ═══════════════════════════════════════════════════
    
    if (!(file->f_mode & FMODE_WRITE))
        return -EBADF;
    //
    // If you opened with O_RDONLY, f_mode won't have FMODE_WRITE.
    // Result: write() returns -1, errno = EBADF.
    //
    // This catches: fd was opened read-only, or fd is a directory,
    // or fd is a special file that doesn't support writing.


    // ═══════════════════════════════════════════════════
    //  CHECK 2: Does the filesystem implement write?
    // ═══════════════════════════════════════════════════
    
    if (!file->f_op->write && !file->f_op->write_iter)
        return -EINVAL;
    //
    // Some filesystems are read-only (like iso9660 for CD-ROMs).
    // Their file_operations table has write = NULL and write_iter = NULL.
    // Can't write to a CD-ROM!
    //
    // For ext4: both write and write_iter are implemented.
    // Modern kernels prefer write_iter (vectorized I/O).


    // ═══════════════════════════════════════════════════
    //  CHECK 3: Is the user buffer valid?
    // ═══════════════════════════════════════════════════
    
    if (!access_ok(buf, count))
        return -EFAULT;
    //
    // access_ok checks:
    //   1. Is buf in the user address range? (below TASK_SIZE)
    //   2. Does buf + count overflow? (wraparound check)
    //
    // This prevents the user from passing a kernel address as buf,
    // which would trick the kernel into reading kernel memory.
    //
    // For our case: buf = 0x7FFE0040, count = 5
    //   0x7FFE0040 < TASK_SIZE (0x7FFFFFFFFFFF)? Yes.
    //   0x7FFE0040 + 5 = 0x7FFE0045, no overflow? Yes.
    //   → access_ok returns true. ✓


    // ═══════════════════════════════════════════════════
    //  CHECK 4: File size limits
    // ═══════════════════════════════════════════════════

    ret = rw_verify_area(WRITE, file, pos, count);
    if (ret)
        return ret;
    //
    // rw_verify_area checks:
    //   a. Is *pos negative? (invalid file position)
    //   b. Would pos + count overflow a loff_t? (64-bit overflow)
    //   c. Does this write exceed the process's RLIMIT_FSIZE?
    //      (ulimit -f sets a maximum file size per process)
    //   d. Does this write violate any mandatory file locks?
    //      (flock/fcntl locks — rare but must be checked)
    //
    // For our case:
    //   pos = 8192 (valid, not negative)
    //   8192 + 5 = 8197 (no overflow)
    //   RLIMIT_FSIZE = unlimited (default)
    //   No mandatory locks
    //   → returns 0 (success). ✓


    // ═══════════════════════════════════════════════════
    //  CHECK 5: Security module check (LSM)
    // ═══════════════════════════════════════════════════
    
    ret = security_file_permission(file, MAY_WRITE);
    if (ret)
        return ret;
    //
    // This calls into the Linux Security Module framework.
    // If SELinux or AppArmor is active, they check their policies:
    //   "Is process 1234 (ferrisdb) allowed to write to
    //    /data/database.db according to the security policy?"
    //
    // On a default system without SELinux: this is a no-op.
    // On a hardened system: this might deny the write!
    //
    // For our case: assume no LSM, returns 0. ✓


    // ═══════════════════════════════════════════════════
    //  ALL CHECKS PASSED — DISPATCH TO FILESYSTEM
    // ═══════════════════════════════════════════════════

    if (file->f_op->write_iter) {
        // Modern path: vectorized I/O (preferred)
        ret = new_sync_write(file, buf, count, pos);
    } else if (file->f_op->write) {
        // Legacy path: simple write
        ret = file->f_op->write(file, buf, count, pos);
    }

    // ═══════════════════════════════════════════════════
    //  POST-WRITE: notifications and accounting
    // ═══════════════════════════════════════════════════

    if (ret > 0) {
        // Notify anyone watching this file (inotify, fanotify)
        fsnotify_modify(file);
        
        // Update accounting: how many bytes has this process written?
        add_wchar(current, ret);
        
        // Increment the syscall counter
        inc_syscw(current);
    }

    return ret;
}
```

**Five checks before any data moves:**

```
CHECK 1: f_mode & FMODE_WRITE?     "Were you allowed to write when you opened?"
CHECK 2: f_op->write_iter exists?   "Does this filesystem support writing at all?"
CHECK 3: access_ok(buf, count)?     "Is your buffer pointer valid and in user space?"
CHECK 4: rw_verify_area()?          "Would this write exceed limits or violate locks?"
CHECK 5: security_file_permission()? "Does the security policy allow this?"

Only after ALL FIVE pass does the VFS call the filesystem.
```

**Analogy**: It's like a bank teller processing a transaction. Before moving
any money, they check: "Is the account active?" (CHECK 1), "Does this branch
handle this type of transaction?" (CHECK 2), "Is the withdrawal slip filled
out correctly?" (CHECK 3), "Would this exceed the daily limit?" (CHECK 4),
"Is this person on the sanctions list?" (CHECK 5). Only after all checks
pass do they actually process the transaction.


---

# Chapter 5: new_sync_write() — Building the I/O Descriptor

The VFS doesn't call ext4's write function directly with a simple buffer
pointer. It wraps the write in a standardized I/O descriptor structure.

```c
// fs/read_write.c

static ssize_t new_sync_write(struct file *filp, const char __user *buf,
                              size_t len, loff_t *ppos)
{
    // ═══════════════════════════════════════════════════
    //  BUILD the iov_iter — a "scatter-gather" descriptor
    // ═══════════════════════════════════════════════════
    
    struct iovec iov = {
        .iov_base = (void __user *)buf,    // 0x7FFE0040
        .iov_len = len                      // 5
    };
    //
    // An iovec describes one contiguous memory region.
    // For writev() (vectorized write), there would be multiple iovecs:
    //   iov[0] = {buf1, len1}
    //   iov[1] = {buf2, len2}
    //   iov[2] = {buf3, len3}
    // The kernel would write all of them in one operation.
    // For regular write(), there's just one iovec.
    
    struct iov_iter iter;
    iov_iter_init(&iter, WRITE, &iov, 1, len);
    //
    // iov_iter is the UNIVERSAL I/O descriptor in the Linux kernel.
    // It abstracts over:
    //   - User-space buffers (our case)
    //   - Kernel-space buffers (for sendfile, splice)
    //   - Page arrays (for direct I/O)
    //   - Pipe buffers (for pipe I/O)
    //   - Bio vectors (for block I/O)
    //
    // The filesystem's write_iter() function receives this and
    // doesn't need to know what kind of buffer is behind it.

    
    // ═══════════════════════════════════════════════════
    //  BUILD the kiocb — the I/O control block
    // ═══════════════════════════════════════════════════
    
    struct kiocb kiocb;
    init_sync_kiocb(&kiocb, filp);
    kiocb.ki_pos = *ppos;      // 8192 (write at this file offset)
    //
    // kiocb contains:
    //   ki_filp = filp         → which file
    //   ki_pos = 8192          → at what offset
    //   ki_flags = 0           → synchronous (not async I/O)
    //   ki_complete = NULL     → no async completion callback
    //
    // For async I/O (io_uring, aio), ki_complete would be set
    // and the write would return immediately, calling ki_complete
    // later when the I/O finishes. For normal write(), it's sync.


    // ═══════════════════════════════════════════════════
    //  DISPATCH to the filesystem
    // ═══════════════════════════════════════════════════
    
    ssize_t ret = call_write_iter(filp, &kiocb, &iter);
    //
    // This calls: filp->f_op->write_iter(&kiocb, &iter)
    // Which resolves to: ext4_file_write_iter(&kiocb, &iter)


    // Update the file position with however many bytes were written
    *ppos = kiocb.ki_pos;
    
    return ret;
}
```


## 5.1: What call_write_iter() Actually Does

```c
// include/linux/fs.h

static inline ssize_t call_write_iter(struct file *file,
                                      struct kiocb *kio,
                                      struct iov_iter *iter)
{
    return file->f_op->write_iter(kio, iter);
}
```

That's it — one line. It reads the function pointer from the operations
table and calls it. Let's see what's in that table.


---

# Chapter 6: The file_operations Table — The Strategy Pattern

The `f_op` pointer in our struct file points to a table of function
pointers. This table was set when the file was opened, based on which
filesystem the file belongs to.

```c
// fs/ext4/file.c — ext4's function table

const struct file_operations ext4_file_operations = {
    .llseek         = ext4_llseek,
    .read_iter      = ext4_file_read_iter,      // used by read()
    .write_iter     = ext4_file_write_iter,      // ★ used by write()
    .open           = ext4_file_open,
    .release        = ext4_release_file,
    .fsync          = ext4_sync_file,            // used by fsync()
    .mmap           = ext4_file_mmap,
    .splice_read    = generic_file_splice_read,
    .splice_write   = iter_file_splice_write,
    .fallocate      = ext4_fallocate,
    .unlocked_ioctl = ext4_ioctl,
};
```

**This is the Strategy Pattern at its purest.** The VFS code says
`file->f_op->write_iter(...)` and it doesn't know or care whether
that calls ext4's function, XFS's function, or NFS's function.

Different filesystems have different tables:

```c
// fs/xfs/xfs_file.c
const struct file_operations xfs_file_operations = {
    .write_iter     = xfs_file_write_iter,     // XFS implementation
    .fsync          = xfs_file_fsync,
    // ...
};

// fs/btrfs/file.c  
const struct file_operations btrfs_file_operations = {
    .write_iter     = btrfs_file_write_iter,   // Btrfs implementation
    .fsync          = btrfs_sync_file,
    // ...
};

// fs/nfs/file.c
const struct file_operations nfs_file_operations = {
    .write_iter     = nfs_file_write,          // NFS implementation (network!)
    .fsync          = nfs_file_fsync,
    // ...
};

// fs/fat/file.c
const struct file_operations fat_file_operations = {
    .write_iter     = generic_file_write_iter, // FAT uses the generic version
    .fsync          = fat_file_fsync,
    // ...
};
```

**How the correct table gets assigned:**

```
Timeline of how f_op gets set:

1. You call: open("/data/database.db", O_WRONLY)

2. VFS does path resolution:
   "/" → "data" → "database.db"
   
3. VFS finds the inode for "database.db" (inode 4521)
   The inode was created by ext4, so:
     inode->i_fop = &ext4_file_operations
   
4. VFS creates a new struct file:
     file->f_op = inode->i_fop = &ext4_file_operations
   
5. Every future read/write/fsync on this fd uses ext4's functions.
   The dispatch decision was made ONCE at open() time.
   No re-lookup needed for every I/O operation.
```

**This is how the VFS achieves its magic:** The filesystem is determined
at open() time and baked into the file object. All subsequent operations
are just indirect function calls through the operations table — O(1)
dispatch, no string comparisons, no type checking.


---

# Chapter 7: The Inputs to the Filesystem

Let's summarize exactly what ext4_file_write_iter() receives:

```
ext4_file_write_iter(kiocb, iov_iter) is called with:

┌─ kiocb (I/O Control Block) ──────────────────────────┐
│                                                       │
│  ki_filp ──→ struct file                              │
│              │ f_inode ──→ inode 4521                  │
│              │ f_mapping ──→ address_space (page cache)│
│              │ f_op ──→ ext4_file_operations           │
│              │ f_flags = O_WRONLY                      │
│              │ f_pos = 8192                            │
│              │                                        │
│  ki_pos = 8192 (write at this byte offset)            │
│  ki_flags = 0 (synchronous write)                     │
│                                                       │
└───────────────────────────────────────────────────────┘

┌─ iov_iter (I/O Data Descriptor) ─────────────────────┐
│                                                       │
│  type = WRITE | ITER_IOVEC (writing from user buffer) │
│  count = 5 (total bytes to write)                     │
│  iov ──→ struct iovec:                                │
│          │ iov_base = 0x7FFE0040 (user buffer addr)   │
│          │ iov_len = 5 (bytes in this segment)        │
│  nr_segs = 1 (one contiguous buffer)                  │
│                                                       │
└───────────────────────────────────────────────────────┘

From these two structures, ext4 can determine:
  WHAT to write:  5 bytes from user address 0x7FFE0040
  WHERE in file:  at byte offset 8192
  WHICH file:     inode 4521 on the ext4 filesystem
  HOW:            synchronous, normal write (not direct I/O)
```


---

# Chapter 8: The inode and address_space — What ext4 Sees

When ext4's write function starts, it looks at the inode and address_space
to understand the file and its cache state:

```
Through kiocb->ki_filp->f_inode, ext4 reaches:

┌─ struct inode (inode 4521) ──────────────────────────────┐
│                                                           │
│  i_ino = 4521              (inode number on this FS)      │
│  i_mode = 0100644          (regular file, rw-r--r--)      │
│  i_size = 10000            (file is 10000 bytes long)     │
│  i_blocks = 24             (24 × 512 = 12288 bytes alloc) │
│                                                           │
│  i_sb ──→ superblock       (ext4 filesystem on /dev/sda1) │
│          │ s_blocksize = 4096                              │
│          │ s_type = ext4_fs_type                           │
│                                                           │
│  i_op ──→ ext4_dir_inode_operations (for directories) or  │
│           ext4_file_inode_operations (for files)           │
│                                                           │
│  i_fop ──→ ext4_file_operations (same as f_op)            │
│                                                           │
│  i_mapping ──→ address_space (THE PAGE CACHE for this file)│
│                                                           │
│  ext4-specific fields (in ext4_inode_info wrapper):       │
│    i_data[0..3] = extent tree root                        │
│    (maps file offsets → physical disk blocks)              │
│                                                           │
└───────────────────────────────────────────────────────────┘

Through i_mapping, ext4 reaches the page cache:

┌─ struct address_space (page cache for inode 4521) ───────┐
│                                                           │
│  host = inode 4521         (which inode I cache)          │
│  nrpages = 3               (3 pages currently cached)     │
│                                                           │
│  i_pages (xarray / radix tree):                           │
│    index 0 → page (bytes 0-4095)       clean              │
│    index 1 → page (bytes 4096-8191)    clean              │
│    index 2 → (NOT CACHED)              ← our write target!│
│    index 3 → (NOT CACHED)                                 │
│                                                           │
│  a_ops ──→ ext4_aops:                                     │
│    .readpage = ext4_readpage                               │
│    .readahead = ext4_readahead                             │
│    .writepage = ext4_writepage                             │
│    .writepages = ext4_writepages                           │
│    .write_begin = ext4_write_begin                         │
│    .write_end = ext4_write_end                             │
│    .direct_IO = ext4_direct_IO                             │
│                                                           │
│  ★ a_ops is ANOTHER operations table!                     │
│    file_operations handles the syscall-level interface.    │
│    address_space_operations handles page-level I/O.        │
│    VFS → file_operations → address_space_operations.       │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

**There are THREE operation tables involved in a write:**

```
Level 1: file_operations (ext4_file_operations)
  → handles the syscall interface
  → write_iter() receives kiocb + iov_iter
  → called by VFS when user calls write()

Level 2: address_space_operations (ext4_aops)
  → handles page cache operations
  → write_begin() prepares a page for writing
  → write_end() marks the page dirty after writing
  → writepages() flushes dirty pages to disk
  → called by the page cache infrastructure

Level 3: inode_operations (ext4_file_inode_operations)
  → handles metadata operations
  → setattr() for truncate, chmod
  → getattr() for stat
  → not directly involved in write data path
```

This three-table architecture means each level of abstraction has its
own pluggable interface. The VFS calls into level 1, level 1 calls into
level 2 through the page cache, and level 2 eventually creates bios that
go to the block layer. Each level only knows about its immediate neighbors.


---

# Chapter 9: What Happens AFTER VFS Dispatches

```
ext4_file_write_iter(&kiocb, &iter) now runs.

It does:
  1. Check if this is a direct I/O write (O_DIRECT flag)
     → No, our file was opened without O_DIRECT
     → Take the BUFFERED write path (through page cache)
  
  2. Call generic_perform_write(file, &iter, pos)
     This is a generic kernel function shared by most filesystems.
     It handles the page cache interaction:
     
     a. Call a_ops->write_begin(file, mapping, pos, len, &page)
        → ext4_write_begin()
        → finds or creates the page at index 2 in the page cache
        → may read existing data from disk (for partial page writes)
        → returns a LOCKED page
     
     b. Copy data from user buffer to the page
        → copy_page_from_iter_atomic(page, offset, len, &iter)
        → internally calls copy_from_user()
        → 5 bytes copied from 0x7FFE0040 to the kernel page
     
     c. Call a_ops->write_end(file, mapping, pos, len, copied, page)
        → ext4_write_end()
        → marks the page as dirty
        → updates inode size if file grew
        → unlocks the page
  
  3. Return bytes written (5) to the VFS
```

This is where Part A of the I/O Pipeline (Page Cache) takes over.
The VFS's job is DONE — it translated fd → file → filesystem → function call,
checked all permissions and limits, wrapped the arguments in standardized
structures (kiocb + iov_iter), and dispatched to ext4. From here, ext4
and the page cache infrastructure handle the actual data movement.


---

# Chapter 10: The Return Path — Back Through the VFS

```
ext4_file_write_iter() returns 5 (bytes written)
  │
  ▼
new_sync_write():
  → updates *ppos = kiocb.ki_pos (8192 + 5 = 8197)
  → returns 5
  │
  ▼
vfs_write():
  → fsnotify_modify(file)     // tell inotify watchers: file was modified
  → add_wchar(current, 5)     // accounting: process wrote 5 bytes
  → inc_syscw(current)        // accounting: process did 1 write syscall
  → returns 5
  │
  ▼
sys_write():
  → f.file->f_pos = 8197     // update the file's position for next write
  → fdput_pos(f)              // release the position lock
  → returns 5
  │
  ▼
syscall return path:
  → stores 5 in RAX (return value to user space)
  → restores all user registers
  → sysretq → back to ring 3
  │
  ▼
Your Rust code:
  → file.write_all(b"Hello")? → Ok(())
```


---

# Chapter 11: The Complete VFS Data Structure Map

```
YOUR CODE
  write(fd=3, buf, 5)
       │
       ▼
  ┌─ Syscall Table ──┐
  │  table[1] =       │
  │  sys_write ───────────────────────────┐
  └───────────────────┘                   │
                                          ▼
                                    ┌─ sys_write() ─┐
                                    │                │
                                    │  fdget_pos(3)  │
                                    │       │        │
                                    └───────┼────────┘
       ┌────────────────────────────────────┘
       ▼
  ┌─ current->files->fdt->fd[3] ──────────────────────────┐
  │                                                        │
  │  struct file (f_op, f_inode, f_mapping, f_pos)         │
  │       │            │             │           │         │
  └───────┼────────────┼─────────────┼───────────┼─────────┘
          │            │             │           │
          ▼            ▼             ▼           ▼
  ┌─ f_op ───┐  ┌─ f_inode ─┐  ┌─ f_mapping ──┐  f_pos=8192
  │ext4_file_│  │inode 4521 │  │address_space  │
  │operations│  │ i_size    │  │ i_pages (tree)│
  │          │  │ i_blocks  │  │ a_ops ──────────→ ext4_aops
  │.write_   │  │ i_sb ───────→ superblock     │    .write_begin
  │  iter ───────→ ext4_file │  │ nrpages=3    │    .write_end
  │.fsync ─────→  _write_   │  │              │    .writepages
  │.read_    │  │  iter()   │  │ page idx 0   │
  │  iter    │  │           │  │ page idx 1   │
  └──────────┘  └───────────┘  │ (idx 2 miss) │
                               └──────────────┘
```


---

# Summary: What the VFS Actually Did

```
INPUT:  fd=3, buf=0x7FFE0040, count=5

VFS did these things:
  1. TRANSLATE:  fd 3 → struct file (4 pointer dereferences)
  2. CHECK:      5 permission/validity checks
  3. WRAP:       user buffer → iov_iter, file+pos → kiocb
  4. DISPATCH:   file->f_op->write_iter(kiocb, iov_iter)
  5. ACCOUNT:    update f_pos, fsnotify, byte counters

VFS did NOT:
  ✗ Copy any file data (the filesystem does that)
  ✗ Touch the page cache (the filesystem does that)
  ✗ Know which filesystem it's talking to (Strategy Pattern)
  ✗ Know what ext4, XFS, or NFS do internally
  ✗ Change the page table or TLB

Time spent in VFS: ~10-20 nanoseconds
Time spent in ext4 + page cache: ~50-100 nanoseconds
Time spent in disk I/O (if needed): ~20,000-8,000,000 nanoseconds

The VFS is a thin dispatch layer — it validates, wraps, and routes.
The real work happens in the filesystem and below.
```
