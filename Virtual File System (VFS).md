# Mastering the Virtual File System (VFS)

## From Path Strings to Disk Blocks — The Complete Guide

---

# Chapter 1: Why the VFS Exists

## 1.1 The Problem It Solves

Imagine you're the Linux kernel. A program calls:

```c
int fd = open("/mnt/usb/photos/cat.jpg", O_RDONLY);
```

You need to answer several questions:
- What filesystem is `/mnt/usb` mounted on? (FAT32? NTFS? ext4?)
- How do I translate `photos` into a location on that filesystem?
- How do I read `cat.jpg`'s data blocks from the USB drive?
- What if another program opens the same file simultaneously?

Without VFS, every program would need to know how to speak FAT32, ext4,
NFS, tmpfs, procfs, and every other filesystem. That's insane.

The VFS is an **abstraction layer** — a set of interfaces (like a Rust
trait or a C# interface) that every filesystem must implement. The kernel
talks to the VFS. The VFS talks to the specific filesystem driver.

```
User Program
    │
    │  open("/mnt/usb/photos/cat.jpg", O_RDONLY)
    ▼
┌──────────────────────────────────────────────┐
│              SYSTEM CALL LAYER                │
│   (translates userspace call → kernel call)   │
└──────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────┐
│                                              │
│            VIRTUAL FILE SYSTEM               │
│                                              │
│  Path Resolution:   "/" → "mnt" → "usb" →   │
│                     "photos" → "cat.jpg"     │
│                                              │
│  Core Objects:                               │
│    superblock ─ mount point metadata         │
│    inode      ─ file metadata (THE file)     │
│    dentry     ─ directory entry (name→inode) │
│    file       ─ open file instance           │
│                                              │
│  Operation Tables:                           │
│    inode_operations   ─ create, lookup, link  │
│    file_operations    ─ read, write, mmap    │
│    super_operations   ─ mount, umount, sync  │
│    dentry_operations  ─ compare, hash        │
│                                              │
└──────────────────────────────────────────────┘
    │                         │                │
    ▼                         ▼                ▼
┌──────────┐         ┌──────────┐      ┌──────────┐
│   ext4   │         │  FAT32   │      │   NFS    │
│  driver  │         │  driver  │      │  client  │
└──────────┘         └──────────┘      └──────────┘
    │                     │                │
    ▼                     ▼                ▼
 Local SSD            USB Drive         Network
```

This is the **Strategy Pattern** at the operating system level — the same
pattern you've been applying in your banking software refactoring. The VFS
defines the interface, each filesystem provides the implementation.


## 1.2 The Four Core Objects

Everything in the VFS revolves around four data structures. Understanding
these four objects IS understanding the VFS.

### Object 1: superblock

One per mounted filesystem. Contains global metadata about the filesystem.

```c
struct super_block {
    struct list_head    s_list;        // List of all superblocks
    dev_t               s_dev;         // Device identifier
    unsigned long       s_blocksize;   // Block size in bytes (e.g., 4096)
    unsigned char       s_blocksize_bits; // log2(blocksize)
    loff_t              s_maxbytes;    // Max file size
    struct file_system_type *s_type;   // ext4, xfs, btrfs...
    struct super_operations *s_op;     // ★ Operations table
    struct dentry       *s_root;       // Root dentry of this mount
    // ...
};
```

Think of it as: "I am an ext4 filesystem on /dev/sda1, mounted at /home,
with 4KB blocks and a max file size of 16TB."

### Object 2: inode (index node)

One per file (or directory, or symlink, or device, or socket...).
The inode IS the file. The filename is just a label pointing to an inode.

```c
struct inode {
    umode_t             i_mode;     // File type + permissions (rwxrwxrwx)
    kuid_t              i_uid;      // Owner user ID
    kgid_t              i_gid;      // Owner group ID
    unsigned int        i_flags;    // Filesystem flags
    struct inode_operations *i_op;  // ★ Operations (lookup, create, mkdir...)
    struct super_block  *i_sb;      // Which filesystem I belong to
    unsigned long       i_ino;      // Inode number (unique within FS)
    loff_t              i_size;     // File size in bytes
    struct timespec64   i_atime;    // Last access time
    struct timespec64   i_mtime;    // Last modification time
    struct timespec64   i_ctime;    // Last status change time
    unsigned int        i_nlink;    // Hard link count
    blkcnt_t            i_blocks;   // Number of 512-byte blocks allocated
    struct file_operations *i_fop;  // ★ File operations (read, write...)
    struct address_space *i_mapping; // Page cache for this inode
    // ...
};
```

**Critical insight**: The inode has NO name. The name lives in the dentry.
This is why hard links work — multiple names (dentries) can point to the
same inode. The inode doesn't know or care what its name is.

### Object 3: dentry (directory entry)

Maps a name to an inode. One per path component.

```c
struct dentry {
    struct dentry        *d_parent;   // Parent directory's dentry
    struct qstr          d_name;      // This component's name ("photos")
    struct inode         *d_inode;    // The inode this name points to
    struct super_block   *d_sb;       // Which filesystem
    struct dentry_operations *d_op;   // ★ Operations (compare, hash...)
    struct list_head     d_child;     // Siblings in parent directory
    struct list_head     d_subdirs;   // My children (if I'm a directory)
    // ...
};
```

For the path `/mnt/usb/photos/cat.jpg`, there are 5 dentries:

```
dentry: "/"        → inode 2     (root of /)
dentry: "mnt"      → inode 1048  (directory)
dentry: "usb"      → inode 5001  (mount point → different superblock!)
dentry: "photos"   → inode 22    (directory on USB filesystem)
dentry: "cat.jpg"  → inode 87    (regular file on USB filesystem)
```

### Object 4: file (open file description)

Created when a process opens a file. One per `open()` call.

```c
struct file {
    struct path          f_path;      // dentry + mount info
    struct inode         *f_inode;    // The inode (shortcut)
    struct file_operations *f_op;     // ★ Operations (read, write...)
    loff_t               f_pos;       // Current read/write offset
    unsigned int         f_flags;     // O_RDONLY, O_WRONLY, O_RDWR...
    fmode_t              f_mode;      // FMODE_READ, FMODE_WRITE
    struct address_space *f_mapping;  // Page cache mapping
    // ...
};
```

The file object is EPHEMERAL — it exists only while the file is open.
The inode is PERSISTENT — it exists as long as the file exists on disk.
Multiple file objects can point to the same inode (multiple opens of
the same file).

## 1.3 The Relationship Between Objects

```
Process A: fd 3 ──→ file object ──┐
                     (offset=100)  │
                                   ├──→ dentry "cat.jpg" ──→ inode 87
Process B: fd 5 ──→ file object ──┘         │                 │
                     (offset=500)            │              belongs to
                                             │                 │
                                    child of dentry            ▼
                                             │            superblock
                                     dentry "photos"      (ext4 on
                                             │             /dev/sda1)
                                     dentry "mnt"
                                             │
                                     dentry "/"
```

Each process has its own file object (its own offset), but they share
the same dentry and inode. This is how concurrent file access works.


---

# Chapter 2: Path Resolution — The Kernel's Walk

## 2.1 What Happens Step by Step

When you call `open("/mnt/usb/photos/cat.jpg", O_RDONLY)`, the kernel
performs **path resolution** — also called a "path walk" or "namei"
(name-to-inode).

```
Step 1: Start at the root dentry "/"
        Current inode: 2 (root inode, always 2 on ext4/most FSes)

Step 2: Look up "mnt" in the root directory
        → Call inode_operations->lookup(root_inode, "mnt")
        → Filesystem reads its directory data structure
        → Returns inode 1048
        → Create/find dentry: "mnt" → inode 1048

Step 3: Look up "usb" in directory inode 1048
        ★ MOUNT POINT CHECK: is anything mounted on this dentry?
        → Yes! A FAT32 filesystem is mounted here
        → CROSS MOUNT BOUNDARY: switch to FAT32's superblock
        → New root dentry is FAT32's root
        → Current inode: FAT32's root inode

Step 4: Look up "photos" on FAT32
        → Call FAT32's inode_operations->lookup(root, "photos")
        → FAT32 reads its directory table (different format than ext4!)
        → Returns FAT32 inode for the "photos" directory

Step 5: Look up "cat.jpg" in "photos"
        → Call FAT32's inode_operations->lookup(photos_inode, "cat.jpg")
        → Returns FAT32 inode for cat.jpg
        → Create file object, link to this inode
        → Return fd to userspace
```

**The key insight**: At step 3, the path walk CROSSES a filesystem boundary.
The VFS transparently switches from ext4's `lookup()` to FAT32's `lookup()`.
The user program has no idea this happened.

## 2.2 The Dentry Cache (dcache)

Path resolution is expensive — it requires reading directory data from disk
for every component. So the kernel caches dentries aggressively.

```
Dentry Cache (dcache) — a hash table in kernel memory

  hash("cat.jpg", parent_dentry) → cached dentry → inode 87

  hash("photos", parent_dentry)  → cached dentry → inode 22
```

The dcache is one of the most performance-critical data structures in the
kernel. A path lookup that hits the dcache for every component requires
ZERO disk I/O — it's just hash lookups in memory.

**Negative dentries**: The dcache also caches "this name does NOT exist."
If you call `open("/tmp/nonexistent")` and it fails, the kernel caches a
negative dentry for "nonexistent." The next time you try, the kernel knows
immediately it doesn't exist without consulting the filesystem.

## 2.3 The Anatomy of lookup()

Each filesystem implements `lookup()` differently, because each filesystem
stores directories differently:

**ext4** stores directories as:
- Small directories: inline data in the inode itself
- Medium directories: a linked list of directory entry blocks
- Large directories: an HTree (hash tree, similar to a B-tree)

**FAT32** stores directories as:
- A flat table of 32-byte entries in sequential clusters

**XFS** stores directories as:
- Small: inline in the inode
- Medium: a single leaf block
- Large: a B+ tree of directory entries

**Your storage engine**: Your B+ tree IS a directory, conceptually.
When the VFS calls `lookup("cat.jpg")`, the filesystem searches its
directory data structure — which for ext4 is an HTree, for your engine
would be a B+ tree keyed on filenames.

## 2.4 Symbolic Links and Path Resolution

When the path walk encounters a symbolic link:

```
/home/user/link → /data/actual_file

Path walk for /home/user/link:
  1. Resolve "/" → "home" → "user" → "link"
  2. Read inode for "link" → it's a symlink!
  3. Read symlink target: "/data/actual_file"
  4. ★ RESTART path walk from "/" for "/data/actual_file"
  5. Resolve "/" → "data" → "actual_file"
```

The kernel limits symlink resolution depth to prevent infinite loops
(typically 40 levels). This is why `ELOOP` error exists.

## 2.5 Mount Points — Crossing Filesystem Boundaries

A mount point is a dentry that has a different filesystem "stacked" on top:

```
Before mount:
  /mnt/usb/  → inode 5001 on ext4 (empty directory)

After mount:
  /mnt/usb/  → inode 5001 on ext4 COVERED BY
               root inode on FAT32 (from /dev/sdb1)
```

During path resolution, the kernel checks each dentry: "Is there a
filesystem mounted on top of this?" If yes, it crosses to the mounted
filesystem's root. The original directory inode is "covered" — you can't
access it until the filesystem is unmounted.

```c
// Kernel code (simplified) for crossing mount points:
struct dentry *lookup_one(struct dentry *parent, const char *name) {
    struct dentry *child = d_lookup(parent, name);  // dcache lookup

    if (!child) {
        child = parent->d_inode->i_op->lookup(parent->d_inode, name);
    }

    // ★ Check for mount point
    while (d_mountpoint(child)) {
        struct vfsmount *mnt = lookup_mnt(child);
        child = mnt->mnt_root;  // Cross to mounted FS root
    }

    return child;
}
```

## 2.6 Path Resolution Flags

The `open()` system call takes flags that affect path resolution:

| Flag | Effect on path walk |
|------|-------------------|
| O_NOFOLLOW | Don't follow final symlink (error if it IS a symlink) |
| O_DIRECTORY | Error if final component isn't a directory |
| O_CREAT | If final component doesn't exist, CREATE it |
| O_EXCL | With O_CREAT: error if file already exists |
| O_PATH | Don't actually open; just get an fd for the path itself |

For O_CREAT, the path walk resolves all components EXCEPT the last one,
then calls `inode_operations->create()` on the parent directory to create
a new inode and dentry.


---

# Chapter 3: The Inode — The True Identity of a File

## 3.1 What an Inode Stores (On Disk)

An ext4 inode on disk (256 bytes, the actual structure):

```
┌────────────────────────────────────────────────────┐
│ Offset  Field              Size   Example          │
├────────────────────────────────────────────────────┤
│ 0x00    i_mode             2B     0100644 (regular,│
│                                   rw-r--r--)       │
│ 0x02    i_uid_lo           2B     1000             │
│ 0x04    i_size_lo          4B     4096             │
│ 0x08    i_atime            4B     1709251200       │
│ 0x0C    i_ctime            4B     1709251200       │
│ 0x10    i_mtime            4B     1709251200       │
│ 0x14    i_dtime            4B     0 (not deleted)  │
│ 0x18    i_gid_lo           2B     1000             │
│ 0x1A    i_links_count      2B     1                │
│ 0x1C    i_blocks_lo        4B     8 (× 512B = 4KB)│
│ 0x20    i_flags            4B     0x80000 (extents)│
│ 0x28    i_block[0..14]     60B    ★ THE DATA MAP   │
│ ...     (more fields)                              │
│ 0x82    i_extra_isize      2B     32               │
│ 0x88    i_crtime            4B    creation time     │
│ ...     (extended attributes, ACLs)                │
└────────────────────────────────────────────────────┘
```

The 60 bytes at `i_block` are where it gets interesting. In ext4, this
space is used as an **extent tree** — a compact B-tree that maps logical
file offsets to physical disk block numbers.

## 3.2 How Inodes Map to Disk Blocks

The whole point of an inode is to answer: "Give me byte offset X of this
file — which disk block is it in?"

**ext4 extents** (modern):

```
An extent: "Logical blocks 0-99 are at physical blocks 50000-50099"

┌──────────────────────────────────────────────┐
│ Extent Header                                 │
│   magic: 0xF30A                              │
│   entries: 2                                 │
│   depth: 0 (leaf node)                       │
├──────────────────────────────────────────────┤
│ Extent 1:                                    │
│   logical start: 0                           │
│   length: 100 blocks                         │
│   physical start: 50000                      │  → Contiguous!
├──────────────────────────────────────────────┤
│ Extent 2:                                    │
│   logical start: 100                         │
│   length: 50 blocks                          │
│   physical start: 80000                      │  → Separate area
└──────────────────────────────────────────────┘

File layout on disk:
  Bytes 0-409599      → disk blocks 50000-50099  (400KB contiguous)
  Bytes 409600-614399 → disk blocks 80000-80049  (200KB contiguous)
```

For a small file with ≤4 extents, the entire extent tree fits in the
inode's 60-byte `i_block` area. For larger files, it becomes a real
B-tree with internal nodes pointing to leaf blocks on disk.

**This is directly analogous to your B+ tree**: the inode's extent tree
is a specialized B+ tree optimized for mapping (logical offset → physical
block). Your storage engine's B+ tree maps (key → value). Same data
structure, different domain.

## 3.3 Inode Numbers and the Inode Table

Every inode has a unique number within its filesystem. The filesystem
stores all inodes in a predictable location — the **inode table**.

```
ext4 disk layout:

Block Group 0:
┌──────────┬──────────┬──────────┬──────────┬──────────────────┐
│ Super    │ Group    │ Block    │ Inode    │ Inode Table      │
│ Block    │ Desc.    │ Bitmap   │ Bitmap   │ (inodes 1-8192)  │
└──────────┴──────────┴──────────┴──────────┴──────────────────┘

Block Group 1:
┌──────────┬──────────┬──────────┬──────────┬──────────────────┐
│ Super    │ Group    │ Block    │ Inode    │ Inode Table      │
│ Block    │ Desc.    │ Bitmap   │ Bitmap   │ (inodes 8193+)   │
│ (backup) │          │          │          │                  │
└──────────┴──────────┴──────────┴──────────┴──────────────────┘
```

To read inode N:
1. Calculate which block group: `group = (N - 1) / inodes_per_group`
2. Calculate offset within group: `offset = (N - 1) % inodes_per_group`
3. Read the inode table block at: `inode_table_start + offset * inode_size`

This is a constant-time operation — O(1) to go from inode number to inode data.

**Special inode numbers in ext4:**

| Number | Purpose |
|--------|---------|
| 1 | Bad blocks list |
| 2 | Root directory (always!) |
| 7 | Reserved for group descriptors |
| 8 | Journal |
| 11 | First non-reserved inode (lost+found) |

## 3.4 The Inode Cache (icache)

Just like the dcache caches dentries, the kernel caches inodes in memory.
When you access a file, its inode is read from disk once and kept in the
inode cache. Subsequent accesses use the cached copy.

```
Inode Cache:
  (device, inode_number) → cached struct inode

  (/dev/sda1, 87) → { size=45000, mode=0644, blocks=12, ... }
```

The inode cache is indexed by (device, inode_number), so the same inode
number on different filesystems doesn't collide.

## 3.5 Inode Lifetime

```
File created:
  1. Filesystem allocates inode on disk (sets inode bitmap)
  2. VFS creates in-memory inode (icache)
  3. VFS creates dentry linking name → inode
  4. dentry goes into dcache

File opened:
  1. Path resolution walks dentries → finds inode
  2. VFS creates file object → links to inode
  3. inode's reference count incremented

File closed:
  1. file object destroyed
  2. inode's reference count decremented
  3. If refcount > 0: inode stays in icache
  4. If refcount == 0 AND nlink > 0: inode stays in icache (can be evicted)
  5. If refcount == 0 AND nlink == 0: inode is DELETED from disk

File deleted (rm):
  1. dentry removed from parent directory
  2. inode's nlink decremented
  3. If nlink > 0: other hard links still exist, inode stays
  4. If nlink == 0 AND refcount > 0: file "deleted" but data stays
     (process still has it open!)
  5. If nlink == 0 AND refcount == 0: inode freed, blocks returned to free space
```

**The classic puzzle**: "I deleted a huge log file but disk space wasn't freed."
Answer: Some process still has the file open (refcount > 0). The inode
won't be freed until that process closes the file descriptor. Use `lsof`
to find which process.


---

# Chapter 4: How Directories Work

## 4.1 A Directory Is Just a File

A directory is a special file whose "data" is a list of (name, inode_number)
pairs. The VFS treats it as a file — it has an inode, it has data blocks —
but the data format is defined by the filesystem.

```
Directory "photos" on ext4 (simplified):

Offset  Inode   Rec_len  Name_len  Type   Name
0       22      12       1         DIR    "."       (self)
12      5001    12       2         DIR    ".."      (parent)
24      87      20       7         FILE   "cat.jpg"
44      88      16       6         FILE   "dog.png"
60      89      24       10        FILE   "sunset.raw"
84      0       3932     0         -      (free space to end of block)
```

Each entry has:
- **inode number**: which inode this name points to
- **rec_len**: total length of this entry (for traversal)
- **name_len**: length of the name string
- **type**: file, directory, symlink, etc. (avoids reading the inode just to check type)

## 4.2 Directory Lookup Implementations

The performance of `lookup()` — finding a name in a directory — varies
enormously by filesystem and directory size:

**ext4 small directories** (linear scan):
```
For a directory with 10 entries:
  Read the directory data block(s)
  Scan entry by entry: compare name with each entry
  O(N) — fine for small directories
```

**ext4 large directories** (HTree / directory index):
```
For a directory with 10,000 entries:
  Hash the name: hash("cat.jpg") → 0x4A3B2C1D
  Walk the HTree (a 2-level hash tree):
    Level 1: hash → which leaf block
    Level 2: scan leaf block for exact match
  O(1) amortized — constant time!
```

**XFS directories** (B+ tree):
```
For a directory with any number of entries:
  Search the B+ tree keyed on filename hash
  O(log N)
```

**Your storage engine parallel**: When your engine does a point lookup
(`db.get("account_1001")`), it walks a B+ tree. When ext4 does a
directory lookup, it walks an HTree. Same concept, different shape.

## 4.3 Creating a File — The Full Stack

When you call `open("/data/newfile.txt", O_CREAT | O_WRONLY, 0644)`:

```
1. VFS: Path walk resolves "/data/" → gets parent directory inode

2. VFS: Calls ext4's inode_operations->lookup(parent, "newfile.txt")
   → NOT FOUND (returns negative dentry)

3. VFS: Since O_CREAT is set, calls ext4's inode_operations->create()
   a. ext4 allocates a new inode number (scans inode bitmap)
   b. ext4 initializes the inode on disk (mode=0644, size=0, uid, gid)
   c. ext4 adds entry to parent directory's data blocks
   d. ext4 updates parent directory's mtime
   e. ext4 journals all these changes (ext4 uses journaling!)

4. VFS: Creates in-memory inode, dentry, file objects
   → Links them together
   → Returns fd to userspace

5. Total disk writes (with journaling):
   - Journal: inode bitmap, inode data, directory block, parent inode
   - Commit block in journal
   - Later: actual inode bitmap, inode data, directory block, parent inode
   - That's ~8 block writes for creating one empty file!
```

This is why creating many small files is slow — each file creation
requires multiple metadata writes and journal commits.


---

# Chapter 5: The Operations Tables — The VFS Interfaces

## 5.1 super_operations (Filesystem-Level)

Called on the filesystem as a whole:

```c
struct super_operations {
    // Allocate a new in-memory inode
    struct inode *(*alloc_inode)(struct super_block *sb);

    // Free an in-memory inode
    void (*destroy_inode)(struct inode *inode);

    // Write a dirty inode to disk
    int (*write_inode)(struct inode *inode, struct writeback_control *wbc);

    // Delete an inode from disk (nlink reached 0)
    void (*evict_inode)(struct inode *inode);

    // Sync filesystem metadata to disk
    int (*sync_fs)(struct super_block *sb, int wait);

    // Provide filesystem statistics (df command)
    int (*statfs)(struct dentry *dentry, struct kstatfs *buf);

    // Called on mount
    int (*remount_fs)(struct super_block *sb, int *flags, char *data);
};
```

**Your storage engine analogy:**
- `alloc_inode` → your page manager's `allocate_page()`
- `write_inode` → your buffer pool's `flush_page()`
- `sync_fs` → your checkpoint operation
- `evict_inode` → your page manager's `free_page()`

## 5.2 inode_operations (File-Metadata-Level)

Called to manipulate files/directories by name:

```c
struct inode_operations {
    // ★ Look up a name in a directory → return dentry
    struct dentry *(*lookup)(struct inode *dir, struct dentry *dentry,
                             unsigned int flags);

    // Create a new regular file
    int (*create)(struct inode *dir, struct dentry *dentry,
                  umode_t mode, bool excl);

    // Create a hard link
    int (*link)(struct dentry *old_dentry, struct inode *dir,
                struct dentry *new_dentry);

    // Remove a name (unlink for files, rmdir for directories)
    int (*unlink)(struct inode *dir, struct dentry *dentry);
    int (*rmdir)(struct inode *dir, struct dentry *dentry);

    // Create a symbolic link
    int (*symlink)(struct inode *dir, struct dentry *dentry, const char *name);

    // Create a directory
    int (*mkdir)(struct inode *dir, struct dentry *dentry, umode_t mode);

    // Rename a file
    int (*rename)(struct inode *old_dir, struct dentry *old_dentry,
                  struct inode *new_dir, struct dentry *new_dentry,
                  unsigned int flags);

    // Get file attributes
    int (*getattr)(const struct path *path, struct kstat *stat,
                   u32 request_mask, unsigned int flags);

    // Set file attributes (chmod, chown, truncate)
    int (*setattr)(struct dentry *dentry, struct iattr *attr);
};
```

**The critical one is `lookup()`** — it's called for every path component
during path resolution. It must be fast.

## 5.3 file_operations (Open-File-Level)

Called on an open file descriptor — the data plane:

```c
struct file_operations {
    // Read data from file
    ssize_t (*read)(struct file *f, char __user *buf,
                    size_t count, loff_t *offset);

    // Write data to file
    ssize_t (*write)(struct file *f, const char __user *buf,
                     size_t count, loff_t *offset);

    // Read using io vectors (scatter-gather)
    ssize_t (*read_iter)(struct kiocb *iocb, struct iov_iter *to);

    // Write using io vectors
    ssize_t (*write_iter)(struct kiocb *iocb, struct iov_iter *from);

    // Memory map the file
    int (*mmap)(struct file *f, struct vm_area_struct *vma);

    // Open callback (when file is opened)
    int (*open)(struct inode *inode, struct file *f);

    // Flush (called on every close(), even if other fds remain)
    int (*flush)(struct file *f, fl_owner_t id);

    // Release (called when last reference is closed)
    int (*release)(struct inode *inode, struct file *f);

    // Sync file data to disk
    int (*fsync)(struct file *f, loff_t start, loff_t end, int datasync);

    // Change file offset
    loff_t (*llseek)(struct file *f, loff_t offset, int whence);

    // ioctl — filesystem-specific commands
    long (*unlocked_ioctl)(struct file *f, unsigned int cmd, unsigned long arg);

    // Pre-allocate space
    long (*fallocate)(struct file *f, int mode, loff_t offset, loff_t len);
};
```

**Your storage engine analogy:**
- `read`/`write` → your `BufferPool::fetch_page()` + `PageHandle::data()`
- `fsync` → your WAL flush + checkpoint
- `mmap` → your (optional) memory-mapped read path
- `open` → your `Database::open()`
- `release` → your `Database::close()` (flush dirty pages)
- `fallocate` → your `PageManager::allocate_page()` (pre-allocate space)

## 5.4 How a read() Call Flows Through VFS

Complete flow of `pread(fd, buf, 4096, 0)`:

```
1. User space: pread(fd=3, buf, 4096, 0)

2. Syscall layer:
   → Validate fd
   → Get struct file from fd table
   → Verify F_MODE_READ

3. VFS layer (vfs_read):
   → Check file_operations->read or ->read_iter exists
   → Call security_file_permission() (SELinux/AppArmor check)
   → Call file->f_op->read_iter()

4. Filesystem (ext4_file_read_iter):
   → Compute which logical blocks are needed
   → Check page cache (address_space):
       → Page found in cache? Return it directly (FAST PATH)
       → Page NOT in cache? Continue to step 5

5. ext4 block mapping:
   → Read inode's extent tree
   → Map logical block 0 → physical block 50000
   → Submit I/O request to block layer

6. Block layer:
   → Merge/schedule the I/O request
   → Submit to device driver

7. Device driver → disk → data read into page cache

8. VFS: copy_to_user(buf, page_cache_data, 4096)
   → Copy data from kernel page cache to user buffer

9. Return 4096 to user space
```

**The page cache is step 4** — if the data is already cached, steps 5-7
are skipped entirely. This is why re-reading the same file is fast.


---

# Chapter 6: Special Filesystems — VFS Beyond Disk

## 6.1 procfs (/proc)

`/proc` is not on any disk. It's a virtual filesystem where each "file"
is generated on-the-fly by kernel code.

```
/proc/self/status → kernel generates process status text
/proc/meminfo     → kernel generates memory statistics
/proc/self/fd/    → directory listing of open file descriptors
```

procfs implements the VFS interfaces, but `read()` calls a C function
that generates the content instead of reading from a block device:

```c
// Simplified procfs read for /proc/meminfo
static ssize_t meminfo_read(struct file *f, char __user *buf,
                            size_t count, loff_t *pos) {
    char *page = (char *)__get_free_page(GFP_KERNEL);
    int len = sprintf(page, "MemTotal: %lu kB\n", totalram_pages * 4);
    copy_to_user(buf, page, len);
    return len;
}
```

**The VFS doesn't care that there's no disk.** It calls `file_operations->read()`
and gets bytes back. The strategy pattern at work.

## 6.2 tmpfs (/tmp, /dev/shm)

tmpfs stores files entirely in the page cache — RAM only, no disk backing.
It uses the VFS's own page cache as its "disk."

Writes go to the page cache and STAY there (never flushed to disk because
there's no disk). Data is lost on reboot. But it implements the full VFS
interface — you can create directories, hard links, symlinks, chmod, etc.

## 6.3 FUSE (Filesystem in Userspace)

FUSE lets YOU implement a filesystem as a regular program. The kernel's
FUSE module implements the VFS interfaces and forwards operations to your
userspace daemon via a special `/dev/fuse` device file.

```
User app: open("/mnt/myfs/file.txt")
    → Kernel VFS: path resolution
    → Kernel FUSE module: "I handle /mnt/myfs"
    → /dev/fuse: sends LOOKUP request to your daemon
    → Your daemon: looks up "file.txt" in your data structure
    → /dev/fuse: sends response back to kernel
    → Kernel: creates inode, dentry, file objects
    → User app: gets fd

User app: read(fd, buf, 4096)
    → Kernel FUSE: sends READ request to your daemon
    → Your daemon: reads data from wherever (database? network? thin air?)
    → Kernel FUSE: copies data to user
```

**This is directly relevant to your storage engine**: You could expose
your ferrisdb as a FUSE filesystem. Users would `ls` to list keys and
`cat` to read values. It's not practical for production, but it's a
powerful demonstration and debugging tool.

## 6.4 The Filesystem Registration

Every filesystem registers itself with the kernel:

```c
// ext4 module initialization
static struct file_system_type ext4_fs_type = {
    .owner    = THIS_MODULE,
    .name     = "ext4",
    .mount    = ext4_mount,          // Called when you mount an ext4 partition
    .kill_sb  = kill_block_super,    // Called on unmount
    .fs_flags = FS_REQUIRES_DEV,    // Needs a block device
};

static int __init ext4_init(void) {
    register_filesystem(&ext4_fs_type);
    return 0;
}
```

When you run `mount -t ext4 /dev/sda1 /mnt`, the kernel:
1. Finds the registered `ext4_fs_type`
2. Calls `ext4_mount()` → reads the superblock from `/dev/sda1`
3. Creates a `super_block` structure with ext4's operations tables
4. Attaches it to the dentry at `/mnt`


---

# Chapter 7: VFS Internals That Affect Your Storage Engine

## 7.1 The Address Space and Page Cache

Every inode has an `address_space` — this IS the page cache for that file.

```c
struct address_space {
    struct inode        *host;       // The inode I cache data for
    struct xarray       i_pages;     // Radix tree of cached pages
    unsigned long       nrpages;     // Number of cached pages
    struct address_space_operations *a_ops;  // ★ readpage, writepage
};

struct address_space_operations {
    // Read a page from disk into cache
    int (*readpage)(struct file *f, struct page *page);

    // Write a dirty page from cache to disk
    int (*writepage)(struct page *page, struct writeback_control *wbc);

    // Write multiple pages (batch)
    int (*writepages)(struct address_space *mapping,
                      struct writeback_control *wbc);

    // Prepare for a write (allocate blocks if needed)
    int (*write_begin)(struct file *f, struct address_space *mapping,
                       loff_t pos, unsigned len, unsigned flags,
                       struct page **pagep, void **fsdata);

    // Finish a write
    int (*write_end)(struct file *f, struct address_space *mapping,
                     loff_t pos, unsigned len, unsigned copied,
                     struct page *page, void *fsdata);
};
```

**This is the kernel's buffer pool.** Your `BufferPool` is doing the same
thing as `address_space` — caching pages in RAM, tracking which are dirty,
deciding which to evict, and flushing dirty pages to disk.

The difference: `address_space` uses the kernel's generic LRU. Your buffer
pool uses LRU-K, which is smarter about which pages to keep.

## 7.2 How fsync() Really Works

When your storage engine calls `file.sync_all()`, here's the full stack:

```
1. Rust: file.sync_all()
2. libc: fsync(fd)
3. Kernel syscall handler: do_fsync(fd)
4. VFS: vfs_fsync(file, 0)  // 0 = sync data AND metadata
5. file->f_op->fsync(file, 0, LLONG_MAX, 0)
   → This calls the FILESYSTEM's fsync, e.g., ext4_sync_file()

6. ext4_sync_file():
   a. Write all dirty pages for this inode: filemap_write_and_wait()
      → Iterates address_space->i_pages
      → Calls writepage() for each dirty page
      → Waits for all I/O to complete
   b. Flush the ext4 journal (if this inode has journal entries)
      → jbd2_complete_transaction()
   c. Issue FLUSH CACHE command to the disk
      → blkdev_issue_flush(sb->s_bdev)
      → This tells the drive controller: "Write your internal cache to media NOW"

7. Returns to userspace only when ALL data is on stable storage
```

**The FLUSH CACHE command in step 6c** is critical. Without it, data could
be in the drive's volatile write cache, which is lost on power failure.
Enterprise SSDs with battery-backed caches make this a no-op (the cache
IS stable storage). Consumer SSDs might take 50μs-1ms for the flush.

## 7.3 File Locking and the VFS

The VFS provides two locking mechanisms:

**Advisory locks** (flock / fcntl): Cooperative — only processes that check
for locks will see them. Other processes can still read/write the file.

```c
// Advisory lock — other processes can ignore it
flock(fd, LOCK_EX);  // Exclusive lock
flock(fd, LOCK_SH);  // Shared lock
```

**Mandatory locks** (rare, mostly deprecated): Enforced by the kernel.
Any process that tries to read/write a locked region will block.

**For your storage engine**: Don't rely on file locks for concurrency.
Use your own lock manager (which you're building in Phase 3). File locks
are too coarse-grained — they lock the entire file or byte ranges, not
individual pages or records.

## 7.4 Extended Attributes (xattrs)

Files can have arbitrary key-value metadata beyond the standard inode fields:

```bash
# Set a custom attribute
setfattr -n user.mydb.version -v "2" database.db

# Read it
getfattr -n user.mydb.version database.db
```

**Storage engine use case**: You could store your database's format version,
page size, or other metadata as xattrs on the database file. This way,
tools can inspect the database without parsing the file format:

```rust
use std::os::unix::fs;
// Hypothetical — Rust doesn't have xattr in std
xattr::set("database.db", "user.ferrisdb.version", b"1")?;
xattr::set("database.db", "user.ferrisdb.page_size", b"4096")?;
```

## 7.5 The VFS and Your Storage Engine — The Complete Picture

Here's how everything maps together:

```
VFS Concept              Your Storage Engine Equivalent
─────────────────────    ─────────────────────────────
superblock               Database file header (page 0, meta page)
inode                    B+ tree node / heap page header
dentry + dcache          In-memory key→page_id cache
file object              Transaction / cursor handle
address_space            Buffer pool
writepage()              BufferPool::flush_page()
readpage()               BufferPool::fetch_page()
inode_operations->lookup B+ tree search
inode_operations->create B+ tree insert
file_operations->read    Cursor::next() / db.get()
file_operations->write   db.put()
file_operations->fsync   WAL flush + checkpoint
super_operations->sync   Full database checkpoint
ext4 journal             Your write-ahead log
extent tree              Your B+ tree
inode table              Your page manager's page allocation
block bitmap             Your free page list
```

Every concept in the VFS has a direct parallel in your storage engine.
The kernel's filesystem is a database. Your database is a filesystem.
They're the same thing at different levels of abstraction.

---

# VFS Quick Reference

## The Four Objects

| Object | Kernel struct | Per-what | Lifetime | Your engine equivalent |
|--------|--------------|----------|----------|----------------------|
| superblock | `super_block` | Per mounted FS | Mount → unmount | Database file header (page 0) |
| inode | `inode` | Per file | File exists on disk | B+ tree node / page header |
| dentry | `dentry` | Per name | Cached in dcache | Key → PageId lookup cache |
| file | `file` | Per open() call | open() → close() | Transaction / cursor handle |

## The Three Operation Tables

| Table | Scope | Key methods | Your engine equivalent |
|-------|-------|-------------|----------------------|
| super_operations | Filesystem | sync_fs, statfs, alloc_inode | checkpoint, db stats, allocate page |
| inode_operations | Name/metadata | lookup, create, unlink, mkdir | B+ tree search, insert, delete |
| file_operations | Open file data | read, write, fsync, mmap | cursor read, put, WAL flush |

## Path Resolution Algorithm

```
input:  "/mnt/data/reports/Q3.pdf"
output: inode for Q3.pdf

1. current = root dentry ("/", inode 2)
2. for each component [mnt, data, reports, Q3.pdf]:
   a. check dcache(current_inode, component_name)
      → hit? use cached dentry, skip to (e)
      → miss? continue to (b)
   b. call current_fs->inode_operations->lookup(current_inode, name)
   c. filesystem reads its directory structure (htree/btree/linear)
   d. cache result in dcache (even if not found → negative dentry)
   e. check: is this dentry a mount point?
      → yes? cross to mounted FS's root dentry
   f. check: is this a symbolic link?
      → yes? restart walk from symlink target (depth limit: 40)
   g. current = resolved dentry
3. return current.inode
```

## Observing VFS from Userspace

| What to inspect | Where to look | Command |
|----------------|---------------|---------|
| Inode of a file | stat() syscall | `stat filename` or `ls -i` |
| All mount points | /proc/mounts | `cat /proc/mounts` |
| Open fds | /proc/self/fd/ | `ls -la /proc/self/fd/` |
| fd details | /proc/self/fdinfo/N | `cat /proc/self/fdinfo/3` |
| Inode cache size | /proc/slabinfo | `grep inode /proc/slabinfo` |
| Dentry cache size | /proc/slabinfo | `grep dentry /proc/slabinfo` |
| FS of a file | stat() → st_dev | `stat -f filename` |
| Deleted open files | /proc/*/fd | `lsof +L1` |
| File's block map | FIEMAP ioctl | `filefrag filename` |
| Directory entries | getdents() syscall | `ls` / readdir() |

## Key Linux Files for VFS Study

| File | What's inside |
|------|--------------|
| fs/namei.c | Path resolution (path_walk, lookup_slow) |
| fs/inode.c | Inode management (iget, iput, evict) |
| fs/dcache.c | Dentry cache (d_lookup, d_alloc, d_drop) |
| fs/open.c | open() syscall implementation |
| fs/read_write.c | read()/write() VFS layer |
| fs/ext4/namei.c | ext4's directory lookup (htree) |
| fs/ext4/inode.c | ext4's inode operations |
| include/linux/fs.h | All VFS struct definitions |

## VFS ↔ Storage Engine Concept Map

```
VFS Concept              Storage Engine Concept
─────────────            ─────────────────────
superblock            ↔  Database metadata (page 0)
inode                 ↔  Page/record metadata
inode number          ↔  PageId
dentry                ↔  Key → PageId mapping
dcache                ↔  In-memory index cache
icache                ↔  Buffer pool
address_space         ↔  Buffer pool (page cache per file)
readpage()            ↔  BufferPool::fetch_page()
writepage()           ↔  BufferPool::flush_page()
lookup()              ↔  BTree::search()
create()              ↔  BTree::insert()
unlink()              ↔  BTree::delete()
ext4 journal          ↔  Write-ahead log
extent tree           ↔  B+ tree
block group bitmap    ↔  Free page list
mount                 ↔  Database::open()
unmount               ↔  Database::close() + checkpoint
sync_fs               ↔  Full checkpoint
```

## Why This Matters for Your Storage Engine

1. **Your B+ tree IS a directory index.** ext4's HTree and your B+ tree
   solve the same problem: "given a name/key, find the data."

2. **Your buffer pool IS the kernel's page cache.** Both cache pages in RAM,
   track dirty pages, and use eviction policies.

3. **Your WAL IS ext4's journal.** Both write changes to a sequential log
   before modifying the main data structure, enabling crash recovery.

4. **Your PageManager IS the block allocator.** Both track which blocks/pages
   are free and allocate them on demand.

5. Understanding VFS teaches you that these patterns are UNIVERSAL — every
   system that manages persistent structured data reinvents the same
   architecture, whether it's a filesystem, a database, or a key-value store.
