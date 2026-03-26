# Deep Dive: Syscall Entry — The Privilege Switch

## Every Register, Every Nanosecond, Every Data Structure

---

# Chapter 1: Where We Are Right Now

Your program just executed the `syscall` instruction. Let's freeze time
and look at the EXACT state of the CPU right before this instruction runs.

```
THE CPU STATE RIGHT BEFORE "syscall" EXECUTES:

  ┌─────────────────────────────────────────────────────────────┐
  │  REGISTERS (your program's values):                         │
  │                                                             │
  │  RAX = 1              ← syscall number (1 = SYS_write)     │
  │  RDI = 3              ← arg1: file descriptor               │
  │  RSI = 0x7FFE0040     ← arg2: pointer to your buffer       │
  │  RDX = 5              ← arg3: byte count                   │
  │  R10 = (unused)       ← arg4 (not needed for write)        │
  │  R8  = (unused)       ← arg5                               │
  │  R9  = (unused)       ← arg6                               │
  │                                                             │
  │  RIP = 0x00401234     ← your code: the syscall instruction │
  │  RSP = 0x7FFE0100     ← your USER stack pointer            │
  │  RFLAGS = 0x00000202  ← processor flags (interrupts on)    │
  │                                                             │
  │  CS = 0x33            ← code segment: CPL=3 (user mode)    │
  │                       ← the "3" in 0x33 means ring 3       │
  │                                                             │
  │  CR3 = 0x1A000        ← page table base: THIS process's    │
  │                          page table physical address        │
  ├─────────────────────────────────────────────────────────────┤
  │  HIDDEN REGISTERS (set at boot time, never change):        │
  │                                                             │
  │  MSR_LSTAR = 0xFFFFFFFF81000000                             │
  │    ← "When syscall runs, jump to THIS address"              │
  │    ← This is the kernel's entry_SYSCALL_64 function         │
  │    ← Set by the kernel during boot, never changes           │
  │                                                             │
  │  MSR_STAR = 0x0023001000000000                              │
  │    ← Contains the CS/SS values for kernel and user mode     │
  │    ← Kernel CS=0x10 (ring 0), User CS=0x33 (ring 3)        │
  │                                                             │
  │  MSR_SFMASK = 0x00000200                                    │
  │    ← Which RFLAGS bits to clear during syscall              │
  │    ← 0x200 = clear IF (interrupt flag) → disable interrupts │
  │    ← Must disable interrupts until kernel stack is set up   │
  └─────────────────────────────────────────────────────────────┘
```

**Why are RAX, RDI, RSI, RDX used for syscall arguments?**

This is a convention agreed upon by the Linux kernel and libc. The kernel
says: "When you want to make a syscall, put the syscall number in RAX,
the first argument in RDI, second in RSI, third in RDX, fourth in R10,
fifth in R8, sixth in R9. Then execute the `syscall` instruction."

This is documented in the Linux x86-64 ABI. All 6 arguments are passed
in registers — never on the stack. This makes the syscall boundary fast
because there's nothing to copy from user stack to kernel stack.

```
Syscall argument convention (Linux x86-64):

  RAX = syscall number
  RDI = argument 1
  RSI = argument 2
  RDX = argument 3
  R10 = argument 4    (note: R10, not RCX — because syscall clobbers RCX)
  R8  = argument 5
  R9  = argument 6
  
  Return value → RAX (after syscall returns)

For write(fd=3, buf=0x7FFE0040, count=5):
  RAX = 1 (SYS_write)
  RDI = 3 (fd)
  RSI = 0x7FFE0040 (buf)
  RDX = 5 (count)
```


---

# Chapter 2: The SYSCALL Instruction — What the CPU Does in Hardware

The `syscall` instruction is a SINGLE CPU instruction that does 6 things
atomically (in one cycle, as one indivisible operation):

