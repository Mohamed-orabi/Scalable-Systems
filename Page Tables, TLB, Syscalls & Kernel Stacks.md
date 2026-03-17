# Page Tables, TLB, Syscalls & Kernel Stacks

## The Complete Guide — From Virtual Addresses to Physical RAM

---

# Part A: Page Tables

## Chapter 1: The Problem — Why Virtual Addresses Exist

Your program uses **virtual addresses** — when you write `let x = 42;`, the variable `x` lives at an address like `0x7FFE0040`. But that's NOT a real location in physical RAM. Your RAM chip has no idea what `0x7FFE0040` means.

Something needs to translate:

```
Virtual address (what your program sees)
         ↓
    [ ??? magic ??? ]
         ↓
Physical address (actual location in RAM chip)
```

That "magic" is the **page table** (the translation dictionary) and the **TLB** (a cache that makes it fast).

Without virtual memory, every program would need to know exactly which physical RAM addresses are free. Two programs couldn't both use address `0x1000` — they'd overwrite each other.

```
WITHOUT virtual memory (the old days):
  Program A uses physical addresses 0x0000 - 0x3FFF
  Program B uses physical addresses 0x4000 - 0x7FFF
  
  Problem: What if A needs more memory? B is in the way!
  Problem: What if A has a bug and writes to 0x5000? B is corrupted!
  Problem: A must know at compile time where it'll be loaded.

WITH virtual memory:
  Program A thinks it starts at 0x0000 (virtual)  → actually at physical 0x50000
  Program B thinks it starts at 0x0000 (virtual)  → actually at physical 0x80000
  
  Both programs use the same virtual addresses.
  The CPU translates them to different physical locations.
  A can't even SEE B's memory — it doesn't exist in A's virtual world.
```

**Analogy**: Virtual addresses are like apartment numbers. Every building has apartment 101. But "apartment 101 in Building A" and "apartment 101 in Building B" are completely different physical locations. The building directory (page table) translates "apartment 101" to an actual GPS coordinate.


## Chapter 2: Pages — The Unit of Translation

The CPU doesn't translate every single byte address individually — that would require a translation entry for every byte (billions of entries for 4GB of RAM). Instead, memory is divided into fixed-size chunks called **pages** (typically 4096 bytes = 4KB).

```
Virtual memory is divided into pages:

Virtual Page 0: bytes 0x0000 - 0x0FFF    (4096 bytes)
Virtual Page 1: bytes 0x1000 - 0x1FFF    (4096 bytes)
Virtual Page 2: bytes 0x2000 - 0x2FFF    (4096 bytes)
...

Physical memory is divided into frames (same size):

Physical Frame 0: bytes 0x0000 - 0x0FFF
Physical Frame 1: bytes 0x1000 - 0x1FFF
Physical Frame 2: bytes 0x2000 - 0x2FFF
...
```

The page table maps **virtual pages → physical frames**:

```
Page Table:
  Virtual Page 0 → Physical Frame 5
  Virtual Page 1 → Physical Frame 200
  Virtual Page 2 → Physical Frame 12
  Virtual Page 3 → NOT IN MEMORY (on disk!)
```

**Analogy**: Instead of translating every single street address, the post office translates ZIP codes. "ZIP 12345" → "the Cairo sorting facility." Then the last few digits find the exact house within that zone. Pages are ZIP codes for memory.


## Chapter 3: How a Virtual Address Is Split

When the CPU sees virtual address `0x00403042`, it splits it into two parts:

```
Virtual address: 0x00403042

In binary: 0000 0000 0100 0000 0011 | 0000 0100 0010
           ├──── page number ──────┤ ├── offset ───┤
                    0x403                 0x042

Page number = 0x403 = 1027    → "Which page?"
Offset      = 0x042 = 66     → "Where inside that page?"
```

The page number is looked up in the page table. The offset stays the same (it's the position WITHIN the page — that doesn't change).

