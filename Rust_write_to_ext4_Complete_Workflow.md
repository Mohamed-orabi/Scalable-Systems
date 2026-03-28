# From Rust write() to ext4 on Disk

## Mimicking the Actual Linux Kernel Workflow in Rust Code and Analogy

---

# What This Guide Does

We're going to BUILD a miniature version of the entire Linux I/O stack
in Rust — from the write() call all the way down to "data on disk."
Every layer is implemented as actual runnable code, AND explained with
an analogy.

The layers we'll build:

```
Layer 1: User Program          → You calling write()
Layer 2: Syscall Interface     → Translating your call into kernel terms
Layer 3: VFS (sys_write)       → fd lookup, permission checks, dispatch
Layer 4: VFS (vfs_write)       → Building I/O descriptors, more checks
Layer 5: ext4 file_operations  → ext4_file_write_iter()
Layer 6: Page Cache            → Finding or creating the page in RAM
Layer 7: ext4 address_space    → write_begin / write_end
Layer 8: Data Copy             → Your bytes enter kernel memory
Layer 9: Dirty Marking         → "This page needs to be flushed"
Layer 10: Return Path          → Unwinding back to your code

(Later, on fsync:)
Layer 11: Writeback            → Page cache → bio
Layer 12: Block I/O            → Merging, scheduling
Layer 13: Device Driver        → Hardware command
Layer 14: Disk Hardware         → NAND flash / platters
```

Each layer is implemented as a Rust function that calls the next layer.
You can read the code top-to-bottom and trace exactly how data flows.


---

# The Analogy: A Hospital

We'll use a hospital analogy throughout. Each kernel layer maps to a
hospital department:

```
You (the patient)              = Your Rust program
Hospital reception             = Syscall entry
Triage nurse                   = VFS sys_write (checks your ID, insurance)
Head nurse                     = VFS vfs_write (5 safety checks)
Department (Cardiology)        = ext4 (the specific filesystem)
The patient's CHART            = struct file (tracks your session)
The medicine CABINET           = Page cache (stores drugs/data in RAM)
A SHELF in the cabinet         = struct page (one 4KB data chunk)
The PHARMACY warehouse         = Disk (permanent storage)
Writing on the chart           = Marking a page dirty
Restocking from warehouse      = Reading from disk on cache miss
Sending to warehouse           = fsync (flushing dirty pages)
```


---

# The Code: Complete Miniature Linux I/O Stack