```
WHAT THE CPU HARDWARE DOES (not software — actual silicon logic):

Step 1:  RCX ← RIP
         Save the CURRENT instruction pointer into RCX.
         RCX now holds 0x00401236 (address AFTER the syscall instruction).
         This is the return address — where to come back to.

Step 2:  R11 ← RFLAGS
         Save the current processor flags into R11.
         R11 now holds 0x00000202.
         This saves interrupt enable state, comparison results, etc.

Step 3:  RFLAGS ← RFLAGS AND NOT(MSR_SFMASK)
         Clear specific flag bits. MSR_SFMASK = 0x200 (IF flag).
         This DISABLES INTERRUPTS.
         
         Why? We're about to switch stacks. If an interrupt fires
         BETWEEN switching the privilege level and switching the stack,
         the interrupt handler would use the WRONG stack. Disabling
         interrupts prevents this race condition.

Step 4:  CS ← kernel code segment (from MSR_STAR)
         CS changes from 0x33 (ring 3) to 0x10 (ring 0).
         
         ★ THIS IS THE PRIVILEGE SWITCH ★
         
         The CPU is now in ring 0 (kernel mode).
         From this moment, ALL memory access checks use ring 0 rules:
         - Pages with User/Supervisor=0 are now ACCESSIBLE
         - Privileged instructions (CLI, HLT, etc.) are now ALLOWED
         - I/O ports are now accessible

Step 5:  SS ← kernel stack segment (from MSR_STAR)
         SS changes from 0x2B (user data) to 0x18 (kernel data).
         (On x86-64 this is mostly ceremonial — segments aren't really
          used for memory protection anymore, that's the page table's job.
          But the CPU still requires valid segment values.)

Step 6:  RIP ← MSR_LSTAR
         Jump to the kernel's syscall entry point.
         RIP changes from 0x00401236 (your code) to 
         0xFFFFFFFF81000000 (kernel's entry_SYSCALL_64).

         The CPU starts executing kernel code at the very next cycle.
```

```
REGISTER STATE AFTER syscall INSTRUCTION:

  BEFORE (your program)          AFTER (kernel entry)
  ─────────────────────          ──────────────────────
  RAX = 1 (SYS_write)           RAX = 1         (unchanged — kernel reads this)
  RDI = 3 (fd)                  RDI = 3         (unchanged — kernel reads this)
  RSI = 0x7FFE0040 (buf)        RSI = 0x7FFE0040 (unchanged)
  RDX = 5 (count)               RDX = 5         (unchanged)
  
  RCX = (whatever)              RCX = 0x00401236 ★ overwritten with return addr
  R11 = (whatever)              R11 = 0x00000202 ★ overwritten with saved flags
  
  RIP = 0x00401234              RIP = 0xFFFFFFFF81000000 ★ kernel entry
  RSP = 0x7FFE0100              RSP = 0x7FFE0100 (unchanged! NOT switched yet!)
  
  CS = 0x33 (ring 3)            CS = 0x10 (ring 0) ★ KERNEL MODE
  
  CR3 = 0x1A000                 CR3 = 0x1A000    (unchanged — same page table!)
```

**CRITICAL**: Notice that RSP (the stack pointer) is NOT changed by the
`syscall` instruction! The CPU is now in kernel mode but STILL using the
user stack. This is dangerous — the very first thing the kernel code must
do is switch to the kernel stack. This is why interrupts were disabled in
step 3 — an interrupt now would try to push data onto the user stack
while in kernel mode, which is a security and stability disaster.


---

# Chapter 3: Why RCX and R11 Are "Destroyed"

The `syscall` instruction overwrites RCX and R11 with the return address
and flags. Whatever your program had in those registers is GONE.

```
YOUR PROGRAM:

  mov rcx, 42          ; RCX = 42 (some value you were using)
  mov r11, 99          ; R11 = 99 (another value)
  
  ; Set up syscall arguments
  mov rax, 1           ; SYS_write
  mov rdi, 3           ; fd
  mov rsi, buffer      ; buf
  mov rdx, 5           ; count
  
  syscall              ; RCX is now DESTROYED (holds return address)
                       ; R11 is now DESTROYED (holds saved flags)
  
  ; After syscall returns:
  ; RCX ≠ 42 anymore! It's the return address.
  ; R11 ≠ 99 anymore! It's the saved flags.
```