```
Step 1: Page number 0x403 → look up in page table → physical frame 0x8F001
Step 2: Physical address = frame_base + offset = 0x8F001000 + 0x042 = 0x8F001042

Virtual  0x00403042  →  Physical 0x8F001042
```

**The offset (last 12 bits for 4KB pages) passes through unchanged.** Only the page number gets translated.


## Chapter 4: The Page Table Entry (PTE)

Each entry in the page table is called a **Page Table Entry (PTE)** and contains:

```
┌─────────────────────────────────────────────────┐
│  Page Table Entry (PTE) — 4 or 8 bytes          │
│                                                  │
│  Physical Frame Number (20+ bits)                │
│  ─── "Where is this page in physical RAM?" ───   │
│                                                  │
│  Present bit (1 bit)                             │
│  ─── "Is this page actually in RAM right now?"   │
│  ─── If 0 → page fault (load from disk)          │
│                                                  │
│  Read/Write bit (1 bit)                          │
│  ─── "Can this page be written to?"              │
│  ─── Code pages (.text) are read-only             │
│                                                  │
│  User/Supervisor bit (1 bit)                     │
│  ─── "Can user-mode code access this?"           │
│  ─── This is what blocks you from kernel space!  │
│                                                  │
│  Dirty bit (1 bit)                               │
│  ─── "Has this page been written to?"            │
│  ─── OS uses this for swapping decisions          │
│                                                  │
│  Accessed bit (1 bit)                            │
│  ─── "Has this page been read or written?"       │
│  ─── OS uses this for LRU-style eviction          │
└─────────────────────────────────────────────────┘
```

The **User/Supervisor bit** is the mechanism that protects kernel space. Every page in the kernel range (`0xC0000000+`) has `User/Supervisor = 0`, which means user-mode code (ring 3) gets an immediate fault when trying to access those addresses.


## Chapter 5: The Problem With a Flat Table

A 32-bit address space has 2³² bytes / 4096 bytes per page = 1,048,576 pages. Each PTE is 4 bytes, so the page table would be **4MB** — for EVERY process. If you have 100 processes, that's 400MB just for page tables. And most of that space is empty (most programs don't use the full 4GB).

A 64-bit address space would need 2⁵² / 4096 = 2⁴⁰ entries = **terabytes** of page table. Impossible.

**Solution: Multi-level page tables.**


## Chapter 6: Multi-Level Page Tables

Instead of one giant flat table, break it into a tree of smaller tables. Only allocate the sub-tables you actually need.

### Two-Level Page Table (x86 32-bit)

The virtual address is split into THREE parts:

```
Virtual address: 0x00403042

Binary: 0000000001 | 0000000011 | 000001000010
        ├─ Dir ──┤   ├─ Table ┤   ├─ Offset ─┤
        10 bits       10 bits       12 bits
        index=1       index=3       offset=66
```

```
                 ┌──────────────────┐
   CR3 register  │ Page Directory   │  ← 1024 entries, one per process
   (points here) │  (Level 1)       │
                 │                  │
                 │ Entry 0 → (null) │  ← not allocated (saves memory!)
                 │ Entry 1 → ─────────┐
                 │ Entry 2 → (null) │  │
                 │ ...              │  │
                 └──────────────────┘  │
                                       │
                 ┌─────────────────────┘
                 │
                 ▼
                 ┌──────────────────┐
                 │ Page Table       │  ← 1024 entries
                 │  (Level 2)       │
                 │                  │
                 │ Entry 0 → Frame X│
                 │ Entry 1 → Frame Y│
                 │ Entry 2 → Frame Z│
                 │ Entry 3 → Frame 12 │ ← THIS is our target
                 │ ...              │
                 └──────────────────┘
                           │
                           ▼
                 Physical Frame 12
                 byte offset 66 (0x042)
                           │
                           ▼
                 Physical address: Frame 12 base + 66
```

**The walk:**
1. CPU reads the Page Directory (level 1) entry at index 1
2. That points to a Page Table (level 2)
3. CPU reads the Page Table entry at index 3
4. That gives us physical frame 12
5. Add offset 66 → final physical address