```rust
// ══════════════════════════════════════════════════════════════
// COMPLETE MINIATURE LINUX I/O STACK
// ══════════════════════════════════════════════════════════════
//
// This is a simplified but structurally accurate model of how
// Linux handles write(fd, buf, count) on an ext4 filesystem.
//
// Every function mirrors an actual kernel function.
// Comments show the real kernel source file and function name.

use std::collections::HashMap;

// ─────────────────────────────────────────────────────────────
//  DATA STRUCTURES — mirrors of actual kernel structs
// ─────────────────────────────────────────────────────────────

const PAGE_SIZE: usize = 4096;

/// The inode — the TRUE identity of a file on disk.
/// One per file, regardless of how many processes open it.
///
/// Real kernel: struct inode (include/linux/fs.h)
/// Analogy: The patient's MEDICAL RECORD in the hospital archives.
///          There's only one per patient, no matter how many doctors see them.
struct Inode {
    ino: u64,                              // inode number (unique ID)
    size: u64,                             // file size in bytes
    mapping: AddressSpaceId,               // which page cache (always my own)
}

/// The address_space — the page cache for ONE file.
/// Maps page indices to cached pages in RAM.
///
/// Real kernel: struct address_space (include/linux/fs.h)
/// Analogy: A medicine CABINET labeled with the patient's name.
///          Each shelf (slot) holds one drug (4KB page).
///          The cabinet belongs to the patient (inode), not to any doctor.
struct AddressSpace {
    host_ino: u64,                         // which inode I belong to
    pages: HashMap<u64, PageCachePage>,     // THE MAP: page_index → page
    nrpages: u32,                          // how many pages cached
}

type AddressSpaceId = usize;               // index into our address_spaces vec

/// A page in the page cache — one 4KB chunk of file data in RAM.
///
/// Real kernel: struct page (include/linux/mm_types.h)
/// Analogy: One SHELF in the medicine cabinet.
///          The shelf holds one drug (4KB of data).
///          It might be "dirty" (modified but not sent to the warehouse yet).
struct PageCachePage {
    data: [u8; PAGE_SIZE],                 // THE ACTUAL DATA (4096 bytes)
    index: u64,                            // which page of the file (0, 1, 2...)
    dirty: bool,                           // modified in memory, not on disk yet?
    uptodate: bool,                        // does this page contain valid data?
}

/// The struct file — an open file session.
/// Created by open(), personal to the opener.
///
/// Real kernel: struct file (include/linux/fs.h)
/// Analogy: The patient's CHART that a doctor carries around.
///          It says: which patient (inode), what page we're on (f_pos),
///          and what we're allowed to do (read-only? read-write?).
struct OpenFile {
    f_pos: u64,                            // current read/write offset
    f_flags: u32,                          // how file was opened
    f_mode: u32,                           // allowed operations
    f_inode_idx: usize,                    // which inode (index into inodes vec)
    f_mapping_idx: AddressSpaceId,         // shortcut to inode's page cache
    f_op: FileOperationsType,              // which filesystem handles I/O
}

/// Flags
const O_RDONLY: u32 = 0;
const O_WRONLY: u32 = 1;
const O_RDWR: u32 = 2;
const FMODE_WRITE: u32 = 0x2;
const FMODE_READ: u32 = 0x1;

/// Which filesystem's operations to use.
/// Real kernel: struct file_operations (function pointer table)
/// Analogy: Which DEPARTMENT of the hospital handles this patient.
///          Cardiology has different procedures than Neurology.
#[derive(Debug, Clone, Copy)]
enum FileOperationsType {
    Ext4,
    // Xfs,     // would be another variant
    // Nfs,     // another
}

/// The file descriptor table — maps fd numbers to open files.
/// Real kernel: struct files_struct → struct fdtable (include/linux/fdtable.h)
/// Analogy: The reception desk's LOG BOOK.
///          "Bed 3 (fd 3) → that patient's chart (struct file)"
struct FdTable {
    entries: Vec<Option<usize>>,           // fd → index into open_files vec
}

/// The kiocb — I/O control block passed to the filesystem.
/// Real kernel: struct kiocb (include/linux/fs.h)
/// Analogy: A WORK ORDER form that the head nurse fills out and gives
///          to the department. "Patient X, write 5ml at position Y."
struct Kiocb {
    ki_pos: u64,                           // file offset
    ki_file_idx: usize,                    // which open file
}

/// The iov_iter — describes the user's data buffer.
/// Real kernel: struct iov_iter (include/linux/uio.h)
/// Analogy: The actual MEDICINE VIAL that the doctor brings.
///          "Here's the 5ml of data to inject at the specified location."
struct IovIter {
    data: Vec<u8>,                         // the user's data
    offset: usize,                         // how much has been consumed
}

/// The "kernel" — holds all global state.
/// In the real kernel, these are global variables and per-process structures.
/// Analogy: The entire HOSPITAL — all departments, all cabinets, all records.
struct Kernel {
    inodes: Vec<Inode>,
    address_spaces: Vec<AddressSpace>,
    open_files: Vec<OpenFile>,
    fd_table: FdTable,
    disk: DiskBackend,                     // simulates the physical disk
    indent: usize,                         // for pretty-printing call depth
}

/// Simulates the physical disk — just a big byte array.
struct DiskBackend {
    data: Vec<u8>,                         // the "disk" contents
}

impl DiskBackend {
    fn new(size: usize) -> Self {
        Self { data: vec![0u8; size] }
    }
    fn read_page(&self, block_number: u64) -> [u8; PAGE_SIZE] {
        let offset = block_number as usize * PAGE_SIZE;
        let mut page = [0u8; PAGE_SIZE];
        if offset + PAGE_SIZE <= self.data.len() {
            page.copy_from_slice(&self.data[offset..offset + PAGE_SIZE]);
        }
        page
    }
    fn write_page(&mut self, block_number: u64, data: &[u8; PAGE_SIZE]) {
        let offset = block_number as usize * PAGE_SIZE;
        if offset + PAGE_SIZE <= self.data.len() {
            self.data[offset..offset + PAGE_SIZE].copy_from_slice(data);
        }
    }
}

// ─────────────────────────────────────────────────────────────
//  HELPER: pretty-print with indentation to show call depth
// ─────────────────────────────────────────────────────────────

impl Kernel {
    fn log(&self, msg: &str) {
        let indent = "│ ".repeat(self.indent);
        println!("  {}  {}", indent, msg);
    }
    fn enter(&mut self, layer: &str, real_fn: &str, analogy: &str) {
        let indent = "│ ".repeat(self.indent);
        println!("  {}┌─ {} ─────────────────────────────", indent, layer);
        println!("  {}│  Kernel function: {}", indent, real_fn);
        println!("  {}│  Analogy: {}", indent, analogy);
        self.indent += 1;
    }
    fn exit(&mut self, layer: &str, result: &str) {
        self.indent -= 1;
        let indent = "│ ".repeat(self.indent);
        println!("  {}│  Returns: {}", indent, result);
        println!("  {}└─ {} done ──────────────────────────", indent, layer);
    }
}

// ═════════════════════════════════════════════════════════════
//  LAYER 1: YOUR PROGRAM
// ═════════════════════════════════════════════════════════════

/// Your Rust code calls file.write_all(b"Hello").
/// Under the hood, Rust's std calls libc::write(fd, buf, count).
///
/// Analogy: You (the patient) walk into the hospital and say
///          "I need to update my prescription."
fn user_program_write(kernel: &mut Kernel, fd: usize, data: &[u8]) -> Result<usize, String> {
    println!("\n  ╔═══════════════════════════════════════════════════════╗");
    println!("  ║  YOUR CODE: write(fd={}, {:?}, {})  ║",
        fd, String::from_utf8_lossy(data), data.len());
    println!("  ╚═══════════════════════════════════════════════════════╝\n");

    // In reality, Rust std calls libc::write which executes the SYSCALL instruction.
    // The CPU flips to ring 0 (kernel mode) and jumps to the syscall entry point.
    // We simulate that by directly calling the kernel function.

    syscall_entry_write(kernel, fd, data)
}

// ═════════════════════════════════════════════════════════════
//  LAYER 2: SYSCALL ENTRY
//  Real: arch/x86/entry/entry_64.S → entry_SYSCALL_64
// ═════════════════════════════════════════════════════════════

/// The CPU has flipped to ring 0. The kernel's assembly entry code:
/// 1. Switches to the kernel stack (swapgs + mov rsp)
/// 2. Saves all user registers onto the kernel stack
/// 3. Reads RAX (syscall number = 1 = SYS_write)
/// 4. Looks up sys_call_table[1] = sys_write
/// 5. Calls sys_write(fd, buf, count)
///
/// Analogy: Hospital RECEPTION.
///          "Please sign in." (save registers)
///          "Which department?" (look up syscall table)
///          "Ah, write — that goes to the file I/O desk." (dispatch)
fn syscall_entry_write(kernel: &mut Kernel, fd: usize, data: &[u8]) -> Result<usize, String> {
    kernel.enter(
        "LAYER 2: Syscall Entry",
        "entry_SYSCALL_64 → sys_call_table[1]",
        "Hospital reception — sign in, find the right desk"
    );

    kernel.log("CPU: CPL 3→0 (ring 3 → ring 0, privilege switch)");
    kernel.log("CPU: RSP → kernel stack (switch from user stack)");
    kernel.log("Saving all user registers onto kernel stack...");
    kernel.log("RAX=1 → sys_call_table[1] = sys_write");

    let result = sys_write(kernel, fd, data);

    kernel.log("Restoring all user registers from kernel stack...");
    kernel.log("CPU: CPL 0→3 (back to user mode via sysretq)");
    kernel.exit("LAYER 2", &format!("{:?}", result));

    result
}

// ═════════════════════════════════════════════════════════════
//  LAYER 3: VFS — sys_write()
//  Real: fs/read_write.c → SYSCALL_DEFINE3(write, ...)
// ═════════════════════════════════════════════════════════════

/// sys_write: converts fd (integer) to struct file, reads f_pos,
/// then calls vfs_write with the actual file object and position.
///
/// Analogy: TRIAGE NURSE.
///          "Show me your wristband." (fd → struct file)
///          "Let me check which bed you're in." (read f_pos)
///          "OK, sending you to the head nurse for safety checks."
fn sys_write(kernel: &mut Kernel, fd: usize, data: &[u8]) -> Result<usize, String> {
    kernel.enter(
        "LAYER 3: VFS sys_write()",
        "fs/read_write.c → SYSCALL_DEFINE3(write)",
        "Triage nurse — check wristband (fd), find bed (f_pos)"
    );

    // ── Step 1: fdget_pos(fd) — convert fd to struct file ──

    kernel.log(&format!("fdget_pos(fd={}):", fd));
    kernel.log("  current→files→fdt→fd[3] → struct file pointer");

    let file_idx = kernel.fd_table.entries
        .get(fd)
        .and_then(|opt| *opt)
        .ok_or_else(|| {
            kernel.log("  fd not found in table → EBADF!");
            "EBADF: Bad file descriptor".to_string()
        })?;

    let f_pos = kernel.open_files[file_idx].f_pos;
    kernel.log(&format!("  Found struct file: f_pos={}, f_mode={:#x}",
        f_pos, kernel.open_files[file_idx].f_mode));

    // ── Step 2: Call vfs_write with the position ──

    let result = vfs_write(kernel, file_idx, data, f_pos);

    // ── Step 3: Update f_pos if write succeeded ──

    if let Ok(bytes_written) = &result {
        let new_pos = f_pos + *bytes_written as u64;
        kernel.open_files[file_idx].f_pos = new_pos;
        kernel.log(&format!("f_pos updated: {} → {}", f_pos, new_pos));
    }

    kernel.exit("LAYER 3", &format!("{:?}", result));
    result
}

// ═════════════════════════════════════════════════════════════
//  LAYER 4: VFS — vfs_write()
//  Real: fs/read_write.c → vfs_write()
// ═════════════════════════════════════════════════════════════

/// vfs_write: performs 5 safety checks, then dispatches to the filesystem.
///
/// Analogy: HEAD NURSE performing 5 safety checks before any procedure:
///          1. "Can this patient receive medication?" (FMODE_WRITE)
///          2. "Does this department handle injections?" (f_op->write_iter)
///          3. "Is the prescription form valid?" (access_ok)
///          4. "Would this exceed the dosage limit?" (rw_verify_area)
///          5. "Is this patient cleared by security?" (security check)
fn vfs_write(kernel: &mut Kernel, file_idx: usize, data: &[u8], pos: u64)
    -> Result<usize, String>
{
    kernel.enter(
        "LAYER 4: VFS vfs_write()",
        "fs/read_write.c → vfs_write()",
        "Head nurse — 5 safety checks before any procedure"
    );

    let file = &kernel.open_files[file_idx];

    // ── CHECK 1: Does the file allow writing? ──
    kernel.log("CHECK 1: f_mode & FMODE_WRITE?");
    if file.f_mode & FMODE_WRITE == 0 {
        kernel.log("  FAIL: file opened read-only → EBADF");
        kernel.exit("LAYER 4", "Err(EBADF)");
        return Err("EBADF: Not open for writing".to_string());
    }
    kernel.log("  PASS ✓ (file is writable)");

    // ── CHECK 2: Does the filesystem implement write? ──
    kernel.log("CHECK 2: f_op->write_iter exists?");
    match file.f_op {
        FileOperationsType::Ext4 => kernel.log("  PASS ✓ (ext4 has write_iter)"),
    }

    // ── CHECK 3: Is the user buffer valid? ──
    kernel.log("CHECK 3: access_ok(buf, count)?");
    kernel.log(&format!("  buf={:#x}, count={}", 0x7FFE0040u64, data.len()));
    kernel.log("  Is buf < TASK_SIZE? Yes. No overflow? Yes.");
    kernel.log("  PASS ✓ (buffer is in valid user address range)");

    // ── CHECK 4: File size limits and locks ──
    kernel.log("CHECK 4: rw_verify_area(WRITE)?");
    kernel.log(&format!("  pos={}, count={} → end={}", pos, data.len(), pos + data.len() as u64));
    kernel.log("  No RLIMIT_FSIZE exceeded. No mandatory locks.");
    kernel.log("  PASS ✓");

    // ── CHECK 5: Security module ──
    kernel.log("CHECK 5: security_file_permission(MAY_WRITE)?");
    kernel.log("  SELinux/AppArmor check (if active).");
    kernel.log("  PASS ✓ (no LSM objection)");

    kernel.log("");
    kernel.log("ALL 5 CHECKS PASSED — dispatching to filesystem");

    // ── Build the kiocb and iov_iter ──

    kernel.log("Building kiocb (I/O control block):");
    kernel.log(&format!("  ki_pos = {}", pos));
    kernel.log(&format!("  ki_file = struct file #{}", file_idx));

    let kiocb = Kiocb {
        ki_pos: pos,
        ki_file_idx: file_idx,
    };

    kernel.log("Building iov_iter (data descriptor):");
    kernel.log(&format!("  data = {:?} ({} bytes)", String::from_utf8_lossy(data), data.len()));

    let iov_iter = IovIter {
        data: data.to_vec(),
        offset: 0,
    };

    // ── Dispatch: call_write_iter → f_op->write_iter ──

    kernel.log("");
    kernel.log("call_write_iter(file, &kiocb, &iov_iter)");
    kernel.log("  → file->f_op->write_iter(&kiocb, &iov_iter)");

    let f_op = kernel.open_files[file_idx].f_op;
    let result = match f_op {
        FileOperationsType::Ext4 => {
            kernel.log("  → ext4_file_write_iter(&kiocb, &iov_iter)");
            ext4_file_write_iter(kernel, kiocb, iov_iter)
        }
    };

    // ── Post-write: notifications and accounting ──

    if let Ok(bytes) = &result {
        kernel.log(&format!("fsnotify_modify() — notify watchers: {} bytes written", bytes));
        kernel.log(&format!("add_wchar(current, {}) — accounting", bytes));
    }

    kernel.exit("LAYER 4", &format!("{:?}", result));
    result
}

// ═════════════════════════════════════════════════════════════
//  LAYER 5: ext4 — ext4_file_write_iter()
//  Real: fs/ext4/file.c → ext4_file_write_iter()
// ═════════════════════════════════════════════════════════════

/// The ext4 filesystem's write entry point. Decides between direct I/O
/// and buffered I/O, then calls generic_perform_write for buffered writes.
///
/// Analogy: The CARDIOLOGY DEPARTMENT receives the work order.
///          "Is this an emergency direct procedure? Or standard treatment?"
///          → Standard: go through the medicine cabinet (page cache).
fn ext4_file_write_iter(kernel: &mut Kernel, kiocb: Kiocb, iov_iter: IovIter)
    -> Result<usize, String>
{
    kernel.enter(
        "LAYER 5: ext4_file_write_iter()",
        "fs/ext4/file.c → ext4_file_write_iter()",
        "Cardiology department — received the work order"
    );

    kernel.log("Check: is O_DIRECT set?");
    kernel.log("  → No. Using BUFFERED write path (through page cache).");
    kernel.log("  (Direct I/O would bypass the page cache entirely.)");
    kernel.log("");
    kernel.log("Calling generic_perform_write()...");

    let result = generic_perform_write(kernel, kiocb, iov_iter);

    kernel.exit("LAYER 5", &format!("{:?}", result));
    result
}

// ═════════════════════════════════════════════════════════════
//  LAYER 6: Generic Write → Page Cache Interaction
//  Real: mm/filemap.c → generic_perform_write()
// ═════════════════════════════════════════════════════════════

/// The generic buffered write loop. For each page that needs to be written:
///   1. Call write_begin (prepare the page)
///   2. Copy user data into the page
///   3. Call write_end (mark dirty, update metadata)
///
/// Analogy: The PHARMACIST's procedure:
///   1. Find the right shelf in the cabinet (write_begin)
///   2. Put the medicine on the shelf (data copy)
///   3. Mark the shelf as "needs restocking to warehouse" (write_end/dirty)
fn generic_perform_write(kernel: &mut Kernel, kiocb: Kiocb, mut iov_iter: IovIter)
    -> Result<usize, String>
{
    kernel.enter(
        "LAYER 6: generic_perform_write()",
        "mm/filemap.c → generic_perform_write()",
        "Pharmacist — find shelf, place medicine, mark for restocking"
    );

    let file_idx = kiocb.ki_file_idx;
    let mapping_idx = kernel.open_files[file_idx].f_mapping_idx;
    let total_bytes = iov_iter.data.len();
    let mut pos = kiocb.ki_pos;
    let mut bytes_written = 0;

    kernel.log(&format!("Total bytes to write: {}", total_bytes));
    kernel.log(&format!("Starting file offset: {}", pos));

    // Loop: one iteration per page that needs writing
    while bytes_written < total_bytes {
        let page_index = pos / PAGE_SIZE as u64;
        let offset_in_page = (pos % PAGE_SIZE as u64) as usize;
        let bytes_this_page = (PAGE_SIZE - offset_in_page).min(total_bytes - bytes_written);

        kernel.log("");
        kernel.log(&format!("── Page iteration: page_index={}, offset_in_page={}, bytes={}",
            page_index, offset_in_page, bytes_this_page));

        // ── Step 1: write_begin — prepare the page ──

        let page_key = ext4_write_begin(kernel, mapping_idx, page_index, offset_in_page, bytes_this_page)?;

        // ── Step 2: copy data from user buffer to page cache ──

        kernel.log("STEP 2: copy_from_user()");
        kernel.log(&format!("  Source: user buffer offset {}", bytes_written));
        kernel.log(&format!("  Dest: page cache page {}, byte offset {}", page_index, offset_in_page));

        {
            let page = kernel.address_spaces[mapping_idx].pages.get_mut(&page_key).unwrap();
            let src = &iov_iter.data[bytes_written..bytes_written + bytes_this_page];
            page.data[offset_in_page..offset_in_page + bytes_this_page].copy_from_slice(src);

            kernel.log(&format!("  Copied {} bytes: {:?}",
                bytes_this_page, String::from_utf8_lossy(src)));
            kernel.log("  ★ YOUR DATA IS NOW IN THE PAGE CACHE (kernel RAM) ★");
        }

        // ── Step 3: write_end — finalize ──

        ext4_write_end(kernel, mapping_idx, page_key, bytes_this_page)?;

        pos += bytes_this_page as u64;
        bytes_written += bytes_this_page;
    }

    kernel.exit("LAYER 6", &format!("Ok({} bytes)", bytes_written));
    Ok(bytes_written)
}

// ═════════════════════════════════════════════════════════════
//  LAYER 7a: ext4_write_begin()
//  Real: fs/ext4/inode.c → ext4_write_begin()
// ═════════════════════════════════════════════════════════════

/// Prepare a page for writing. Find it in the cache or create it.
/// If we're doing a partial page write, read the existing data from disk first.
///
/// Analogy: The pharmacist checks the medicine cabinet:
///          "Is the shelf for this drug already stocked?"
///          → Yes: use it. Lock the shelf so nobody else touches it.
///          → No: get an empty shelf. If we're only updating PART of the
///            shelf's contents, get the current stock from the warehouse first.
fn ext4_write_begin(kernel: &mut Kernel, mapping_idx: AddressSpaceId,
                    page_index: u64, offset_in_page: usize, write_len: usize)
    -> Result<u64, String>
{
    kernel.enter(
        "LAYER 7a: ext4_write_begin()",
        "fs/ext4/inode.c → ext4_write_begin()",
        "Pharmacist checks cabinet — shelf stocked or need warehouse trip?"
    );

    let mapping = &kernel.address_spaces[mapping_idx];

    // ── Check if this page is already in the page cache ──

    kernel.log(&format!("Looking up page {} in address_space.i_pages (XArray)...", page_index));

    if mapping.pages.contains_key(&page_index) {
        kernel.log("  → CACHE HIT! Page is already in RAM.");
        kernel.log("  → Locking the page (no one else modifies it during our write).");
        kernel.exit("LAYER 7a", &format!("page {} (cached)", page_index));
        return Ok(page_index);
    }

    // ── CACHE MISS: allocate a new page ──

    kernel.log("  → CACHE MISS. Page is NOT in RAM.");
    kernel.log("  → Allocating a new 4KB page from kernel memory...");

    let is_partial_write = offset_in_page != 0 || write_len != PAGE_SIZE;
    let mut new_page_data = [0u8; PAGE_SIZE];

    if is_partial_write {
        kernel.log("  → This is a PARTIAL page write (not overwriting all 4096 bytes).");
        kernel.log("  → Must read existing data from disk first to preserve other bytes.");
        kernel.log(&format!("  → ext4 extent tree: page {} → disk block {}", page_index, page_index));
        kernel.log("  → Reading from disk...");

        new_page_data = kernel.disk.read_page(page_index);

        kernel.log("  → Disk read complete. Page now has old data from disk.");
    } else {
        kernel.log("  → Full page write. No need to read from disk (we'll overwrite everything).");
    }

    // ── Insert into the page cache ──

    let new_page = PageCachePage {
        data: new_page_data,
        index: page_index,
        dirty: false,
        uptodate: true,
    };

    kernel.address_spaces[mapping_idx].pages.insert(page_index, new_page);
    kernel.address_spaces[mapping_idx].nrpages += 1;

    kernel.log(&format!("  → Page {} added to page cache. nrpages={}",
        page_index, kernel.address_spaces[mapping_idx].nrpages));
    kernel.log("  → Page is LOCKED (exclusive access for our write).");

    kernel.exit("LAYER 7a", &format!("page {} (new, from disk or zeroed)", page_index));
    Ok(page_index)
}

// ═════════════════════════════════════════════════════════════
//  LAYER 7b: ext4_write_end()
//  Real: fs/ext4/inode.c → ext4_write_end()
// ═════════════════════════════════════════════════════════════

/// Finalize after writing data into a page:
///   1. Mark the page DIRTY
///   2. Update the inode's file size if the file grew
///   3. Unlock the page
///
/// Analogy: After placing the medicine on the shelf:
///          1. Put a "NEEDS RESTOCKING TO WAREHOUSE" sticker (dirty flag)
///          2. Update the patient's chart if we added a new medicine
///          3. Remove the lock from the shelf (others can read it now)
fn ext4_write_end(kernel: &mut Kernel, mapping_idx: AddressSpaceId,
                  page_key: u64, bytes_copied: usize)
    -> Result<(), String>
{
    kernel.enter(
        "LAYER 7b: ext4_write_end()",
        "fs/ext4/inode.c → ext4_write_end()",
        "Pharmacist finishes — sticker the shelf, update chart, unlock"
    );

    // ── Step 1: Mark the page dirty ──

    kernel.log("STEP 1: set_page_dirty(page)");
    {
        let page = kernel.address_spaces[mapping_idx].pages.get_mut(&page_key).unwrap();
        page.dirty = true;
    }
    kernel.log("  → page.dirty = true");
    kernel.log("  → Page added to inode's dirty list");
    kernel.log("  → Inode added to filesystem's dirty inode list");
    kernel.log("  ★ THIS PAGE MUST BE WRITTEN TO DISK BEFORE EVICTION ★");

    // ── Step 2: Update inode size if file grew ──

    let inode_idx = 0; // simplified: we only have one inode
    let page = kernel.address_spaces[mapping_idx].pages.get(&page_key).unwrap();
    let write_end_offset = (page.index + 1) * PAGE_SIZE as u64;
    let old_size = kernel.inodes[inode_idx].size;
    if write_end_offset > old_size {
        kernel.inodes[inode_idx].size = write_end_offset;
        kernel.log(&format!("STEP 2: inode.i_size updated: {} → {} (file grew)",
            old_size, write_end_offset));
    } else {
        kernel.log(&format!("STEP 2: inode.i_size unchanged at {} (no growth)", old_size));
    }

    // ── Step 3: Unlock the page ──

    kernel.log("STEP 3: unlock_page(page)");
    kernel.log("  → Other threads can now access this page.");

    // ── NOTE about what has NOT happened ──

    kernel.log("");
    kernel.log("╔═══════════════════════════════════════════════════════╗");
    kernel.log("║ DATA IS IN PAGE CACHE (RAM) ONLY.                    ║");
    kernel.log("║ IT IS *** NOT *** ON DISK YET.                       ║");
    kernel.log("║                                                      ║");
    kernel.log("║ To get it on disk, you must call fsync().            ║");
    kernel.log("║ Until then:                                          ║");
    kernel.log("║   Power failure → DATA LOST                         ║");
    kernel.log("║   Process crash → data survives (kernel has it)      ║");
    kernel.log("╚═══════════════════════════════════════════════════════╝");

    kernel.exit("LAYER 7b", "Ok(())");
    Ok(())
}

// ═════════════════════════════════════════════════════════════
//  FSYNC PATH: ext4_sync_file → writeback → block I/O → disk
// ═════════════════════════════════════════════════════════════

/// Simulates fsync(): flushes all dirty pages for this file to disk.
///
/// Analogy: The warehouse RESTOCKING RUN.
///          Go through the cabinet, find every shelf with a "needs restocking"
///          sticker, copy the contents to the warehouse, remove the sticker.
fn ext4_sync_file(kernel: &mut Kernel, file_idx: usize) -> Result<(), String> {
    kernel.enter(
        "FSYNC: ext4_sync_file()",
        "fs/ext4/fsync.c → ext4_sync_file()",
        "Warehouse restocking run — flush all dirty shelves to warehouse"
    );

    let mapping_idx = kernel.open_files[file_idx].f_mapping_idx;

    // ── Step 1: filemap_write_and_wait — find and flush dirty pages ──

    kernel.log("Step 1: filemap_write_and_wait_range()");
    kernel.log("  Scanning address_space for dirty pages...");

    let dirty_indices: Vec<u64> = kernel.address_spaces[mapping_idx].pages.iter()
        .filter(|(_, page)| page.dirty)
        .map(|(idx, _)| *idx)
        .collect();

    if dirty_indices.is_empty() {
        kernel.log("  No dirty pages found. Nothing to flush.");
        kernel.exit("FSYNC", "Ok(())");
        return Ok(());
    }

    kernel.log(&format!("  Found {} dirty page(s): {:?}", dirty_indices.len(), dirty_indices));

    for &page_idx in &dirty_indices {
        kernel.log(&format!("  ── Flushing page {} ──", page_idx));

        // ── ext4_writepages: map file offset → disk block ──
        kernel.log(&format!("    ext4 extent tree: page {} → disk block {}", page_idx, page_idx));

        // ── Create bio (block I/O request) ──
        kernel.log(&format!("    Creating bio: WRITE, sector={}, 4096 bytes", page_idx * 8));

        // ── Block I/O layer: merge, schedule, dispatch ──
        kernel.log("    Block I/O: checking for merge opportunities...");
        kernel.log("    Block I/O: I/O scheduler ordering request...");
        kernel.log("    Block I/O: dispatching to device driver...");

        // ── Device driver: build hardware command ──
        kernel.log("    NVMe driver: building NVMe Write command");
        kernel.log(&format!("    NVMe: opcode=0x01, LBA={}, PRP=page_phys_addr", page_idx));
        kernel.log("    NVMe: ringing doorbell (MMIO write)...");

        // ── Actual disk write ──
        {
            let page_data = kernel.address_spaces[mapping_idx]
                .pages.get(&page_idx).unwrap().data;
            kernel.disk.write_page(page_idx, &page_data);
        }

        kernel.log("    SSD: DMA read data from RAM → NAND write cache");
        kernel.log("    SSD: completion interrupt received");

        // ── Mark page clean ──
        kernel.address_spaces[mapping_idx]
            .pages.get_mut(&page_idx).unwrap().dirty = false;
        kernel.log(&format!("    Page {} marked CLEAN (dirty=false)", page_idx));
    }

    // ── Step 2: Flush drive's volatile write cache ──

    kernel.log("");
    kernel.log("Step 2: blkdev_issue_flush()");
    kernel.log("  Sending FLUSH CACHE command to drive...");
    kernel.log("  NVMe: opcode=0x00 (Flush), NSID=1");
    kernel.log("  SSD: writing volatile DRAM cache → permanent NAND flash...");
    kernel.log("  SSD: flush complete. Interrupt received.");
    kernel.log("");
    kernel.log("  ★ DATA IS NOW ON PHYSICAL MEDIA. DURABLE. ★");
    kernel.log("  ★ Power failure from this point: data survives. ★");

    kernel.exit("FSYNC", "Ok(())");
    Ok(())
}

// ═════════════════════════════════════════════════════════════
//  SETUP AND MAIN
// ═════════════════════════════════════════════════════════════

fn setup_kernel() -> Kernel {
    // Create a simulated disk (100 pages = 400KB)
    let disk = DiskBackend::new(100 * PAGE_SIZE);

    // Create an inode for our database file (inode 4521)
    let inode = Inode {
        ino: 4521,
        size: 10 * PAGE_SIZE as u64,  // 40KB file, 10 pages
        mapping: 0,                    // address_space index 0
    };

    // Create the address_space (page cache) for this inode
    let address_space = AddressSpace {
        host_ino: 4521,
        pages: HashMap::new(),
        nrpages: 0,
    };

    // Create an open file (what you get from open())
    let open_file = OpenFile {
        f_pos: 8192,                       // cursor at byte 8192 (page 2)
        f_flags: O_RDWR,
        f_mode: FMODE_READ | FMODE_WRITE,
        f_inode_idx: 0,
        f_mapping_idx: 0,
        f_op: FileOperationsType::Ext4,
    };

    // Create the fd table: fd 3 → open file 0
    let fd_table = FdTable {
        entries: vec![
            Some(0),   // fd 0 → stdin (not really, but placeholder)
            Some(0),   // fd 1 → stdout
            Some(0),   // fd 2 → stderr
            Some(0),   // fd 3 → our database file ★
        ],
    };

    Kernel {
        inodes: vec![inode],
        address_spaces: vec![address_space],
        open_files: vec![open_file],
        fd_table,
        disk,
        indent: 0,
    }
}

fn main() {
    println!("╔═══════════════════════════════════════════════════════════════╗");
    println!("║  From Rust write() to ext4 on Disk                           ║");
    println!("║  Mimicking the Actual Linux Kernel Workflow                   ║");
    println!("╚═══════════════════════════════════════════════════════════════╝");
    println!();
    println!("  File: /data/database.db (inode 4521, 40KB, ext4)");
    println!("  fd=3, f_pos=8192 (byte 8192 = page 2, offset 0)");
    println!("  Operation: write(3, \"Hello\", 5)");

    let mut kernel = setup_kernel();

    // ─── THE WRITE ───

    let result = user_program_write(&mut kernel, 3, b"Hello");

    println!("\n  ╔═══════════════════════════════════════════════════════╗");
    println!("  ║  write() returned: {:?}", result);
    println!("  ║  Data is in page cache (RAM) — NOT on disk yet!      ║");
    println!("  ╚═══════════════════════════════════════════════════════╝");

    // ─── Verify: data is in page cache but NOT on disk ───

    println!("\n  Verification:");
    let mapping = &kernel.address_spaces[0];
    if let Some(page) = mapping.pages.get(&2) {
        let preview = String::from_utf8_lossy(&page.data[0..5]);
        println!("    Page cache page 2: {:?} dirty={}", preview, page.dirty);
    }
    let disk_page = kernel.disk.read_page(2);
    let disk_preview = String::from_utf8_lossy(&disk_page[0..5]);
    println!("    Disk page 2:       {:?} (still old data!)", disk_preview);

    // ─── THE FSYNC ───

    println!("\n  Now calling fsync(fd=3)...\n");

    let _ = ext4_sync_file(&mut kernel, 0);

    // ─── Verify: data is now on disk ───

    println!("\n  Verification after fsync:");
    let mapping = &kernel.address_spaces[0];
    if let Some(page) = mapping.pages.get(&2) {
        let preview = String::from_utf8_lossy(&page.data[0..5]);
        println!("    Page cache page 2: {:?} dirty={}", preview, page.dirty);
    }
    let disk_page = kernel.disk.read_page(2);
    let disk_preview = String::from_utf8_lossy(&disk_page[0..5]);
    println!("    Disk page 2:       {:?} (NOW has our data!)", disk_preview);

    println!("\n  ╔═══════════════════════════════════════════════════════╗");
    println!("  ║  Complete! Data went from your Rust variable         ║");
    println!("  ║  through 7+ kernel layers to physical disk.          ║");
    println!("  ╚═══════════════════════════════════════════════════════╝");
}
```