This is why the Linux syscall convention uses R10 for the 4th argument
instead of RCX — because RCX gets clobbered. The C calling convention
uses RCX for the 4th argument, but the syscall convention can't.

```
Normal C function call:  args in RDI, RSI, RDX, RCX, R8, R9
Linux syscall:           args in RDI, RSI, RDX, R10, R8, R9
                                                 ^^^ R10 not RCX!
```

The libc wrapper handles this translation:

```c
// Inside glibc's write() wrapper:
long write(int fd, const void *buf, size_t count) {
    long result;
    // The compiler's C calling convention puts args in RDI, RSI, RDX
    // which is already correct for syscall — no register shuffling needed
    // for write() (it only has 3 args).
    
    // For syscalls with 4+ args, glibc moves arg4 from RCX to R10:
    // asm("mov %%rcx, %%r10" ::: "r10");
    
    asm volatile(
        "syscall"
        : "=a" (result)         // output: RAX = return value
        : "a" (SYS_write),     // input: RAX = syscall number
          "D" (fd),             // input: RDI = fd
          "S" (buf),            // input: RSI = buf
          "d" (count)           // input: RDX = count
        : "rcx", "r11", "memory" // clobbered: RCX and R11 are destroyed
    );
    return result;
}
```

The `"rcx", "r11"` clobber list tells the compiler: "After this instruction,
RCX and R11 have unpredictable values. Don't keep anything important there."


---

# Chapter 4: entry_SYSCALL_64 — The First Kernel Code

Now the CPU is executing at `0xFFFFFFFF81000000`, which is the Linux kernel's
`entry_SYSCALL_64` function (defined in `arch/x86/entry/entry_64.S`).

This is assembly code — the kernel can't use C yet because we don't have
a proper stack set up.

```asm
; arch/x86/entry/entry_64.S (simplified, real code is more complex)

entry_SYSCALL_64:
    ; ─── STEP 1: SWITCH TO THE KERNEL STACK ───
    ;
    ; We're still on the user stack (RSP = 0x7FFE0100).
    ; We MUST switch to the kernel stack before doing anything else.
    ;
    ; Where is the kernel stack? The kernel stores a pointer to each
    ; process's kernel stack in a per-CPU variable. On x86-64, this
    ; is accessed via the GS segment register.
    
    swapgs                  ; Switch the GS base register from user to kernel
                            ; Now GS points to the kernel's per-CPU data area
                            ; which contains a pointer to the current process's
                            ; kernel stack.
    
    mov rsp, [gs:kernel_stack_top]
                            ; RSP now points to the TOP of this process's
                            ; kernel stack. We're safe!
                            ;
                            ; RSP was: 0x7FFE0100 (user stack)
                            ; RSP is now: 0xFFFFC90001234000 (kernel stack)
```

### What is swapgs?

`swapgs` is a special instruction that swaps the GS segment base register
between two values: one for user space, one for kernel space.

```
BEFORE swapgs:
  GS base = user's GS value (typically pointing to thread-local storage)
  MSR_KERNEL_GS_BASE = 0xFFFF888000000000 (kernel per-CPU data)

AFTER swapgs:
  GS base = 0xFFFF888000000000 (kernel per-CPU data)
  MSR_KERNEL_GS_BASE = user's old GS value (saved for later)

This swap lets the kernel access per-CPU data structures using
instructions like: mov rsp, [gs:0x28]  ← reads from per-CPU data
```

The per-CPU data area contains, among other things, a pointer to the
current process's kernel stack:

```
Per-CPU Data Area:
  ┌──────────────────────────────────────────┐
  │ Offset 0x00: current_task pointer        │ → points to task_struct
  │ Offset 0x08: kernel_stack_top            │ → top of kernel stack ★
  │ Offset 0x10: irq_count                   │
  │ Offset 0x18: cr3 value                   │
  │ ...                                      │
  └──────────────────────────────────────────┘
```


---

# Chapter 5: Saving User Registers Onto the Kernel Stack

