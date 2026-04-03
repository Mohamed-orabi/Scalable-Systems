# Building a Minimal Kernel: Process Creation & User↔Kernel Transitions

## What We're Building

A bare-metal x86_64 kernel that:
1. Boots up and takes control from the bootloader
2. Sets up the CPU structures needed for protection (GDT, IDT, TSS)
3. Creates a user-space process from scratch
4. Drops from kernel mode (ring 0) into user mode (ring 3)
5. Handles a syscall from user space back into kernel mode
6. Returns to user space again

This is the **absolute minimum** to demonstrate the full user↔kernel round trip. Every step builds on the previous one, and I'll explain *why* each piece exists, not just *what* it does.

---

## The Roadmap

```
Step 0: Boot → We're in ring 0, flat memory, no protection
   │
Step 1: GDT → Define "what ring 0 looks like" and "what ring 3 looks like"
   │
Step 2: TSS → Tell the CPU "when you enter ring 0, use THIS stack"
   │
Step 3: IDT → Tell the CPU "when a syscall/interrupt happens, jump HERE"
   │
Step 4: Paging → Create page tables that map memory for both kernel and user
   │
Step 5: User program → Put actual code in the user-space pages
   │
Step 6: IRETQ → "Fake" a return-to-user-mode to drop into ring 3
   │
Step 7: SYSCALL → User code calls back into the kernel
   │
Step 8: SYSRETQ → Kernel returns to user code
```

---

## Step 0: The Starting Point — After Boot

When your bootloader (GRUB, Limine, or a custom one) hands control to your kernel, the CPU is in a specific state:

- **Ring 0** (kernel mode) — the bootloader ran in ring 0 and so do you
- **Long mode** (64-bit) — assuming a 64-bit bootloader protocol
- **Paging enabled** — the bootloader set up identity-mapped page tables (virtual address == physical address)
- **No GDT you control** — the bootloader set up a temporary one
- **No IDT** — there's no interrupt handling; if anything goes wrong, triple fault → reboot
- **No TSS** — no mechanism for stack switching on ring transitions
- **Flat memory** — everything is accessible, no user/kernel split

**Your job:** replace all the temporary bootloader structures with your own, then build the machinery for user-space processes.

### Project Structure

```
minikernel/
├── src/
│   ├── main.rs          # Kernel entry point
│   ├── gdt.rs           # Global Descriptor Table
│   ├── tss.rs           # Task State Segment
│   ├── idt.rs           # Interrupt Descriptor Table
│   ├── paging.rs        # Page tables
│   ├── syscall.rs       # Syscall handler
│   └── user_program.rs  # The user-space binary (embedded)
├── src/boot.asm         # Assembly entry & low-level transitions
├── linker.ld            # Linker script
└── Cargo.toml
```

---

## Step 1: The GDT — Defining Privilege Levels

### What Is It?

The **Global Descriptor Table (GDT)** is a CPU structure that defines **segment descriptors**. In long mode (64-bit), segmentation is mostly disabled — the CPU ignores base/limit fields. But the GDT still serves one critical purpose: **it defines the privilege level (ring) associated with each code/data segment.**

When the CPU executes code, it checks the `CS` (Code Segment) register. The descriptor that `CS` points to determines your **Current Privilege Level (CPL)**. So:

- `CS` points to a ring-0 code segment → you're in kernel mode
- `CS` points to a ring-3 code segment → you're in user mode

### What We Need

```
GDT Entry 0: Null descriptor       (required by CPU, must be all zeros)
GDT Entry 1: Kernel Code Segment   (ring 0, execute)
GDT Entry 2: Kernel Data Segment   (ring 0, read/write)
GDT Entry 3: User Code Segment     (ring 3, execute)
GDT Entry 4: User Data Segment     (ring 3, read/write)
GDT Entry 5-6: TSS Descriptor      (takes 2 entries because it's 16 bytes in long mode)
```

### The Code