**Why this saves memory:** If your program only uses a small portion of the address space, most level-1 entries point to NULL (no level-2 table allocated). A program using 10MB of memory might need only 3 level-2 tables (12KB) instead of a full 4MB flat table.

**Analogy**: Instead of a phone book with a page for every possible phone number (10 billion entries), you have a country code directory (200 entries) → area code directory (1000 entries per country) → local numbers. Most countries have only a few area codes allocated.

### Four-Level Page Table (x86-64)

64-bit systems use 4 levels (only 48 bits of the address are actually used):

```
Virtual address (48 bits used):

 PML4 index | PDP index | PD index | PT index | Offset
  9 bits      9 bits      9 bits     9 bits     12 bits
  (512 entries per table at each level)

Level 4: PML4 (Page Map Level 4)     ← 1 per process, CR3 points here
  ↓
Level 3: PDPT (Page Directory Pointer Table)
  ↓
Level 2: PD (Page Directory)
  ↓
Level 1: PT (Page Table)              ← contains the physical frame numbers
  ↓
Physical Frame + Offset = Physical Address
```

4 levels × 1 memory read per level = **4 memory reads to translate one address.** That's terrifyingly slow if done for every single memory access.

This is where the TLB saves us.


---

# Part B: The TLB (Translation Lookaside Buffer)

## Chapter 7: What the TLB Is

The TLB is a **cache inside the CPU** that stores recent page table translations. Instead of walking 4 levels of page tables for every memory access, the CPU checks the TLB first.

```
CPU wants to access virtual address 0x00403042:

Step 1: Split into page number (0x403) + offset (0x042)

Step 2: Check TLB:
  ┌──────────────────────────────────────┐
  │ TLB (inside the CPU, very fast)      │
  │                                      │
  │ Virtual Page  →  Physical Frame      │
  │ ──────────       ──────────────      │
  │ 0x200        →   Frame 88           │
  │ 0x403        →   Frame 12   ★ HIT!  │
  │ 0x7FFE0      →   Frame 301          │
  │ 0x001        →   Frame 5            │
  │ ...          →   ...                │
  └──────────────────────────────────────┘

Step 3: TLB hit! Physical Frame = 12.
        Physical address = 12 × 4096 + 66 = 49218
        Done! No page table walk needed.
```

### TLB Miss

```
CPU wants virtual address 0x00999000 (page 0x999):

Step 1: Check TLB → NOT FOUND (miss)

Step 2: Walk the page table (4 levels on x86-64):
        Level 4 → Level 3 → Level 2 → Level 1 → Frame 77

Step 3: Store the result in TLB:
        TLB: 0x999 → Frame 77   (for next time)

Step 4: Access physical address.
```

### TLB Characteristics

```
Size:        Tiny! Only 64-1536 entries (varies by CPU)
Speed:       1 clock cycle to check (effectively free)
Hit rate:    Usually 99%+ (programs access nearby memory repeatedly)
Miss penalty: ~10-100 clock cycles (page table walk)
```

**Why so small but so effective?** Because of **locality** — programs tend to access the same pages repeatedly. Your loop variable, your stack, your current function's code — all in a handful of pages that stay in the TLB.

**Analogy**: The TLB is like a sticky note on your monitor with the 5 phone numbers you call most often. You don't need the whole phone book — just the ones you use constantly. Looking at the sticky note is instant; opening the phone book takes time.


## Chapter 8: TLB Flush — Why It's Expensive

When the OS switches from Process A to Process B (**context switch**), Process B has a completely different page table. Process A's virtual page 5 maps to physical frame 100, but Process B's virtual page 5 maps to physical frame 200.

The old TLB entries are **wrong** for the new process. The OS must **flush** (clear) the TLB:

```
Before context switch:
  TLB: page 5 → frame 100   (Process A's mapping)
  TLB: page 8 → frame 300   (Process A's mapping)

Context switch to Process B:
  Step 1: Load Process B's page table (set CR3 register)
  Step 2: TLB is automatically flushed (all entries invalidated)

After context switch:
  TLB: (empty — everything is a miss now!)

Process B accesses page 5:
  TLB miss → walk B's page table → frame 200 → store in TLB
  TLB miss → walk B's page table → ...
  TLB miss → walk B's page table → ...
  (Every access is a miss until the TLB warms up again!)
```

**A TLB flush costs thousands of cycles** because every subsequent memory access becomes a TLB miss until the cache refills. This is why context switches are expensive, and why the kernel is mapped into every process's address space — to AVOID flushing the TLB on syscalls.

```
Syscall WITHOUT kernel in every process:
  1. Switch to kernel's page table → TLB FLUSH (everything gone!)
  2. Kernel runs (all TLB misses — slow!)
  3. Switch back to user's page table → TLB FLUSH again!
  Cost: ~thousands of extra cycles

Syscall WITH kernel in every process:
  1. Flip CPU privilege to ring 0 (no page table change)
  2. Kernel runs (kernel pages may TLB miss, but user TLB entries survive)
  3. Flip CPU privilege back to ring 3
  Cost: ~tens of cycles
```


## Chapter 9: PCID/ASID — Avoiding Unnecessary Flushes

Modern CPUs have a clever optimization: **PCID (Process Context ID)** on x86, or **ASID (Address Space ID)** on ARM.

Each TLB entry is tagged with a process ID:

```
TLB with PCID:
  (PCID=1, page 5) → frame 100    ← Process A's mapping
  (PCID=2, page 5) → frame 200    ← Process B's mapping
  (PCID=1, page 8) → frame 300    ← Process A's mapping
```

Now context switches DON'T flush the TLB! When Process B runs, the CPU only matches entries with PCID=2. When Process A runs again, its TLB entries are still there.

**This turned context switches from "destroy the TLB" to "almost free."** Modern Linux uses PCID on all CPUs that support it.


## Chapter 10: Page Faults — When Translation Fails

When the CPU walks the page table and finds `Present=0`, it triggers a **page fault** — a hardware interrupt that transfers control to the kernel. The kernel then decides what to do:

```
Page Fault Types:

1. MINOR FAULT (soft fault):
   The page exists but isn't in the page table yet.
   Example: First access to a newly mmap'd region.
   Kernel: allocate a physical frame, update the PTE, resume.
   Cost: ~1-10 microseconds

2. MAJOR FAULT (hard fault):
   The page was swapped out to disk.
   Kernel: read the page from swap space into a physical frame,
           update the PTE, resume.
   Cost: ~1-10 MILLISECONDS (1000x slower — disk I/O!)

3. SEGMENTATION FAULT:
   The page doesn't exist at all (invalid access).
   Example: null pointer dereference, accessing kernel space from user mode.
   Kernel: send SIGSEGV signal → process dies.
```

**This is demand paging**: the OS doesn't load all your program's memory upfront. It marks pages as "not present" and only loads them when you first access them (causing a minor fault). This is why `mmap()` returns instantly — no data is loaded until you touch it.


## Chapter 11: The Complete Translation Flow

Here's what happens for EVERY memory access your program makes:

```
Your code: let x = data[42];    // access virtual address 0x7F403042

  ┌───────────────────────────────────────────────────────┐
  │ CPU splits address: page=0x7F403 | offset=0x042       │
  │                                                        │
  │ ① Check TLB                                           │
  │    ├─ HIT (99% of the time) → physical address → done │
  │    └─ MISS → continue to ②                            │
  │                                                        │
  │ ② Walk the page table (hardware page table walker)     │
  │    CR3 → PML4[index] → PDPT[index] → PD[index]        │
  │    → PT[index] → PTE                                   │
  │    ├─ Present=1 → load TLB entry → physical addr → done│
  │    └─ Present=0 → continue to ③                        │
  │                                                        │
  │ ③ PAGE FAULT (hardware trap to kernel)                 │
  │    ├─ Valid mapping? → load page from disk/allocate     │
  │    │                  → update PTE → retry access       │
  │    └─ Invalid? → SEGFAULT → process dies                │
  └───────────────────────────────────────────────────────┘
```