Now we have a valid kernel stack. The next job: save ALL user registers
so we can restore them when returning from the syscall.

```asm
    ; ─── STEP 2: SAVE ALL USER REGISTERS ───
    ;
    ; Push them onto the kernel stack in a specific order.
    ; This creates a "pt_regs" structure on the stack.
    
    push    r11             ; saved RFLAGS (put here by syscall instruction)
    push    rcx             ; saved RIP (return address, from syscall)
    push    rax             ; syscall number (also return value slot)
    push    rdi             ; arg1: fd
    push    rsi             ; arg2: buf pointer  
    push    rdx             ; arg3: count
    push    r8              ; arg5 (unused for write, but save anyway)
    push    r9              ; arg6 (unused for write, but save anyway)
    push    r10             ; arg4 (unused for write, but save anyway)
    push    rbx             ; callee-saved register (your program's value)
    push    rbp             ; callee-saved register  
    push    r12             ; callee-saved register
    push    r13             ; callee-saved register
    push    r14             ; callee-saved register
    push    r15             ; callee-saved register
    
    ; Also save the user's RSP (which we replaced in step 1):
    ; The old RSP was saved by the CPU in a special MSR during syscall,
    ; or was stored in the per-CPU area. Push it too.
    push    [gs:user_rsp]   ; saved user stack pointer
```

After all these pushes, the kernel stack looks like:

```
KERNEL STACK AFTER SAVING REGISTERS:

  High address (stack top, where we started)
  ┌────────────────────────────────────────────┐
  │  (empty space above)                        │
  ├────────────────────────────────────────────┤
  │  user RSP:    0x7FFE0100                    │  ← saved user stack pointer
  │  saved R11:   0x00000202 (user RFLAGS)      │  ← will restore flags on return
  │  saved RCX:   0x00401236 (user RIP)         │  ← will return HERE
  │  RAX:         1 (SYS_write)                 │  ← syscall number / return value
  │  RDI:         3 (fd)                        │  ← argument 1
  │  RSI:         0x7FFE0040 (buf)              │  ← argument 2
  │  RDX:         5 (count)                     │  ← argument 3
  │  R8:          (don't care)                  │  ← argument 5
  │  R9:          (don't care)                  │  ← argument 6
  │  R10:         (don't care)                  │  ← argument 4
  │  RBX:         (your program's value)        │  ← must restore
  │  RBP:         (your program's value)        │  ← must restore
  │  R12:         (your program's value)        │  ← must restore
  │  R13:         (your program's value)        │  ← must restore
  │  R14:         (your program's value)        │  ← must restore
  │  R15:         (your program's value)        │  ← must restore
  ├────────────────────────────────────────────┤
  │  RSP points here now ──────────────────►    │
  │  (free space for kernel function calls)     │
  │                                             │
  │  ...                                        │
  └────────────────────────────────────────────┘
  Low address (stack bottom)

This layout IS the struct pt_regs in the kernel:

  struct pt_regs {
      unsigned long r15, r14, r13, r12;
      unsigned long rbp, rbx;
      unsigned long r10, r9, r8;
      unsigned long rdx, rsi, rdi;    // syscall arguments
      unsigned long orig_rax;          // syscall number
      unsigned long rip, cs;           // return address
      unsigned long flags;             // saved RFLAGS
      unsigned long rsp, ss;           // saved user stack
  };
```

**Why save ALL registers, not just the ones write() uses?**

The kernel doesn't know which registers your program was using. Maybe you
had important values in R12, R13, RBX, etc. The kernel must preserve them
ALL so your program continues correctly after the syscall returns. If the
kernel accidentally changed R12 and your program was using it to hold a
loop counter, your program would silently produce wrong results.


---

# Chapter 6: The Syscall Table Lookup

Now the kernel reads the syscall number from RAX and looks up which
function to call.