```rust
// gdt.rs

use core::mem::size_of;

/// A single 8-byte GDT entry
#[repr(C, packed)]
#[derive(Copy, Clone)]
pub struct GdtEntry {
    limit_low: u16,
    base_low: u16,
    base_middle: u8,
    access: u8,
    granularity: u8,  // also holds limit_high bits
    base_high: u8,
}

impl GdtEntry {
    /// Create a null descriptor
    const fn null() -> Self {
        GdtEntry { limit_low: 0, base_low: 0, base_middle: 0,
                    access: 0, granularity: 0, base_high: 0 }
    }

    /// Create a code/data segment descriptor
    ///
    /// In long mode, the CPU ignores base and limit for code/data segments.
    /// The only things that matter are:
    ///   - access byte: present, ring level, type (code vs data)
    ///   - the L (long mode) bit in granularity byte for code segments
    const fn new(access: u8, long_mode_code: bool) -> Self {
        GdtEntry {
            limit_low: 0xFFFF,          // Ignored in long mode, but set conventionally
            base_low: 0,
            base_middle: 0,
            // Access byte breakdown:
            //   bit 7: Present (1 = valid segment)
            //   bit 6-5: DPL (Descriptor Privilege Level) — the ring
            //   bit 4: Descriptor type (1 = code/data, 0 = system)
            //   bit 3: Executable (1 = code, 0 = data)
            //   bit 2: Direction/Conforming
            //   bit 1: Readable (code) / Writable (data)
            //   bit 0: Accessed
            access,
            // Granularity byte:
            //   bit 7: Granularity (1 = 4KB blocks)
            //   bit 6: Size (0 for long mode)
            //   bit 5: Long mode (1 = 64-bit code segment)
            //   bit 4: Available
            //   bits 3-0: limit_high
            granularity: if long_mode_code { 0xAF } else { 0xCF },
            base_high: 0,
        }
    }
}

// Access byte values:
// 0x9A = 1_00_1_1010 = Present, Ring 0, Code/Data, Executable, Readable
// 0x92 = 1_00_1_0010 = Present, Ring 0, Code/Data, Data, Writable
// 0xFA = 1_11_1_1010 = Present, Ring 3, Code/Data, Executable, Readable
// 0xF2 = 1_11_1_0010 = Present, Ring 3, Code/Data, Data, Writable
//            ^^--- these two bits are DPL (ring level)

#[repr(C, packed)]
pub struct Gdt {
    pub null:        GdtEntry,   // 0x00
    pub kernel_code: GdtEntry,   // 0x08  ← CS for ring 0
    pub kernel_data: GdtEntry,   // 0x10  ← DS/SS for ring 0
    pub user_code:   GdtEntry,   // 0x18  ← CS for ring 3 (loaded as 0x18 | 3 = 0x1B)
    pub user_data:   GdtEntry,   // 0x20  ← DS/SS for ring 3 (loaded as 0x20 | 3 = 0x23)
    pub tss_low:     GdtEntry,   // 0x28  ← TSS descriptor (lower 8 bytes)
    pub tss_high:    GdtEntry,   // 0x30  ← TSS descriptor (upper 8 bytes)
}

pub static mut GDT: Gdt = Gdt {
    null:        GdtEntry::null(),
    kernel_code: GdtEntry::new(0x9A, true),   // Ring 0, code, long mode
    kernel_data: GdtEntry::new(0x92, false),   // Ring 0, data
    user_code:   GdtEntry::new(0xFA, true),    // Ring 3, code, long mode
    user_data:   GdtEntry::new(0xF2, false),   // Ring 3, data
    tss_low:     GdtEntry::null(),             // Filled in at runtime
    tss_high:    GdtEntry::null(),             // Filled in at runtime
};

/// The GDTR register value — tells the CPU where the GDT is
#[repr(C, packed)]
pub struct GdtPtr {
    pub limit: u16,   // Size of GDT in bytes - 1
    pub base: u64,    // Physical address of the GDT
}

/// Segment selectors — the values loaded into CS, DS, SS
/// The low 2 bits are the RPL (Requested Privilege Level)
pub const KERNEL_CS: u16 = 0x08;        // GDT entry 1, ring 0
pub const KERNEL_DS: u16 = 0x10;        // GDT entry 2, ring 0
pub const USER_CS: u16   = 0x18 | 3;    // GDT entry 3, ring 3 (0x1B)
pub const USER_DS: u16   = 0x20 | 3;    // GDT entry 4, ring 3 (0x23)
pub const TSS_SEL: u16   = 0x28;        // GDT entry 5

pub unsafe fn load_gdt() {
    let gdt_ptr = GdtPtr {
        limit: (size_of::<Gdt>() - 1) as u16,
        base: &GDT as *const _ as u64,
    };

    core::arch::asm!(
        "lgdt [{0}]",          // Load the GDTR register
        "push {1:r}",          // Push new CS value
        "lea rax, [rip + 2f]", // Push address of label "2:"
        "push rax",
        "retfq",               // Far return = pops CS:RIP, effectively jump with CS change
        "2:",
        "mov ds, {2:x}",      // Load data segment registers
        "mov es, {2:x}",
        "mov ss, {2:x}",
        "mov fs, {2:x}",
        "mov gs, {2:x}",
        in(reg) &gdt_ptr,
        in(reg) KERNEL_CS as u64,
        in(reg) KERNEL_DS as u16,
    );
}
```

### Why This Matters

After `lgdt` + reloading `CS`, the CPU knows:
- "Code executing with selector 0x08 is ring 0" (kernel)
- "Code executing with selector 0x1B is ring 3" (user)

Without this, there's no concept of privilege levels. The GDT is the **definition** of the two worlds.

**Key insight about selectors:** When you see `USER_CS = 0x18 | 3 = 0x1B`, the `| 3` sets the **RPL** (Requested Privilege Level) bits. The CPU uses this to verify that a ring-3 request matches a ring-3 segment. The formula is: `selector = (GDT_index * 8) | RPL`.

---

## Step 2: The TSS — "When You Enter Ring 0, Use This Stack"

### What Is It?

The **Task State Segment (TSS)** answers one critical question: **when the CPU transitions from ring 3 to ring 0, what stack should it use?**

Why does this matter? When your user-space code triggers an interrupt or syscall, the CPU needs to start executing kernel code. Kernel code needs a stack. But the user-space stack is:
1. **Untrusted** — user code could have set `RSP` to garbage
2. **Wrong size** — user stack might be tiny; kernel needs its own
3. **Insecure** — kernel data on a user-readable stack = information leak

So the CPU looks at the TSS to find the **kernel stack pointer** (`RSP0`), and switches to it automatically on ring transitions.

### The Code