- 99% of the time: TLB hit, 1 cycle, instant.
- 0.99% of the time: TLB miss, page table walk, ~10-100 cycles.
- 0.01% of the time: page fault, kernel intervention, ~1000+ cycles.


---

# Part C: Syscalls — How User Code Enters the Kernel

## Chapter 12: The Privilege Bit (CPU Ring Level)

The CPU has a register that stores the **Current Privilege Level (CPL)** — a 2-bit number:

```
Ring 0: CPL = 00  → Kernel mode (GOD mode — can do anything)
Ring 1: CPL = 01  → (unused on modern OSes)
Ring 2: CPL = 10  → (unused on modern OSes)
Ring 3: CPL = 11  → User mode (restricted — your program runs here)
```

**Every single CPU instruction checks this ring level.** Some instructions are only allowed at ring 0:

```
Ring 3 (your program) CAN do:
  ✓ add, subtract, multiply
  ✓ read/write memory (only pages marked User=1)
  ✓ call functions, loop, branch
  ✓ execute the special SYSCALL instruction

Ring 3 CANNOT do:
  ✗ read/write I/O ports (talk to hardware)
  ✗ modify page tables (CR3 register)
  ✗ disable interrupts (CLI instruction)
  ✗ access pages marked User=0 (kernel pages)
  ✗ halt the CPU (HLT instruction)

  If you try → CPU instantly triggers a "General Protection Fault"
  → kernel handles it → your process gets SIGSEGV → dies
```

**Analogy**: Ring level is like a security badge. Green badge (ring 3) gets you into regular offices. Red badge (ring 0) gets you into the server room, the vault, and the control panel. The door scanner (CPU) checks your badge on every entry.


## Chapter 13: The Five Steps of a Syscall

Let's trace `read(fd, buf, 4096)` in full detail.

### Step 1: Your Program Calls read()

```c
// Your C code (or what Rust's std generates):
ssize_t n = read(3, buffer, 4096);
```

This calls the libc wrapper, which does:

```asm
; x86-64 Linux syscall convention:
mov  rax, 0          ; syscall number 0 = SYS_read
mov  rdi, 3          ; arg1: fd = 3
mov  rsi, buffer     ; arg2: pointer to YOUR buffer (user address!)
mov  rdx, 4096       ; arg3: count = 4096
syscall              ; ★ THE magic instruction
```

The `syscall` instruction does several things **in a single CPU cycle**:

```
What the SYSCALL instruction does (hardware, not software):

1. Save the current instruction pointer (RIP) → RCX register
   "Remember where to come back to"

2. Save the current flags → R11 register

3. Load RIP from the MSR_LSTAR register
   "Jump to the kernel's syscall entry point"
   (This address was set up by the kernel at boot time)

4. Change CPL from 3 (user) to 0 (kernel)     ★ THE PRIVILEGE FLIP
   "You now have the red badge"

5. Change the stack pointer to the KERNEL STACK  ★ WHY EACH PROCESS
   "Switch from user stack to kernel stack"        HAS A KERNEL STACK

That's it. ONE instruction. No page table switch. No TLB flush.
The CPU just changes: privilege level + instruction pointer + stack pointer.
```

### Step 2: CPU Switches to Kernel Mode

After `syscall`, the CPU is now running at ring 0 (kernel mode). What changed and what didn't:

```
CHANGED:
  ✓ CPL: 3 → 0 (kernel mode)
  ✓ RIP: points to kernel's syscall entry function
  ✓ RSP: points to this process's KERNEL STACK
  ✓ The CPU can now access pages marked User=0

NOT CHANGED:
  ✗ CR3 register (page table base) — SAME page table!
  ✗ TLB — all cached translations still valid!
  ✗ All other registers — still contain your program's values
  ✗ The virtual address space — same 4GB (or 256TB) view
```