The code above is a complete, runnable Rust program. To run it:

```bash
cargo init write_to_ext4
# Copy the code block above into write_to_ext4/src/main.rs
cd write_to_ext4
cargo run
```


---

# Layer-by-Layer Summary With Analogies

```
LAYER    KERNEL FUNCTION              ANALOGY                         WHAT IT DOES
─────    ──────────────────           ──────────────────────          ─────────────────────────────────
  1      Your Rust code               You walk into the hospital      Call write(fd, buf, 5)
  
  2      entry_SYSCALL_64             Reception desk                  CPU flips to ring 0
                                      "Sign in, find the              Save registers, look up
                                       right desk"                    syscall table, dispatch

  3      sys_write()                  Triage nurse                    fd → struct file
                                      "Show me your                   Read f_pos (8192)
                                       wristband"                     Call vfs_write

  4      vfs_write()                  Head nurse                      5 safety checks:
                                      "5 safety checks                1. Can write? 2. FS supports it?
                                       before anything"               3. Buffer valid? 4. Limits?
                                                                      5. Security? Then dispatch.

         new_sync_write()             Fill out the work order         Build kiocb (where)
                                                                      Build iov_iter (what)
                                                                      Call f_op->write_iter

  5      ext4_file_write_iter()       Cardiology receives             Direct I/O or buffered?
                                      the work order                  → Buffered: use page cache
                                                                      Call generic_perform_write

  6      generic_perform_write()      Pharmacist's 3-step             For each page:
                                      procedure                       1. write_begin (get shelf)
                                                                      2. copy data (place medicine)
                                                                      3. write_end (sticker + unlock)

  7a     ext4_write_begin()           Check the cabinet               Page in cache? → use it
                                      "Is the shelf stocked?"         Not cached? → alloc new page
                                                                      Partial write? → read from disk
                                                                      Lock the page

  —      copy_from_user()             Place the medicine              memcpy: user buf → page cache
                                      on the shelf                    ★ Data enters kernel memory ★

  7b     ext4_write_end()             Sticker + unlock                Mark page DIRTY
                                      "Needs restocking"              Update inode size
                                                                      Unlock page
                                                                      ★ Data in RAM only, not disk ★

  —      (return path)                Walk back to reception          Unwind all layers
                                                                      Update f_pos (8192 → 8197)
                                                                      Return bytes written (5)

  ═══════════ write() returns here. Data is in RAM but NOT on disk. ═══════════

  8      ext4_sync_file()             Warehouse restocking            Find all dirty pages
         (called by fsync)            run                             For each:

  9      ext4_writepages()            Check each shelf for            Map page index → disk block
                                      "restocking" sticker            via ext4 extent tree

  10     submit_bio()                 Load the delivery truck         Create bio (I/O request)

  11     Block I/O layer              Route optimization              Merge adjacent requests
                                      "Which streets to take?"        I/O scheduler reorders

  12     NVMe driver                  Warehouse forklift              Build NVMe command
                                      operator                        DMA map, ring doorbell

  13     SSD hardware                 Warehouse shelves               DMA read from RAM
                                                                      Write to NAND flash
                                                                      Completion interrupt

  14     FLUSH CACHE                  "Confirm everything             SSD flushes volatile cache
                                       is on the shelves"             to permanent NAND

  ═══════════ fsync() returns here. Data is on physical media. Durable. ═══════════
```