```asm
    ; ─── STEP 3: LOOK UP THE SYSCALL TABLE ───
    
    ; RAX still holds the syscall number (1 = SYS_write)
    ; But first, validate it:
    
    cmp     rax, NR_syscalls    ; is RAX < total number of syscalls (~450)?
    jae     .bad_syscall         ; if not, return -ENOSYS (bad syscall number)
    
    ; Look up the function pointer in the syscall table:
    ;
    ; sys_call_table is an array of function pointers:
    ;   sys_call_table[0] = &sys_read
    ;   sys_call_table[1] = &sys_write    ← THIS ONE
    ;   sys_call_table[2] = &sys_open
    ;   sys_call_table[3] = &sys_close
    ;   ...
    ;   sys_call_table[449] = &sys_futex_waitv
    
    mov     rdi, [rsp + offsetof_rdi]   ; reload RDI from saved pt_regs
    mov     rsi, [rsp + offsetof_rsi]   ; reload RSI
    mov     rdx, [rsp + offsetof_rdx]   ; reload RDX
    
    call    [sys_call_table + rax * 8]
    ;       ^^^^^^^^^^^^^^^^^^^^^^^^
    ;       This is: sys_call_table[1] = address of sys_write
    ;       which calls sys_write(fd=3, buf=0x7FFE0040, count=5)
```

### The Syscall Table

The syscall table is a simple array defined at compile time:

```c
// arch/x86/entry/syscall_64.c

const sys_call_ptr_t sys_call_table[NR_syscalls] = {
    [0]   = sys_read,          // SYS_read
    [1]   = sys_write,         // SYS_write      ★
    [2]   = sys_open,          // SYS_open
    [3]   = sys_close,         // SYS_close
    [4]   = sys_stat,          // SYS_stat
    [5]   = sys_fstat,         // SYS_fstat
    [6]   = sys_lstat,         // SYS_lstat
    [7]   = sys_poll,          // SYS_poll
    [8]   = sys_lseek,         // SYS_lseek
    [9]   = sys_mmap,          // SYS_mmap
    [10]  = sys_mprotect,      // SYS_mprotect
    // ...
    [74]  = sys_fsync,         // SYS_fsync
    // ...
    [449] = sys_futex_waitv,   // latest syscall
};

// The table is ~3600 bytes (450 entries × 8 bytes each).
// It's in kernel memory, so this lookup is just:
//   1. Multiply RAX by 8 (pointer size)
//   2. Add to table base address
//   3. Read 8 bytes (the function pointer)
//   4. Call that address
// Total: ~3-5 CPU cycles. Effectively instant.
```

**Where do syscall numbers come from?**

They're defined in a header file and NEVER change (for backward compatibility):

```c
// include/uapi/asm-generic/unistd.h (simplified)
#define __NR_read           0
#define __NR_write          1
#define __NR_open           2
#define __NR_close          3
// ...
#define __NR_fsync          74
// ...

// If Linux adds a new syscall, it gets the NEXT number.
// Old numbers are NEVER reused or changed.
// A binary compiled 20 years ago still works because
// SYS_write is still 1, SYS_read is still 0, etc.
```


---

# Chapter 7: Calling sys_write()

The `call` instruction pushes a return address onto the kernel stack and
jumps to `sys_write`. Now we're in regular C code:

```c
// fs/read_write.c (simplified)

SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf,
                size_t, count)
{
    // Arguments arrived via registers:
    //   fd    = 3              (was in RDI)
    //   buf   = 0x7FFE0040    (was in RSI)  
    //   count = 5              (was in RDX)
    
    struct fd f = fdget_pos(fd);
    // This does:
    //   1. current->files is the current process's file descriptor table
    //   2. current->files->fdt->fd[3] is the struct file for fd 3
    //   3. Returns struct fd { struct file *file; unsigned int flags; }
    
    if (!f.file)
        return -EBADF;    // "Bad file descriptor" — fd 3 doesn't exist
    
    ssize_t ret = vfs_write(f.file, buf, count, &f.file->f_pos);
    // This calls into the VFS layer (the next step in the pipeline)
    
    fdput_pos(f);
    return ret;    // Return value goes into RAX
}
```

### What is "current"?

`current` is a macro that gives you the `task_struct` for the currently
running process. On x86-64, it's read from the per-CPU data area:

```c
// How "current" works:

#define current  get_current()

static inline struct task_struct *get_current(void) {
    return this_cpu_read_stable(current_task);
    // This reads from the GS-segment per-CPU data area
    // which we set up with swapgs in step 1.
    // It's essentially: *(struct task_struct **)(gs_base + 0x00)
}

// current points to:
struct task_struct {
    pid_t                   pid;           // process ID
    char                    comm[16];      // process name ("ferrisdb")
    struct files_struct     *files;        // ★ file descriptor table
    struct mm_struct        *mm;           // ★ memory mappings (page table)
    void                    *stack;        // ★ kernel stack base
    struct thread_struct    thread;        // CPU register state
    // ... hundreds more fields
};
```

### How fdget_pos() finds the file

```c
// The file descriptor table lookup:

current->files                          // struct files_struct
  └── ->fdt                             // struct fdtable
       └── ->fd                         // struct file **fd  (array of pointers)
            └── [3]                     // struct file * at index 3
                 │
                 ▼
              struct file {
                  struct path     f_path;     // dentry + mount
                  struct inode    *f_inode;   // inode 4521
                  loff_t          f_pos;      // 8192 (current offset)
                  unsigned int    f_flags;    // O_WRONLY
                  fmode_t         f_mode;     // FMODE_WRITE
                  struct file_operations *f_op; // ext4_file_operations
                  struct address_space *f_mapping; // page cache
              };

The lookup is:
  1. current → task_struct        (per-CPU data read, ~1 cycle)
  2. task_struct → files          (pointer dereference, ~1 cycle)
  3. files → fdt → fd            (two pointer dereferences, ~2 cycles)
  4. fd[3]                        (array index, ~1 cycle)
  Total: ~5 cycles = ~2 nanoseconds

This is why fd is an integer index — it's an O(1) array lookup.
If fd were a string (like a filename), the lookup would be much slower.
```


---

# Chapter 8: The __user Annotation — Trusting Nobody

Notice the `__user` annotation on the buffer pointer:

```c
SYSCALL_DEFINE3(write, unsigned int, fd,
                const char __user *, buf,   // ← __user means "this points
                size_t, count)              //    to USER-SPACE memory"
```

`__user` is a compile-time annotation (does nothing at runtime) that tells
kernel developers and static analysis tools: "This pointer points to
user-space memory. You MUST use copy_from_user() to read it. You CANNOT
dereference it directly."

```
WHY?

If the user passes buf = 0xFFFFFFFF81000000 (a kernel address):

  WRONG (direct dereference):
    memcpy(dest, buf, count);
    → This reads from KERNEL memory!
    → The user just read arbitrary kernel data!
    → SECURITY VULNERABILITY!

  RIGHT (copy_from_user):
    copy_from_user(dest, buf, count);
    → Kernel checks: is 'buf' in the user address range?
    → If buf >= TASK_SIZE (start of kernel space): return -EFAULT
    → If the page at 'buf' isn't mapped: return -EFAULT
    → Only if valid: do the copy
    → SAFE!
```

`copy_from_user` internally does:

```c
long copy_from_user(void *to, const void __user *from, long n) {
    // Step 1: Check that 'from' is in user address range
    if (access_ok(from, n)) {    // is from < 0xC0000000 (32-bit)
                                  // or < 0x7FFFFFFFFFFF (64-bit)?
        
        // Step 2: Do the actual copy with fault handling
        // If the page isn't mapped, a page fault occurs.
        // The kernel's page fault handler checks:
        //   "Was this fault during a copy_from_user?"
        //   If yes: return -EFAULT (don't crash the kernel!)
        //   If no: it's a real kernel bug, panic.
        
        n = raw_copy_from_user(to, from, n);
    }
    return n;  // returns 0 on success, >0 = bytes NOT copied
}
```

**This is defense in depth**: even though ring 0 CAN access all memory,
the kernel voluntarily checks every user-provided pointer. A malicious
program cannot trick the kernel into reading/writing kernel memory.


---

# Chapter 9: The Return Path — Giving Control Back