**This is the key**: because the kernel is MAPPED into every process's address space (at `0xC0000000+` on 32-bit, or `0xFFFF800000000000+` on 64-bit), the kernel's code and data are ALREADY visible. The CPU just couldn't access those pages before because CPL was 3. Now CPL is 0, so those pages are accessible.

```
Before syscall (CPL=3):
  0x00400000: your code        → ACCESSIBLE ✓
  0x7FFE0000: your stack       → ACCESSIBLE ✓
  0xC0000000: kernel code      → BLOCKED ✗ (User/Supervisor=0)

After syscall (CPL=0):
  0x00400000: your code        → ACCESSIBLE ✓ (still visible!)
  0x7FFE0000: your stack       → ACCESSIBLE ✓ (still visible!)
  0xC0000000: kernel code      → ACCESSIBLE ✓ (now allowed!)
```

**Nothing moved. Nothing was loaded. Nothing was flushed.** The CPU just changed which pages are accessible by flipping the privilege level.


### Step 3: Kernel Code Is Already Mapped

The kernel's syscall handler starts running:

```c
// Kernel's syscall entry point (simplified)
// This code lives at a virtual address in kernel space

void syscall_entry(void) {
    // We're on this process's KERNEL STACK now
    
    // Save ALL user registers onto the kernel stack
    struct pt_regs regs;
    regs.rax = user_rax;  // syscall number
    regs.rdi = user_rdi;  // arg1 (fd)
    regs.rsi = user_rsi;  // arg2 (buffer pointer — a USER address!)
    regs.rdx = user_rdx;  // arg3 (count)
    regs.rip = user_rcx;  // return address
    
    // Dispatch to the right syscall handler
    switch (regs.rax) {
        case 0: sys_read(regs.rdi, regs.rsi, regs.rdx); break;
        case 1: sys_write(regs.rdi, regs.rsi, regs.rdx); break;
        // ...
    }
}
```

**Why this works without loading anything**: The kernel's `.text` (code), `.data` (variables), and all kernel data structures are permanently mapped at high virtual addresses. The page table ALREADY has entries for them. The TLB might need to load a few kernel page translations on the first access, but that's just normal TLB misses — not a flush.


### Step 4: Kernel Reads YOUR Memory Directly

The kernel's `sys_read()` needs to read file data and copy it into YOUR buffer (at a user-space address):

```c
// Inside the kernel's sys_read() (simplified):
ssize_t sys_read(int fd, char __user *user_buffer, size_t count) {
    // 1. Find the file from the fd
    struct file *f = current->files->fd_table[fd];
    
    // 2. Read data from disk/page cache into a KERNEL buffer
    char *kernel_buf = kmalloc(count);
    f->f_op->read(f, kernel_buf, count, &f->f_pos);
    
    // 3. Copy from kernel buffer to USER buffer
    //    ★ THIS IS THE MAGIC — we can access user_buffer directly!
    copy_to_user(user_buffer, kernel_buf, count);
    
    // 4. Free kernel buffer, return bytes read
    kfree(kernel_buf);
    return count;
}
```

`copy_to_user()` is where the magic happens:

```c
// Simplified copy_to_user:
long copy_to_user(void __user *to, const void *from, long count) {
    // 'to' is something like 0x7FFE0040 — a USER-SPACE address
    // 'from' is something like 0xFFFF888001234000 — a KERNEL address
    
    // Because we're in ring 0, and the page table still has
    // BOTH user pages AND kernel pages mapped, we can just...
    
    memcpy(to, from, count);  // ...copy directly!
    
    // The user page at 0x7FFE0040 is in the SAME page table
    // we're currently using. We can read and write it.
}
```

**Why this works**: The page table hasn't changed! Your user-space buffer at `0x7FFE0040` is still mapped to the same physical frame. The kernel is running with the same page table your process was using. The only difference is CPL=0, which means the kernel can ALSO access kernel pages. But user pages are still there and accessible.