```rust
// tss.rs

/// The 64-bit Task State Segment
/// The CPU reads RSP0 from here when transitioning from ring 3 → ring 0
#[repr(C, packed)]
pub struct Tss {
    reserved0: u32,
    /// Ring 0 stack pointer — THE critical field
    /// When an interrupt or exception fires in ring 3,
    /// the CPU loads RSP from here before pushing anything
    pub rsp0: u64,
    pub rsp1: u64,      // Ring 1 stack (unused in our kernel)
    pub rsp2: u64,      // Ring 2 stack (unused)
    reserved1: u64,
    /// Interrupt Stack Table (IST) — 7 dedicated stacks for specific interrupts
    /// IST1 is commonly used for double faults, NMIs, etc.
    /// These override RSP0 for specific IDT entries
    pub ist: [u64; 7],
    reserved2: u64,
    reserved3: u16,
    /// I/O permission bitmap offset (set to size_of::<Tss>() to disable)
    pub iomap_base: u16,
}

// A 16KB kernel stack (small but sufficient for our demo)
// In a real kernel, each process gets its own kernel stack
static mut KERNEL_STACK: [u8; 16384] = [0; 16384];

pub static mut TSS: Tss = Tss {
    reserved0: 0,
    rsp0: 0,    // Will be set to top of KERNEL_STACK at init
    rsp1: 0,
    rsp2: 0,
    reserved1: 0,
    ist: [0; 7],
    reserved2: 0,
    reserved3: 0,
    iomap_base: 104, // sizeof(Tss) = 104, meaning "no I/O bitmap"
};

pub unsafe fn init_tss() {
    // Stack grows downward, so "top" = base address + size
    TSS.rsp0 = (&KERNEL_STACK as *const _ as u64) + 16384;

    // Now we need to put the TSS address into the GDT's TSS descriptor
    // The TSS descriptor in long mode is 16 bytes (2 GDT entries):
    //
    //  Bytes 0-7 (tss_low):   standard descriptor format with base[31:0] and limit
    //  Bytes 8-15 (tss_high): base[63:32] in the first 4 bytes, rest reserved
    let tss_addr = &TSS as *const _ as u64;
    let tss_limit = (core::mem::size_of::<Tss>() - 1) as u64;

    // Encode the TSS descriptor into the GDT
    let low = GdtEntry {
        limit_low: (tss_limit & 0xFFFF) as u16,
        base_low: (tss_addr & 0xFFFF) as u16,
        base_middle: ((tss_addr >> 16) & 0xFF) as u8,
        access: 0x89,  // Present, 64-bit TSS (available), not busy
        granularity: (((tss_limit >> 16) & 0x0F) as u8),
        base_high: ((tss_addr >> 24) & 0xFF) as u8,
    };

    let high = GdtEntry {
        limit_low: ((tss_addr >> 32) & 0xFFFF) as u16,
        base_low: ((tss_addr >> 48) & 0xFFFF) as u16,
        base_middle: 0, access: 0, granularity: 0, base_high: 0,
    };

    use crate::gdt::{GDT, TSS_SEL};
    GDT.tss_low = low;
    GDT.tss_high = high;

    // Tell the CPU to use this TSS
    core::arch::asm!("ltr {0:x}", in(reg) TSS_SEL);
}
```

### The Flow When An Interrupt Fires in User Mode

```
User code running (ring 3, user stack at some address)
    │
    ▼ ← interrupt/exception/syscall fires
    
CPU reads TSS.rsp0 → gets kernel stack address
CPU switches RSP to TSS.rsp0
CPU pushes onto the NEW (kernel) stack:
    ┌─────────────┐ ← new RSP (from TSS.rsp0)
    │ User SS     │  (the user data segment selector)
    │ User RSP    │  (where the user stack was)
    │ User RFLAGS │  (the user's flags register)
    │ User CS     │  (the user code segment selector)
    │ User RIP    │  (the user's instruction pointer — where to return)
    │ Error code  │  (only for some exceptions, not for syscalls)
    └─────────────┘
    
CPU loads CS from IDT entry → now CPL = 0
CPU jumps to handler address from IDT entry
    │
    ▼
Kernel handler runs on the kernel stack
```

**Without the TSS, the CPU doesn't know where to find a ring-0 stack, and will triple fault.**

---

## Step 3: The IDT — "When X Happens, Jump Here"

### What Is It?

The **Interrupt Descriptor Table (IDT)** is an array of 256 entries. Each entry tells the CPU: "when interrupt/exception number N fires, jump to this address with this privilege level."