After `sys_write()` returns (with return value in RAX), the assembly code
in `entry_SYSCALL_64` resumes to return to user space:

```asm
    ; sys_write returned. RAX = 5 (bytes written), or negative error.
    
    ; Store the return value into the saved pt_regs on the stack
    ; (so it ends up in user RAX when we restore registers)
    mov     [rsp + offsetof_rax], rax
    
    ; ─── RESTORE ALL USER REGISTERS ───
    
    pop     r15
    pop     r14
    pop     r13
    pop     r12
    pop     rbp
    pop     rbx
    pop     r10
    pop     r9
    pop     r8
    pop     rdx             ; restored: 5 (but user doesn't need this anymore)
    pop     rsi             ; restored: 0x7FFE0040
    pop     rdi             ; restored: 3
    pop     rax             ; restored: 5 (the return value from sys_write!)
    pop     rcx             ; restored: 0x00401236 (user return address)
    pop     r11             ; restored: 0x00000202 (user RFLAGS)
    
    ; Restore user stack pointer
    mov     rsp, [gs:user_rsp]
    
    ; Switch GS back to user mode
    swapgs
    
    ; ─── RETURN TO USER SPACE ───
    sysretq
```

### What sysretq Does (Hardware)

The `sysretq` instruction is the mirror of `syscall`:

```
WHAT sysretq DOES (hardware, one instruction):

Step 1:  RIP ← RCX
         Jump back to 0x00401236 (the instruction after your syscall).

Step 2:  RFLAGS ← R11
         Restore the processor flags (including re-enabling interrupts).

Step 3:  CS ← user code segment (from MSR_STAR)
         CS changes from 0x10 (ring 0) to 0x33 (ring 3).
         
         ★ BACK TO USER MODE ★
         
         From this moment, kernel pages are inaccessible again.
         Privileged instructions will fault again.

Step 4:  SS ← user stack segment (from MSR_STAR)

REGISTER STATE AFTER sysretq:

  RAX = 5              ← return value (bytes written)
  RDI = 3              ← restored
  RSI = 0x7FFE0040     ← restored
  RDX = 5              ← restored
  RBX = (your value)   ← restored perfectly
  RBP = (your value)   ← restored
  R12-R15 = (yours)    ← all restored
  
  RCX = 0x00401236     ← clobbered (now holds RIP, not your old value)
  R11 = 0x00000202     ← clobbered (now holds RFLAGS, not your old value)
  
  RIP = 0x00401236     ← back in your code!
  RSP = 0x7FFE0100     ← back on your user stack!
  CS = 0x33            ← ring 3 (user mode)
  
  CR3 = 0x1A000        ← NEVER CHANGED throughout the entire syscall
```


---

# Chapter 10: The Complete Timeline

```
Time   What                              Where            Ring  Stack
─────  ──────────────────────────────     ────────────     ────  ──────────
0 ns   mov rax, 1                        your code        3     user
0.3ns  mov rdi, 3                        your code        3     user  
0.6ns  mov rsi, 0x7FFE0040               your code        3     user
0.9ns  mov rdx, 5                        your code        3     user
1.2ns  syscall                           your code        3→0   user→NONE
       ├─ RCX ← return address
       ├─ R11 ← saved flags
       ├─ disable interrupts
       ├─ CS ← ring 0
       └─ RIP ← kernel entry
1.5ns  swapgs                            entry_SYSCALL_64 0     user→kernel
1.8ns  mov rsp, [gs:kernel_stack]        entry_SYSCALL_64 0     kernel
2.0ns  push r11, rcx, rax, rdi...        entry_SYSCALL_64 0     kernel
3.0ns  cmp rax, NR_syscalls              entry_SYSCALL_64 0     kernel
3.2ns  call sys_call_table[1]            entry_SYSCALL_64 0     kernel
3.5ns  ─── now inside sys_write() ───    fs/read_write.c  0     kernel
4.0ns  fdget_pos(3) → struct file        fs/read_write.c  0     kernel
5.0ns  vfs_write(file, buf, 5, &pos)     fs/read_write.c  0     kernel
       ├─ security check
       ├─ call f_op->write_iter()
       └─ ... (the write path continues from here)

       ... (50-100ns of write processing) ...

~100ns sys_write returns, rax = 5        fs/read_write.c  0     kernel
101ns  pop r15, r14, r13...              entry_SYSCALL_64 0     kernel
103ns  swapgs                            entry_SYSCALL_64 0     kernel→user
103ns  sysretq                           entry_SYSCALL_64 0→3   kernel→user
       ├─ RIP ← RCX (back to your code)
       ├─ RFLAGS ← R11
       └─ CS ← ring 3
104ns  (your next instruction)           your code        3     user

Total overhead of ENTERING and LEAVING the kernel: ~4-8 ns
The actual work (vfs_write, page cache, etc): ~50-100 ns
```