```
Page table during syscall (SAME as before syscall):

Virtual Address      Physical Frame    User/Supervisor
───────────────      ──────────────    ───────────────
0x00400000 (code)    Frame 50          User=1 (both can access)
0x7FFE0040 (buffer)  Frame 200         User=1 (both can access) ★
0xC0100000 (kernel)  Frame 5           User=0 (only ring 0)
```

The kernel writes to `0x7FFE0040` (your buffer), which goes through the same page table and lands in physical Frame 200 — exactly where your program expects the data to appear.


### Step 5: Return to Your Process

When the syscall is done:

```asm
; Kernel restores your registers from the kernel stack
; Then executes:
sysretq

; What SYSRETQ does (one instruction):
; 1. Change CPL from 0 → 3 (back to user mode)
; 2. Load RIP from RCX (return to where you were)
; 3. Load flags from R11
; 4. Switch RSP back to user stack
;
; Again: NO page table switch. NO TLB flush.
; Just flip the privilege bit and jump back.
```

Your program continues as if nothing happened. The buffer now contains the file data the kernel copied there.


---

# Part D: The Kernel Stack — Why Every Process Has One

## Chapter 14: The Problem Without Per-Process Kernel Stacks

Imagine all processes sharing ONE kernel stack:

```
WRONG: All processes share ONE kernel stack:

  Process A makes a syscall (read)
    → Enters kernel, pushes registers onto THE kernel stack
    → Kernel starts reading from disk... disk is slow...
    → Kernel decides: "I'll context-switch to Process B while waiting"
    
  Process B runs, makes a syscall (write)
    → Enters kernel, pushes registers onto THE SAME kernel stack
    → ★ DISASTER: B's registers OVERWRITE A's saved registers!
    → When A resumes, its saved state is GONE. Crash.
```


## Chapter 15: The Solution — Per-Process Kernel Stacks

```
RIGHT: Each process has its OWN kernel stack:

  Process A makes a syscall (read)
    → Enters kernel, pushes registers onto A's kernel stack
    → Kernel starts reading from disk... slow...
    → Context switch to Process B
       (A's kernel stack is safely preserved — nobody touches it)
    
  Process B makes a syscall (write)
    → Enters kernel, pushes registers onto B's kernel stack
    → No conflict! Each process's state is on its own stack.
    
  Context switch back to Process A
    → A's kernel stack still has all its saved registers
    → A continues exactly where it left off. Safe.
```


## Chapter 16: What's ON the Kernel Stack?

```
Process A's Kernel Stack (8KB or 16KB):

Top of stack (high address)
┌────────────────────────────────────┐
│ pt_regs (saved user registers)      │ ← saved when entering kernel
│   rax, rbx, rcx, rdx, rsi, rdi     │
│   rip (return address to user code) │
│   rsp (user stack pointer)          │
│   rflags                            │
├────────────────────────────────────┤
│ Kernel function call frames         │ ← normal function calls WITHIN kernel
│   sys_read() local variables        │
│   vfs_read() local variables        │
│   ext4_file_read() local variables  │
│   ...                               │
├────────────────────────────────────┤
│ (unused space)                      │
├────────────────────────────────────┤
│ thread_info (at the very bottom)    │ ← kernel metadata about this thread
└────────────────────────────────────┘
Bottom of stack (low address)
```

**The kernel stack is small** (8-16KB) because the kernel is careful about stack usage. Kernel code avoids large local variables and never does deep recursion. This is why kernel developers never write `char buffer[1000000]` — it would overflow the kernel stack and crash the entire system.


## Chapter 17: Where Is the Kernel Stack Stored?