We need at minimum:
- **Entry 13 (#GP):** General protection fault — catches privilege violations
- **Entry 14 (#PF):** Page fault handler — essential for debugging
- **Entry 0x80:** Syscall via `INT 0x80` — the classic Linux syscall gate
- **A default handler** for everything else — to prevent triple faults

### The Code

```rust
// idt.rs

/// A single IDT entry (16 bytes in long mode)
#[repr(C, packed)]
#[derive(Copy, Clone)]
pub struct IdtEntry {
    offset_low: u16,     // Handler address bits 0-15
    selector: u16,       // Code segment selector (KERNEL_CS)
    ist: u8,             // Interrupt Stack Table index (0 = use TSS.rsp0)
    type_attr: u8,       // Type and attributes
    offset_mid: u16,     // Handler address bits 16-31
    offset_high: u32,    // Handler address bits 32-63
    reserved: u32,
}

impl IdtEntry {
    const fn missing() -> Self {
        IdtEntry {
            offset_low: 0, selector: 0, ist: 0, type_attr: 0,
            offset_mid: 0, offset_high: 0, reserved: 0,
        }
    }

    /// Create an interrupt gate entry
    ///
    /// type_attr breakdown:
    ///   bit 7: Present
    ///   bits 6-5: DPL — minimum ring that can trigger this via INT instruction
    ///             For hardware interrupts/exceptions: 0 (kernel only)
    ///             For syscall via INT 0x80: 3 (user can trigger it)
    ///   bit 4: 0 (must be zero)
    ///   bits 3-0: Gate type (0xE = 64-bit interrupt gate)
    fn new(handler: u64, selector: u16, dpl: u8) -> Self {
        IdtEntry {
            offset_low: (handler & 0xFFFF) as u16,
            selector,
            ist: 0,
            type_attr: 0x8E | ((dpl & 3) << 5),
            offset_mid: ((handler >> 16) & 0xFFFF) as u16,
            offset_high: ((handler >> 32) & 0xFFFFFFFF) as u32,
            reserved: 0,
        }
    }
}

static mut IDT: [IdtEntry; 256] = [IdtEntry::missing(); 256];

#[repr(C, packed)]
struct IdtPtr {
    limit: u16,
    base: u64,
}

extern "C" {
    fn isr_general_protection();
    fn isr_page_fault();
    fn isr_timer();
    fn isr_syscall_int80();
    fn isr_default();
}

pub unsafe fn init_idt() {
    IDT[13] = IdtEntry::new(isr_general_protection as u64, KERNEL_CS, 0);
    IDT[14] = IdtEntry::new(isr_page_fault as u64, KERNEL_CS, 0);
    IDT[32] = IdtEntry::new(isr_timer as u64, KERNEL_CS, 0);

    // DPL=3 so user space can trigger it with INT 0x80!
    IDT[0x80] = IdtEntry::new(isr_syscall_int80 as u64, KERNEL_CS, 3);

    // Fill the rest to prevent triple faults
    for i in 0..256 {
        if IDT[i].type_attr == 0 {
            IDT[i] = IdtEntry::new(isr_default as u64, KERNEL_CS, 0);
        }
    }

    let idt_ptr = IdtPtr {
        limit: (core::mem::size_of::<[IdtEntry; 256]>() - 1) as u16,
        base: &IDT as *const _ as u64,
    };
    core::arch::asm!("lidt [{0}]", in(reg) &idt_ptr);
}
```

### The Assembly Stubs

Each IDT entry points to an assembly stub that saves all registers, calls a Rust function, restores registers, and does `iretq`:

```nasm
; boot.asm — interrupt stubs

isr_common:
    ; Save all general-purpose registers
    push rax
    push rbx
    push rcx
    push rdx
    push rsi
    push rdi
    push rbp
    push r8
    push r9
    push r10
    push r11
    push r12
    push r13
    push r14
    push r15

    ; Pass the stack pointer as argument (pointer to saved registers)
    mov rdi, rsp
    call interrupt_handler     ; Rust function

    ; Restore all registers
    pop r15
    pop r14
    pop r13
    pop r12
    pop r11
    pop r10
    pop r9
    pop r8
    pop rbp
    pop rdi
    pop rsi
    pop rdx
    pop rcx
    pop rbx
    pop rax

    add rsp, 16                ; Remove error code and interrupt number
    iretq                      ; ← Return from interrupt
                               ;   pops RIP, CS, RFLAGS, RSP, SS
                               ;   if CS has RPL=3 → back to ring 3!

global isr_general_protection
isr_general_protection:
    ; CPU pushes error code for #GP
    push 13                    ; interrupt number
    jmp isr_common

global isr_page_fault
isr_page_fault:
    ; CPU pushes error code for #PF
    push 14
    jmp isr_common

global isr_timer
isr_timer:
    push 0                     ; dummy error code
    push 32
    jmp isr_common

global isr_syscall_int80
isr_syscall_int80:
    push 0                     ; dummy error code
    push 0x80
    jmp isr_common

global isr_default
isr_default:
    push 0
    push 0xFF
    jmp isr_common
```

### The Rust Interrupt Handler

```rust
/// The stack frame passed to our handler — matches the push order above
#[repr(C)]
pub struct InterruptFrame {
    // Pushed by our stub
    r15: u64, r14: u64, r13: u64, r12: u64,
    r11: u64, r10: u64, r9: u64, r8: u64,
    rbp: u64, rdi: u64, rsi: u64, rdx: u64,
    rcx: u64, rbx: u64, rax: u64,
    // Pushed by our stub
    int_no: u64,
    error_code: u64,
    // Pushed by CPU on interrupt
    rip: u64,       // Where the interrupted code was
    cs: u64,        // What segment (tells us if it was ring 0 or ring 3)
    rflags: u64,
    rsp: u64,       // User stack pointer (only if coming from ring 3)
    ss: u64,        // User stack segment (only if coming from ring 3)
}

#[no_mangle]
pub extern "C" fn interrupt_handler(frame: &mut InterruptFrame) {
    match frame.int_no {
        13 => {
            panic!("#GP at RIP={:#x}, error_code={:#x}", frame.rip, frame.error_code);
        }
        14 => {
            let cr2: u64;
            unsafe { core::arch::asm!("mov {}, cr2", out(reg) cr2) };
            panic!("#PF at RIP={:#x}, fault_addr={:#x}, error={:#x}",
                   frame.rip, cr2, frame.error_code);
        }
        0x80 => {
            // Syscall! rax = number, rdi/rsi/rdx = args
            handle_syscall(frame);
        }
        32 => {
            // Timer — send EOI to PIC
            unsafe { core::arch::asm!("mov al, 0x20", "out 0x20, al") };
        }
        _ => { /* Unknown — ignore */ }
    }
}

fn handle_syscall(frame: &mut InterruptFrame) {
    match frame.rax {
        1 => {
            // sys_write(fd, buf, len)
            let buf = frame.rsi as *const u8;
            let len = frame.rdx as usize;
            // REAL KERNEL: validate buf is in user space first!
            let slice = unsafe { core::slice::from_raw_parts(buf, len) };
            vga_write(slice);
            frame.rax = len as u64; // Return bytes written
        }
        60 => {
            // sys_exit(code) — just halt for our demo
            loop { unsafe { core::arch::asm!("hlt") }; }
        }
        _ => {
            frame.rax = (-1i64) as u64; // Unknown syscall
        }
    }
}
```

---

## Step 4: Paging — Creating the Two Worlds

### What Is It?

Page tables define the **virtual → physical address mapping** for each process. This is where we actually create the user/kernel split:

- **Kernel pages:** mapped with the Supervisor bit → only ring 0 can access
- **User pages:** mapped with the User bit → ring 3 can access

### x86_64 Page Table Structure

```
Virtual address (48 bits used):
┌────────┬────────┬────────┬────────┬──────────────┐
│ PML4   │ PDPT   │  PD    │  PT    │   Offset     │
│ 9 bits │ 9 bits │ 9 bits │ 9 bits │  12 bits     │
└────────┴────────┴────────┴────────┴──────────────┘
   │         │        │        │         │
   │         │        │        │         └─ Byte within 4KB page
   │         │        │        └─ Index into Page Table (512 entries)
   │         │        └─ Index into Page Directory (512 entries)
   │         └─ Index into PD Pointer Table (512 entries)
   └─ Index into Page Map Level 4 (512 entries, the root)

Each entry is 8 bytes:
┌──────────────────────────────────────────────────────────┐
│ 63   │ 62:52   │ 51:12                │ 11:0             │
│ NX   │ Avail   │ Physical Page Addr   │ Flags            │
└──────────────────────────────────────────────────────────┘

Important flags (bits 0-11):
  bit 0 (P):    Present — page is mapped
  bit 1 (RW):   Read/Write — 0=read-only, 1=writable
  bit 2 (US):   User/Supervisor — 0=kernel only, 1=user accessible  ← THE KEY BIT
  bit 5 (A):    Accessed (set by CPU)
  bit 6 (D):    Dirty (set by CPU on write)
  bit 7 (PS):   Page Size (1 = 2MB huge page at PD level)
```

### The Code

```rust
// paging.rs

const PAGE_PRESENT: u64  = 1 << 0;
const PAGE_WRITABLE: u64 = 1 << 1;
const PAGE_USER: u64     = 1 << 2;  // ← Makes the page accessible from ring 3

/// Page table: 512 entries × 8 bytes = 4096 bytes (one page)
#[repr(C, align(4096))]
pub struct PageTable {
    entries: [u64; 512],
}

// Static page tables (a real kernel allocates these dynamically)
static mut PML4:         PageTable = PageTable { entries: [0; 512] };
static mut PDPT_KERNEL:  PageTable = PageTable { entries: [0; 512] };
static mut PD_KERNEL:    PageTable = PageTable { entries: [0; 512] };
static mut PT_USER:      PageTable = PageTable { entries: [0; 512] };

// Physical memory backing for user pages
#[repr(C, align(4096))]
pub struct UserPage { pub data: [u8; 4096] }

pub static mut USER_CODE_PAGE:  UserPage = UserPage { data: [0; 4096] };
pub static mut USER_STACK_PAGE: UserPage = UserPage { data: [0; 4096] };

pub unsafe fn init_paging() {
    // ═══════════════════════════════════════════════════════
    // KERNEL MAPPING: first 1GB identity mapped, kernel-only
    // Using 2MB huge pages for simplicity
    // ═══════════════════════════════════════════════════════

    // PML4[0] → PDPT_KERNEL (USER bit set because user pages live below too)
    PML4.entries[0] = (&PDPT_KERNEL as *const _ as u64)
        | PAGE_PRESENT | PAGE_WRITABLE | PAGE_USER;

    // PDPT_KERNEL[0] → PD_KERNEL
    PDPT_KERNEL.entries[0] = (&PD_KERNEL as *const _ as u64)
        | PAGE_PRESENT | PAGE_WRITABLE | PAGE_USER;

    // PD_KERNEL: 512 × 2MB = 1GB identity mapped
    for i in 0..512 {
        PD_KERNEL.entries[i] = (i as u64 * 0x200000)  // Physical address
            | PAGE_PRESENT | PAGE_WRITABLE
            | (1 << 7);   // PS bit = 2MB huge page
        // NO PAGE_USER → ring 3 cannot access these pages
    }

    // ═══════════════════════════════════════════════════════
    // USER MAPPING at virtual address 0x400000 (4MB)
    //
    //   0x400000 → PML4[0], PDPT[0], PD[2], PT[0]
    //   PD index = 0x400000 / 0x200000 = 2
    //
    // We replace the 2MB huge page at PD[2] with a page table
    // that has fine-grained 4KB mappings with USER bit
    // ═══════════════════════════════════════════════════════

    PD_KERNEL.entries[2] = (&PT_USER as *const _ as u64)
        | PAGE_PRESENT | PAGE_WRITABLE | PAGE_USER;

    // PT_USER[0] → user code page at virtual 0x400000
    PT_USER.entries[0] = (&USER_CODE_PAGE as *const _ as u64)
        | PAGE_PRESENT | PAGE_USER;    // Readable + executable, not writable

    // PT_USER[1] → user stack page at virtual 0x401000
    PT_USER.entries[1] = (&USER_STACK_PAGE as *const _ as u64)
        | PAGE_PRESENT | PAGE_WRITABLE | PAGE_USER;  // Stack must be writable

    // Load the new page table root into CR3
    core::arch::asm!("mov cr3, {}", in(reg) &PML4 as *const _ as u64);
}
```

### Critical Rule: PAGE_USER Must Be Set at EVERY Level

```
PML4[0]      → must have USER bit if ANY page below it is user-accessible
  └─ PDPT[0]    → must have USER bit
       └─ PD[2]     → must have USER bit
            └─ PT[0]     → has USER bit → this 4KB page is ring-3 accessible
```

If **any** level in the chain is missing the USER bit, the CPU faults with #PF when ring-3 code tries to access it. This is the most common bug when writing a first kernel.

---

## Step 5: The User Program — Code That Runs in Ring 3

We need actual machine code to place in the user-space page. This code will execute in ring 3 and make a syscall.

```rust
// user_program.rs

/// Raw x86_64 machine code for our user-space program.
///
/// This does:
///   1. Write "Hello from user space!\n" to stdout via INT 0x80
///   2. Exit with code 42 via INT 0x80
///
/// Equivalent C:
///   void _start() {
///       write(1, "Hello from user space!\n", 23);
///       exit(42);
///   }
pub static USER_PROGRAM: &[u8] = &[
    // --- sys_write(1, message, 23) ---
    // mov rax, 1              ; syscall number: write
    0x48, 0xC7, 0xC0, 0x01, 0x00, 0x00, 0x00,
    // mov rdi, 1              ; fd: stdout
    0x48, 0xC7, 0xC7, 0x01, 0x00, 0x00, 0x00,
    // mov rsi, 0x400030       ; pointer to string (at offset 0x30 in this page)
    0x48, 0xC7, 0xC6, 0x30, 0x00, 0x40, 0x00,
    // mov rdx, 23             ; length
    0x48, 0xC7, 0xC2, 0x17, 0x00, 0x00, 0x00,
    // int 0x80                ; SYSCALL — enters kernel mode!
    0xCD, 0x80,

    // --- sys_exit(42) ---
    // mov rax, 60             ; syscall number: exit
    0x48, 0xC7, 0xC0, 0x3C, 0x00, 0x00, 0x00,
    // mov rdi, 42             ; exit code
    0x48, 0xC7, 0xC7, 0x2A, 0x00, 0x00, 0x00,
    // int 0x80
    0xCD, 0x80,

    // Padding to reach offset 0x30
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00,

    // The string at offset 0x30: "Hello from user space!\n"
    b'H', b'e', b'l', b'l', b'o', b' ', b'f', b'r',
    b'o', b'm', b' ', b'u', b's', b'e', b'r', b' ',
    b's', b'p', b'a', b'c', b'e', b'!', b'\n',
];

pub unsafe fn load_user_program() {
    let dest = &mut crate::paging::USER_CODE_PAGE.data;
    dest[..USER_PROGRAM.len()].copy_from_slice(USER_PROGRAM);
}
```

---

## Step 6: The Magic Moment — Dropping to Ring 3 with IRETQ

### The Problem

We're in ring 0 and need to start executing code in ring 3. But there's no "go to ring 3" instruction. The only way to **lower** your privilege level is to **fake a return from an interrupt**.

The `iretq` instruction pops 5 values from the stack:

```
         Stack before IRETQ:
         ┌─────────────┐ ← RSP
         │ User RIP    │  → loaded into RIP (where to execute)
         │ User CS     │  → loaded into CS  (ring 3 code segment!)
         │ User RFLAGS │  → loaded into RFLAGS
         │ User RSP    │  → loaded into RSP (user stack)
         │ User SS     │  → loaded into SS  (ring 3 data segment)
         └─────────────┘
```

When the CPU loads `CS` with a ring-3 selector, it transitions to ring 3. It doesn't care that there was no actual interrupt — `iretq` is just "load these values into these registers."

### The Code

```rust
/// Drop from ring 0 into ring 3 and start executing user code
pub unsafe fn jump_to_usermode() {
    let user_code_addr: u64 = 0x400000;                // Virtual address of user code
    let user_stack_top: u64 = 0x401000 + 4096 - 8;     // Top of user stack (grows down)

    core::arch::asm!(
        // Push the 5 values that IRETQ pops, in reverse order:

        "push {user_ds:r}",      // 1. User SS (data segment, RPL=3)
        "push {user_rsp}",       // 2. User RSP
        "pushfq",                // 3. Push current RFLAGS...
        "pop rax",
        "or rax, 0x200",        //    ...set IF (Interrupt Enable Flag)
        "push rax",              //    ...push modified RFLAGS
        "push {user_cs:r}",     // 4. User CS (code segment, RPL=3)
        "push {user_rip}",      // 5. User RIP (entry point)

        // 🔥 THE TRANSITION
        "iretq",

        user_ds  = in(reg) USER_DS as u64,
        user_rsp = in(reg) user_stack_top,
        user_cs  = in(reg) USER_CS as u64,
        user_rip = in(reg) user_code_addr,
    );
    // Never reached — iretq jumps to user code
}
```

### What Happens Inside the CPU During IRETQ

```
1. Pop RIP  = 0x400000     → user code entry point
2. Pop CS   = 0x1B         → GDT entry 3, RPL=3
   └─ CPU sees RPL=3 → sets CPL=3 → 🔥 WE'RE IN RING 3 NOW
3. Pop RFLAGS = (IF set)   → interrupts enabled
4. Pop RSP  = 0x401FF8     → user stack (top of user stack page)
5. Pop SS   = 0x23         → GDT entry 4, RPL=3

CPU starts executing at 0x400000 in ring 3.
Any attempt to:
  - Execute privileged instructions → #GP
  - Access kernel memory (pages without USER bit) → #PF
  - Execute I/O instructions (if IOPL < 3) → #GP
```

---

## Step 7: User Code Calls Back — The Full Round Trip

When the user code executes `INT 0x80`:

```
User code at ring 3, executing at 0x400000+N
    │
    │   INT 0x80
    ▼
1. CPU looks up IDT[0x80]
   - Checks: DPL of gate (3) >= CPL (3) ✓ (user allowed to trigger)
   - Gets: handler address, KERNEL_CS selector

2. CPU reads TSS.rsp0 → kernel stack address

3. CPU pushes onto kernel stack:
   ┌──────────────┐
   │ SS    = 0x23  │  user data segment
   │ RSP   = ...   │  user stack pointer
   │ RFLAGS        │
   │ CS    = 0x1B  │  user code segment
   │ RIP   = ...   │  instruction after INT 0x80
   └──────────────┘

4. CPU loads CS = KERNEL_CS (0x08) → CPL = 0 → 🔥 IN RING 0

5. CPU jumps to isr_syscall_int80 (our assembly stub)

6. Stub saves all registers → calls interrupt_handler() in Rust

7. Rust handler: rax=1 → sys_write → prints to VGA

8. Stub restores registers → IRETQ
   → CPU pops SS, RSP, RFLAGS, CS, RIP
   → CS = 0x1B → back to ring 3
   → Execution continues after INT 0x80 in user code
```

---

## Step 8: Putting It All Together

```rust
// main.rs
#![no_std]
#![no_main]

mod gdt;
mod tss;
mod idt;
mod paging;
mod user_program;

fn vga_write(bytes: &[u8]) {
    static mut POS: usize = 0;
    let vga = 0xB8000 as *mut u8;
    for &b in bytes {
        if b == b'\n' {
            unsafe { POS = (POS / 160 + 1) * 160; }
            continue;
        }
        unsafe {
            *vga.add(POS) = b;
            *vga.add(POS + 1) = 0x0F;  // White on black
            POS += 2;
        }
    }
}

#[no_mangle]
pub extern "C" fn kernel_main() -> ! {
    vga_write(b"[1/5] Loading GDT...\n");
    unsafe { gdt::load_gdt(); }

    vga_write(b"[2/5] Initializing TSS...\n");
    unsafe { tss::init_tss(); }

    vga_write(b"[3/5] Setting up IDT...\n");
    unsafe { idt::init_idt(); }

    vga_write(b"[4/5] Setting up page tables...\n");
    unsafe { paging::init_paging(); }

    vga_write(b"[5/5] Loading user program...\n");
    unsafe { user_program::load_user_program(); }

    vga_write(b"--- Jumping to user mode! ---\n");

    unsafe { jump_to_usermode(); }

    loop {} // Never reached
}

#[panic_handler]
fn panic(_info: &core::panic::PanicInfo) -> ! {
    vga_write(b"!!! KERNEL PANIC !!!");
    loop { unsafe { core::arch::asm!("hlt"); } }
}
```

---

## Bonus: Using SYSCALL/SYSRET Instead of INT 0x80

`INT 0x80` works but is **slow** — it goes through the full IDT lookup. Modern kernels use `SYSCALL`/`SYSRET` which bypass the IDT using MSRs (Model-Specific Registers).

### Setting Up SYSCALL

```rust
// syscall.rs — the modern fast path

const MSR_EFER:  u32 = 0xC0000080;  // Extended Feature Enable Register
const MSR_STAR:  u32 = 0xC0000081;  // Segment selectors for SYSCALL/SYSRET
const MSR_LSTAR: u32 = 0xC0000082;  // Kernel entry point for SYSCALL
const MSR_FMASK: u32 = 0xC0000084;  // RFLAGS mask during SYSCALL

unsafe fn wrmsr(msr: u32, value: u64) {
    let low = value as u32;
    let high = (value >> 32) as u32;
    core::arch::asm!("wrmsr", in("ecx") msr, in("eax") low, in("edx") high);
}

unsafe fn rdmsr(msr: u32) -> u64 {
    let (low, high): (u32, u32);
    core::arch::asm!("rdmsr", in("ecx") msr, out("eax") low, out("edx") high);
    ((high as u64) << 32) | (low as u64)
}

pub unsafe fn init_syscall() {
    // 1. Enable SYSCALL/SYSRET in EFER
    let efer = rdmsr(MSR_EFER);
    wrmsr(MSR_EFER, efer | 1);  // Set SCE (Syscall Extensions) bit

    // 2. STAR MSR — defines segment selectors
    //
    //   bits 47:32 = SYSCALL CS → kernel code segment
    //     CPU loads: CS = STAR[47:32], SS = STAR[47:32] + 8
    //   bits 63:48 = SYSRET base
    //     CPU loads: SS = base + 8, CS = base + 16 (64-bit)
    //     CPU adds RPL=3 automatically
    //
    // ⚠️ GDT ordering must be: KernelCS, KernelDS, UserDS, UserCS
    //    (User Data BEFORE User Code!)
    //
    // SYSCALL: CS=0x08 ✓, SS=0x10 ✓
    // SYSRET:  SS=0x10+8=0x18|3=0x1B (UserDS) ✓, CS=0x10+16=0x20|3=0x23 (UserCS) ✓

    let star: u64 = (0x10u64 << 48) | (0x08u64 << 32);
    wrmsr(MSR_STAR, star);

    // 3. LSTAR — kernel entry point
    wrmsr(MSR_LSTAR, syscall_entry as u64);

    // 4. FMASK — clear IF and DF on entry
    wrmsr(MSR_FMASK, 0x600);
}

/// The SYSCALL entry point
///
/// When SYSCALL executes, the CPU does:
///   RCX = user RIP (return address)
///   R11 = user RFLAGS
///   CS  = kernel code segment
///   RIP = LSTAR (this function)
///   RSP = UNCHANGED — STILL THE USER STACK!
///
/// First job: switch to kernel stack. Anything else first = crash.
#[naked]
unsafe extern "C" fn syscall_entry() {
    core::arch::asm!(
        "swapgs",                              // Access per-CPU kernel data
        "mov [rip + saved_user_rsp], rsp",     // Save user stack
        "mov rsp, [rip + kernel_rsp]",         // Load kernel stack

        // Save user context
        "push rcx",    // User RIP
        "push r11",    // User RFLAGS
        "push rdi",    // arg1
        "push rsi",    // arg2
        "push rdx",    // arg3
        "push rax",    // syscall number

        // Call Rust: syscall_dispatch(number, arg1, arg2, arg3) → rax
        "mov rdi, rax",
        "mov rsi, [rsp + 32]",  // original rdi (arg1)
        "mov rdx, [rsp + 24]",  // original rsi (arg2)
        "mov rcx, [rsp + 16]",  // original rdx (arg3)
        "call syscall_dispatch",

        // rax = return value (already set by function)

        // Restore context (skip saved rax — we want the return value)
        "add rsp, 8",     // skip saved rax
        "pop rdx",
        "pop rsi",
        "pop rdi",
        "pop r11",        // User RFLAGS
        "pop rcx",        // User RIP

        // Restore user stack
        "mov rsp, [rip + saved_user_rsp]",
        "swapgs",

        "sysretq",        // RIP ← RCX, RFLAGS ← R11, CPL ← 3

        options(noreturn)
    );
}

#[no_mangle] static mut saved_user_rsp: u64 = 0;
#[no_mangle] static mut kernel_rsp: u64 = 0; // Set to TSS.rsp0 during init

#[no_mangle]
pub extern "C" fn syscall_dispatch(num: u64, a1: u64, a2: u64, a3: u64) -> u64 {
    match num {
        1 => { /* sys_write — same as before */ a3 }
        60 => { loop { unsafe { core::arch::asm!("hlt"); } } }
        _ => (-1i64) as u64,
    }
}
```

### Why SYSCALL Is Faster

| | INT 0x80 | SYSCALL |
|---|----------|---------|
| Handler lookup | IDT memory read | LSTAR MSR (register) |
| Privilege check | IDT gate DPL check | None (always valid) |
| Stack switch | CPU reads TSS.rsp0 | Kernel does it manually |
| Segment load | Full descriptor load | Hardcoded from STAR MSR |
| ~Cycles | 100-200 | 50-70 |

---

## The Complete Lifecycle

```
POWER ON → BIOS → Bootloader → kernel_main() [Ring 0]
   │
   ├─ load_gdt()     → CPU knows: 0x08=ring0, 0x1B=ring3
   ├─ init_tss()     → CPU knows: "ring 0 stack is at address X"
   ├─ init_idt()     → CPU knows: "INT 0x80 → jump to handler at ring 0"
   ├─ init_paging()  → Memory split: 0x000000-0x3FFFFF kernel, 0x400000+ user
   ├─ load_user_program() → Machine code copied to user page
   │
   └─ jump_to_usermode() via IRETQ
       │
       │  CPU pops CS=0x1B → CPL=3 → RING 3
       ▼
   ┌─────────────────────────────────────────┐
   │  User code at 0x400000                   │
   │  Sets rax=1, rdi=1, rsi=msg, rdx=23     │
   │  INT 0x80                                │
   └──────────────┬──────────────────────────┘
                  │
                  ▼  CPU: IDT[0x80] → switch stack → push state → CPL=0
   ┌─────────────────────────────────────────┐
   │  Kernel: handle_syscall()               │
   │  rax=1 → sys_write → VGA output        │
   │  "Hello from user space!" on screen     │
   │  IRETQ → restore state → CPL=3         │
   └──────────────┬──────────────────────────┘
                  │
                  ▼  Back in ring 3
   ┌─────────────────────────────────────────┐
   │  User code continues after INT 0x80     │
   │  rax=60, rdi=42                          │
   │  INT 0x80 → sys_exit → kernel halts     │
   └─────────────────────────────────────────┘
```

---

## Common Bugs and How to Debug Them

**Triple fault on IRETQ:** User CS/SS selectors don't match valid GDT entries, or user RIP points to unmapped page. Verify selectors match DPL=3 entries and user page has PAGE_USER at all levels.

**#GP when user code runs:** A page table parent entry (PML4/PDPT/PD) is missing PAGE_USER. Walk the entire chain.

**#PF on SYSCALL entry:** After SYSCALL, RSP still points to user stack. If kernel pushes before switching stacks, the page might not be kernel-accessible. Switch RSP first, always.

**Infinite loop after SYSRETQ:** RCX (user RIP) was clobbered during handler. Save/restore it carefully.

**Immediate reboot:** Triple fault from unhandled exception. Set up a double fault handler (IDT[8]) with its own IST stack.

---

## What a Real Kernel Adds

| Feature | Why |
|---------|-----|
| Physical page allocator | Static pages don't scale — real kernels use buddy + slab |
| Per-process page tables | Each process gets its own PML4 with different user mappings |
| Scheduler | Timer interrupt preempts, saves state, loads another process |
| Context switch | Save all regs + CR3 + kernel stack, load another process's |
| ELF loader | Parse ELF binaries, map segments at correct addresses |
| `copy_from_user()` | Validate user pointers before dereferencing |
| KPTI | Separate page tables for user vs kernel (Meltdown fix) |
| vDSO | Kernel page in user space for fast calls that don't need ring 0 |

---

## How to Run This

```bash
# Using QEMU (easiest)
cargo build --target x86_64-unknown-none
qemu-system-x86_64 -kernel target/x86_64-unknown-none/debug/minikernel \
    -serial stdio -no-reboot -d int,cpu_reset
```

The `-d int,cpu_reset` flag prints every interrupt — invaluable for debugging.

### Recommended Frameworks

Instead of writing every bit from scratch, use the Rust OSDev ecosystem:
- **`bootloader` crate:** handles boot, hands you a 64-bit kernel
- **`x86_64` crate:** safe abstractions for GDT, IDT, TSS, paging
- **Philipp Oppermann's blog_os (os.phil-opp.com):** builds up to this exact point step by step