---

# Chapter 11: What Could Go Wrong

## 11.1: Invalid fd

```
Your code: write(999, buf, 5)

  sys_write(fd=999):
    fdget_pos(999) → fd_table[999] → NULL (no file at fd 999)
    return -EBADF  (Bad file descriptor)
    
  Your program: write() returned -1, errno = EBADF
```

## 11.2: Invalid buffer pointer

```
Your code: write(3, 0xDEADBEEF, 5)  // 0xDEADBEEF is not a valid address

  Eventually reaches copy_from_user(dest, 0xDEADBEEF, 5):
    access_ok(0xDEADBEEF, 5) → FALSE (not in user address range, or unmapped)
    return -EFAULT (Bad address)
    
  Your program: write() returned -1, errno = EFAULT
```

## 11.3: Null pointer (buf = 0)

```
Your code: write(3, NULL, 5)

  copy_from_user(dest, NULL, 5):
    access_ok(NULL, 5) → FALSE on modern kernels (page 0 is always unmapped)
    return -EFAULT
    
  Your program: write() returned -1, errno = EFAULT
  (The kernel does NOT crash — it gracefully returns an error)
```

## 11.4: Buffer pointing to kernel space

```
Your code: write(3, 0xFFFFFFFF81000000, 5)  // kernel address!

  copy_from_user(dest, 0xFFFFFFFF81000000, 5):
    access_ok(0xFFFFFFFF81000000, 5) → FALSE (above TASK_SIZE)
    return -EFAULT
    
  The kernel NEVER reads from the address. The check prevents it.
  This is why __user and copy_from_user exist — defense against malicious input.
```


---

# Summary: The Mental Model

```
YOUR PROGRAM (ring 3)          THE KERNEL (ring 0)
═══════════════════            ═══════════════════

mov rax, 1 (SYS_write)
mov rdi, 3 (fd)
mov rsi, buf_ptr
mov rdx, 5 (count)
                              
syscall ─────────────────►    entry_SYSCALL_64:
  │ CPL: 3 → 0                  swapgs
  │ RCX ← return addr           switch to kernel stack
  │ R11 ← flags                 save ALL user registers
  │ interrupts off               │
  │ RIP → kernel entry           validate syscall number
                                  │
                                  call sys_call_table[1]
                                  │
                                  sys_write(3, 0x7FFE0040, 5):
                                    fdget_pos(3) → struct file
                                    vfs_write(file, buf, 5, &pos)
                                    │
                                    (the I/O pipeline continues...)
                                    │
                                    return 5
                                  │
                                  restore ALL user registers
                                  RAX = 5 (return value)
                                  swapgs
                                  │
your next instruction ◄────── sysretq
  RAX = 5 (success!)            │ CPL: 0 → 3
  all other registers            │ RIP ← RCX (your code)
  perfectly restored             │ RFLAGS ← R11
  (except RCX, R11)              │ interrupts back on
```

The entire round trip adds about 4-8 nanoseconds of overhead to your
syscall. The actual work (writing to the page cache, or whatever the
syscall does) is on top of that. This is why system calls are "cheap
but not free" — 4ns is nothing for a single call, but if you're making
millions of calls per second, it adds up. This is why your storage engine
does buffered I/O (batch many small writes into one large write syscall).