```
Process A's virtual address space:

0xFFFFFFFF ┌──────────────────────────────────┐
           │ Kernel code (.text)              │ SHARED — same for all processes
           │ Kernel global data               │ SHARED
           │ ...                              │
           │ Per-CPU data                     │
           │ A's kernel stack (8KB)           │ ★ UNIQUE to process A
           │ A's task_struct                  │ ★ UNIQUE to process A
           │ ...                              │
0xC0000000 ├──────────────────────────────────┤
           │ A's user space                   │ Unique to A
0x00000000 └──────────────────────────────────┘

Process B's virtual address space:

0xFFFFFFFF ┌──────────────────────────────────┐
           │ Kernel code (.text)              │ SAME physical memory as A's
           │ Kernel global data               │ SAME
           │ ...                              │
           │ Per-CPU data                     │
           │ B's kernel stack (8KB)           │ ★ DIFFERENT physical memory!
           │ B's task_struct                  │ ★ DIFFERENT physical memory!
           │ ...                              │
0xC0000000 ├──────────────────────────────────┤
           │ B's user space                   │ Unique to B
0x00000000 └──────────────────────────────────┘
```

The kernel code and global data map to the **same physical frames** in every process. But each process's kernel stack and `task_struct` (the kernel's "process control block") map to **different physical frames**. So within the "shared" kernel space, there are some per-process regions.


---

# Part E: The Complete Syscall Timeline

## Chapter 18: Putting It All Together

```
Time (nanoseconds)   What happens
─────────────────    ──────────────────────────────────

     0 ns           Your code: read(3, buf, 4096)
                    CPU: ring 3, using your user stack

     1 ns           libc: loads syscall number into RAX
                    libc: executes SYSCALL instruction

     2 ns           CPU hardware (one cycle):
                      - CPL: 3 → 0
                      - RIP → kernel entry point
                      - RSP → this process's kernel stack
                      - Save old RIP in RCX, flags in R11

     3-10 ns        Kernel: save all user registers to kernel stack
                    Kernel: look up syscall number → sys_read

    10-20 ns        Kernel: look up fd 3 in process's file table
                    Kernel: find the file, check permissions

    20-50 ns        Kernel: check page cache for the file data
                    → Cache HIT: data already in memory
                    
    50-100 ns       Kernel: copy_to_user(buf, cached_data, 4096)
                    "Write to 0x7FFE0040 — that's a user address,
                     but it's in our page table, so it works"

   100-110 ns       Kernel: restore user registers from kernel stack
                    Kernel: execute SYSRETQ instruction

   111 ns           CPU hardware (one cycle):
                      - CPL: 0 → 3
                      - RIP → your code (right after the syscall)
                      - RSP → your user stack

   112 ns           Your code continues.
                    buf[] now contains the file data.
                    Total time: ~112 nanoseconds

                    If we had flushed the TLB: +3000-10000 ns
                    That's 30-100x slower!
```

**This is why mapping the kernel into every process matters.** A syscall that takes 100ns would take 3000ns+ if it required page table switches and TLB flushes. Your storage engine makes thousands of syscalls per second (`pread`, `pwrite`, `fsync`), so this performance difference is enormous.


---

# Part F: Connection to Your Storage Engine

## Chapter 19: The Same Pattern at Every Level

Your storage engine's buffer pool is doing THE SAME THING as the CPU's virtual memory system, but at a different level:

```
CPU Virtual Memory          Your Buffer Pool
────────────────            ────────────────
Virtual address         ←→  Page ID
Physical frame          ←→  Buffer pool frame
Page table              ←→  Page table (HashMap)
TLB                     ←→  (the HashMap IS the cache)
Page fault (load page)  ←→  Buffer pool miss (read from disk)
Dirty bit               ←→  Dirty flag on frame
Present bit             ←→  "Is this page in the pool?"
Eviction (swap to disk) ←→  LRU-K eviction (write back if dirty)
CR3 (per-process)       ←→  Per-transaction cursor/state
PCID (avoid flush)      ←→  Keeping hot pages across transactions
```

You built this! Your buffer pool in Phase 2 is a software reimplementation of what the CPU does in hardware for virtual memory. The page table is the directory, the TLB is the fast-path cache, and page faults are cache misses. Same pattern at every level of the stack.
