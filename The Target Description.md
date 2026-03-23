# Chapter 11: The Target Description — Comprehensive Explanation

## Overview and Big Picture

This chapter is about one of the most ambitious tasks you can undertake with LLVM: **adding an entirely new CPU backend**. LLVM's architecture is deliberately flexible — it separates the front-end (parsing source code into IR) from the back-end (turning IR into machine code for a specific CPU). The back-end is driven primarily by something called the **target description**, written in LLVM's own domain-specific language called **TableGen**. From this single description, LLVM can auto-generate large amounts of C++ code for the assembler, disassembler, instruction selection, and more.

The chapter walks through building a backend for the **Motorola M88k** architecture — a 1980s RISC CPU that is long out of production but serves as an excellent pedagogical example. By the end of the chapter, you will have a working assembler (text → binary) and disassembler (binary → text) for this architecture, all integrated into the LLVM toolchain.

Think of this like building a translator. The "target description" is the dictionary and grammar rules of the M88k CPU's language. From that dictionary, LLVM's TableGen tool automatically generates the code that can read and write that language. Your job as a backend developer is to write the dictionary accurately and then provide a small amount of "glue code" that connects everything together.

---

## Part 1: Setting the Stage for a New Backend

### Why the M88k?

The Motorola M88k is a RISC (Reduced Instruction Set Computer) architecture from the 1980s. Even though it is no longer manufactured, it was chosen for this chapter because:

1. **Sufficient documentation exists online.** The CPU manuals with the instruction set and timing information are available at bitsavers.org. The System V ABI M88k Processor supplement (which defines the ELF format and calling convention) is archived at archive.org.
2. **A real operating system still supports it.** OpenBSD supports the LUNA-88k system, meaning you can create a GCC cross-compiler for M88k on OpenBSD and have real object files to test against.
3. **An emulator exists.** GXemul can emulate certain OpenBSD releases for M88k, providing a way to test generated code.
4. **It is a clean RISC design.** Being a RISC architecture, the instructions have a regular, fixed-width format (all 32 bits), which simplifies the target description compared to something like x86 with its variable-length encoding.

The key takeaway here is that before starting any backend project, you need **three things**: documentation about the instruction set architecture (ISA), documentation about the binary object file format (usually ELF), and ideally, some existing toolchain or emulator to validate your work against.

---

## Part 2: Adding the New Architecture to the Triple Class

### What is the Triple Class?

In LLVM, a **target triple** is a string that identifies the target platform. It typically has the form `architecture-vendor-operating_system` (e.g., `x86_64-unknown-linux-gnu` or `m88k-unknown-openbsd`). The `Triple` class in LLVM parses and represents this string programmatically.

Every architecture that LLVM knows about must have an entry in the `Triple` class. This is literally the first step — you are telling LLVM, "Hey, this architecture exists."

### What You Need to Do

**Step 1: Modify the header file** (`llvm/include/llvm/TargetParser/Triple.h`):

You add a new member to the `ArchType` enumeration:

```cpp
class Triple {
public:
    enum ArchType {
        // Many more members
        m88k,       // M88000 (big endian): m88k
    };
```

This is the internal numeric identifier for the M88k architecture. Anywhere in LLVM that needs to check "are we targeting M88k?" will compare against `Triple::m88k`.

You also add a convenience predicate method:

```cpp
    /// Tests whether the target is M88k.
    bool isM88k() const {
        return getArch() == Triple::m88k;
    }
```

This is a helper so that code doesn't have to write `getArch() == Triple::m88k` everywhere — they can just call `isM88k()`.

**Step 2: Modify the implementation file** (`llvm/lib/TargetParser/Triple.cpp`):

There are many methods in this file that use `switch` statements on `ArchType`. You must add a `case m88k:` branch to every single one. For example, the `getArchTypeName()` method needs to know how to convert the enum value to a string:

```cpp
switch (Kind) {
    // Many more cases
    case m88k:      return "m88k";
}
```

The chapter notes a practical tip: **the compiler will usually warn you** if you forget to handle the new enumeration member in a switch statement, since exhaustive switch warnings are common in LLVM. This is your safety net.

### Why This Matters

Without this step, nothing else works. The Triple class is used everywhere in LLVM — by the command-line tools, by the code generators, by the linker integration. It is the canonical registry of "what architectures does LLVM know about."

---

## Part 3: Extending the ELF File Format Definition in LLVM

### What is ELF?

ELF (Executable and Linkable Format) is the standard binary file format used on Linux, BSD, and many other operating systems. When you compile code, the compiler produces an ELF object file (`.o`), and the linker combines multiple object files into an ELF executable or shared library.

ELF is architecture-generic, but each CPU architecture defines its own **relocations** (how symbolic references are patched at link time) and **flags** (special bits that indicate features of the object file).

### What Are Relocations?

When the assembler generates machine code, it often cannot know the final address of a function or global variable. Instead, it leaves a placeholder and creates a **relocation entry**. The linker later fills in the correct address. Each architecture defines its own set of relocation types (e.g., `R_88K_NONE`, `R_88K_COPY`, etc.) because different ISAs need different kinds of address patching (absolute addresses, PC-relative offsets, etc.).

### Step-by-Step Additions

**Step 1: Create the relocation definitions file** (`llvm/include/llvm/BinaryFormat/ELFRelocs/M88k.def`):

```cpp
#ifndef ELF_RELOC
#error "ELF_RELOC must be defined"
#endif
ELF_RELOC(R_88K_NONE, 0)
ELF_RELOC(R_88K_COPY, 1)
// Many more…
```

This uses the X-macro pattern: the macro `ELF_RELOC` is defined by the including file before including this `.def` file. The `.def` file just provides the data; the including file controls what to do with it (generate an enum, generate a string table, etc.). This is a very common pattern in LLVM for defining lists of related items.

**Step 2: Add M88k-specific ELF flags** (`llvm/include/llvm/BinaryFormat/ELF.h`):

```cpp
// M88k Specific e_flags
enum : unsigned {
    EF_88K_NABI = 0x80000000,    // Not ABI compliant
    EF_88K_M88110 = 0x00000004   // File uses 88110-specific features
};

// M88k relocations.
enum {
    #include "ELFRelocs/M88k.def"
};
```

The first enum defines architecture-specific flags for the ELF header. These flags tell the linker and other tools about properties of the object file. The second enum uses the X-macro technique to create an enumeration of all relocation types.

The chapter recommends inserting this code before the MIPS architecture's code in the file, to keep the file alphabetically/structurally organized.

**Step 3: Extend `ELFObjectFile.h` methods:**

The `getFileFormatName()` method needs to return a descriptive string:

```cpp
switch (EF.getHeader()->e_ident[ELF::EI_CLASS]) {
    // Many more cases
    case ELF::EM_88K:
        return "elf32-m88k";
}
```

And the `getArch()` method needs to map from the ELF machine constant to the Triple architecture:

```cpp
switch (EF.getHeader().e_machine) {
    // Many more cases
    case ELF::EM_88K:
        return Triple::m88k;
}
```

**Step 4: Wire up relocation name lookup** (`llvm/lib/Object/ELF.cpp`):

The `getELFRelocationTypeName()` method translates a relocation number to a human-readable name. It uses the X-macro pattern again:

```cpp
switch (Machine) {
    // Many more cases
    case ELF::EM_88K:
        switch (Type) {
            #include "llvm/BinaryFormat/ELFRelocs/M88k.def"
            default:
                break;
        }
        break;
}
```

**Step 5: Optionally extend the YAML tools** (`llvm/lib/ObjectYAML/ELFYAML.cpp`):

LLVM has `yaml2obj` and `obj2yaml` tools that convert between ELF files and human-readable YAML descriptions. You add a line to the machine enumeration:

```cpp
ECase(EM_88K);
```

And include the relocations in the relocation traits:

```cpp
case ELF::EM_88K:
    #include "llvm/BinaryFormat/ELFRelocs/M88k.def"
    break;
```

### Is This Mandatory?

The chapter includes an important sidebar: if your architecture uses ELF (which most do), this is easy and you should definitely do it. However, if your architecture needs a **completely new** binary file format (not ELF, not COFF, not Mach-O), that is a much harder task. In that case, a common strategy is to only emit assembler text files and rely on an external assembler to produce object files.

### What Can You Do After This Step?

After these additions, you can use `llvm-readobj` to inspect M88k ELF object files (e.g., files produced by a GCC cross-compiler on OpenBSD). You can also use `yaml2obj` to create M88k ELF object files from YAML descriptions. This is useful for testing.

---

## Part 4: Creating the Target Description

### The Heart of the Backend

The target description is the **single most important piece** of a backend. It is written in the **TableGen language** and defines:

- The **registers** (names, encodings, groupings)
- The **instruction formats** (bit layouts)
- The **instructions** themselves (mnemonics, operands, patterns for instruction selection)

LLVM's TableGen tool processes this description and generates C++ source code fragments (`.inc` files) that implement the assembler matcher, disassembler decoder, instruction printer, code emitter, and parts of instruction selection.

The base definitions are in `llvm/include/llvm/Target/Target.td`. This file defines the superclasses that every target description inherits from (like `Register`, `Instruction`, `RegisterClass`, etc.). It is heavily commented and serves as essential reference documentation.

The target description is split into several files for manageability. The top-level file for M88k is `M88k.td`, located in `llvm/lib/Target/M88k/`. It includes the other files.

---

### Section 4.1: Adding the Register Definition

**File: `M88kRegisterInfo.td`**

#### Background

Every CPU has a set of registers — small, fast storage locations inside the processor. The M88k architecture defines:
- **General-purpose registers (GPRs)**: 32 registers, named `r0` through `r31`, each 32 bits wide.
- **Extended registers**: Used for floating-point operations (not covered in detail to keep the example small).
- **Control registers**: For status/configuration (also not covered).

LLVM needs to know about every register: its name (for assembly text), its binary encoding (for machine code), its size, what types of values it can hold, how registers group together, and so on.

#### Defining Individual Registers

First, you create a superclass that all M88k registers inherit from:

```tablegen
class M88kReg<bits<5> Enc, string n> : Register<n> {
    let HWEncoding{15-5} = 0;
    let HWEncoding{4-0} = Enc;
    let Namespace = "M88k";
}
```

Let's unpack this:

- `M88kReg<bits<5> Enc, string n>` — This class takes two parameters: `Enc` is a 5-bit encoding value (since there are 32 registers, you need 5 bits: 2^5 = 32), and `n` is the register name string.
- `: Register<n>` — It inherits from the built-in `Register` class, passing the name.
- `let HWEncoding{15-5} = 0;` — The `HWEncoding` field is 16 bits wide. Bits 15 through 5 are set to 0 (unused).
- `let HWEncoding{4-0} = Enc;` — Bits 4 through 0 hold the actual encoding value.
- `let Namespace = "M88k";` — All generated C++ code goes into the `M88k` namespace.

#### Instantiating All 32 Registers

Rather than writing 32 separate definitions, TableGen's `foreach` loop is used:

```tablegen
foreach I = 0-31 in {
    let isConstant = !eq(I, 0) in
        def R#I : M88kReg<I, "r"#I>;
}
```

This generates records `R0`, `R1`, ..., `R31`. The `#` operator concatenates: `R#I` becomes `R0`, `R1`, etc., and `"r"#I` becomes `"r0"`, `"r1"`, etc.

**Special note about `r0`**: The M88k architecture has a **hardwired zero register** — reading `r0` always returns 0, regardless of what you write to it. This is common in RISC architectures (MIPS has `$zero`, RISC-V has `x0`). The `isConstant` flag is set to `true` when `I` equals 0, using the TableGen bang operator `!eq(I, 0)`. This tells LLVM that this register always holds a known constant value, which enables optimizations.

#### Register Classes

For the register allocator to work, individual registers must be grouped into **register classes**. A register class tells LLVM:

- What **value types** can be stored (e.g., `i32` for 32-bit integers, `f32` for 32-bit floats)
- The **spill size** (how many bits to save when the register must be stored to memory)
- The **alignment** requirement in memory
- The **allocation order** (which registers the allocator should prefer)

Rather than using the base `RegisterClass` directly, the chapter creates a wrapper class:

```tablegen
class M88kRegisterClass<list<ValueType> types, int size,
                        int alignment, dag regList,
                        int copycost = 1>
    : RegisterClass<"M88k", types, alignment, regList> {
    let Size = size;
    let CopyCost = copycost;
}
```

This avoids repeating the `"M88k"` namespace string and allows customization of the parameter list. Then the GPR register class is defined:

```tablegen
def GPR : M88kRegisterClass<[i32, f32], 32, 32,
                            (add (sequence "R%u", 0, 31))>;
```

This says: the GPR class holds values of type `i32` or `f32`, is 32 bits wide, requires 32-bit alignment, and includes registers `R0` through `R31`. The `sequence` generator produces a list of strings by substituting a counter into the template: `"R0"`, `"R1"`, ..., `"R31"`.

#### Register Operands

An operand describes how a register is used as input or output of an instruction. The chapter creates a custom register operand class:

```tablegen
class M88kRegisterOperand<RegisterClass RC>
    : RegisterOperand<RC> {
    let DecoderMethod = "decode"#RC#"RegisterClass";
}
```

The `DecoderMethod` field specifies the name of the C++ function that the disassembler will call to decode this operand from binary. By using string concatenation (`"decode"#RC#"RegisterClass"`), you automatically get a method name like `decodeGPRRegisterClass` — conforming to LLVM naming conventions.

The GPR operand is then defined:

```tablegen
def GPROpnd : M88kRegisterOperand<GPR> {
    let GIZeroRegister = R0;
}
```

The `GIZeroRegister` field tells the **Global Instruction Selection (GISel)** framework that `R0` is a zero register. This information helps GISel optimize code by recognizing that using `R0` as a source always yields zero.

#### Register Pairs (64-bit Operations)

The M88k uses pairs of adjacent GPRs for 64-bit floating-point operations (since individual registers are only 32 bits). To model this, you first define **sub-register indices**:

```tablegen
let Namespace = "M88k" in {
    def sub_hi : SubRegIndex<32, 0>;
    def sub_lo : SubRegIndex<32, 32>;
}
```

`sub_hi` represents the high 32 bits (starting at bit offset 0, 32 bits wide), and `sub_lo` represents the low 32 bits (starting at bit offset 32). These indices allow LLVM to understand the relationship between a 64-bit register pair and its constituent 32-bit registers.

The pairs are then created using `RegisterTuples`:

```tablegen
def GRPair : RegisterTuples<[sub_hi, sub_lo],
                [(add (sequence "R%u", 0, 30, 2)),
                 (add (sequence "R%u", 1, 31, 2))]>;
```

The fourth parameter of `sequence` is the **stride**. By using stride 2, the first list generates `R0, R2, R4, ..., R30` and the second generates `R1, R3, R5, ..., R31`. This creates even/odd pairs: `(R0, R1)`, `(R2, R3)`, ..., `(R30, R31)`.

The register class and operand for pairs:

```tablegen
def GPR64 : M88kRegisterClass<[i64, f64], 64, 32,
                              (add GRPair), /*copycost=*/ 2>;
def GPR64Opnd : M88kRegisterOperand<GPR64>;
```

The `copycost` is set to 2 because copying a register pair requires two instructions (one for each half), making it twice as expensive as a single-register copy. This helps the register allocator make better decisions.

---

### Section 4.2: Defining Instruction Formats and Instruction Information

**Files: `M88kInstrFormats.td` and `M88kInstrInfo.td`**

#### The Instruction Class Hierarchy

Defining instructions is the most complex part of the target description because an instruction has multiple facets:

1. **Textual representation** (for the assembler/disassembler): e.g., `and %r1, %r2, %r3`
2. **Binary encoding** (for the machine code): the specific bit pattern
3. **Semantics** (for instruction selection): what operation does this instruction perform?

The chapter manages this complexity through a class hierarchy.

#### The Base Class: `InstM88k`

```tablegen
class InstM88k<dag outs, dag ins, string asm, string operands,
               list<dag> pattern = []>
    : Instruction {
    bits<32> Inst;
    bits<32> SoftFail = 0;
    let Namespace = "M88k";
    let Size = 4;
    dag OutOperandList = outs;
    dag InOperandList = ins;
    let AsmString = !if(!eq(operands, ""), asm,
                        !strconcat(asm, " ", operands));
    let Pattern = pattern;
    let DecoderNamespace = "M88k";
}
```

Let's break down every field:

- **`bits<32> Inst`**: This field holds the complete binary encoding of the instruction. All M88k instructions are exactly 32 bits (4 bytes) wide — a hallmark of RISC design. The bits of this field will be set piece by piece in subclasses.

- **`bits<32> SoftFail = 0`**: This is a bitmask for "soft failure" during disassembly. If certain bits in the encoding can vary from the expected pattern and the instruction is still valid, those bits are set in `SoftFail`. Only the ARM architecture needs this, so for M88k it is simply 0.

- **`Namespace = "M88k"`**: Generated C++ code goes into this namespace.

- **`Size = 4`**: The instruction is 4 bytes long.

- **`OutOperandList` and `InOperandList`**: These are DAGs (Directed Acyclic Graphs) describing the output and input operands. They use the special `dag` type in TableGen.

- **`AsmString`**: The full assembly text template. This uses a conditional: if the `operands` string is empty (for instructions with no operands, like `nop`), the assembly string is just the mnemonic. Otherwise, it concatenates the mnemonic, a space, and the operands string. For example, `"and"` + `" "` + `"$rd, $rs1, $rs2"` = `"and $rd, $rs1, $rs2"`. The `$`-prefixed names are placeholders that get replaced with actual register names or immediate values.

- **`Pattern`**: A DAG pattern used for instruction selection. This tells LLVM "if the IR contains this pattern of operations, generate this instruction."

- **`DecoderNamespace = "M88k"`**: Groups the decoding tables. The disassembler uses this to organize its lookup tables.

#### Instruction Format Classes

The M88k CPU manual groups instructions by their encoding format. The chapter models this with intermediate classes. For logical operations:

```tablegen
class F_L<dag outs, dag ins, string asm, string operands,
          list<dag> pattern = []>
    : InstM88k<outs, ins, asm, operands, pattern> {
    bits<5> rd;
    bits<5> rs1;
    let Inst{25-21} = rd;
    let Inst{20-16} = rs1;
}
```

This class captures a pattern common to all logical operations: the destination register `rd` occupies bits 25 through 21, and the first source register `rs1` occupies bits 20 through 16.

The key implementation pattern is: **declare named fields, then assign them to bit positions in `Inst`**. The field `rd` is 5 bits wide. The line `let Inst{25-21} = rd;` says "bits 25 down to 21 of the instruction encoding come from the `rd` field." This is how LLVM knows where to put the register encoding in the binary output, and where to extract it during disassembly.

#### The Triadic Register Format

For three-register operations (like `and %r1, %r2, %r3`):

```tablegen
class F_LR<bits<5> func, bits<1> comp, string asm,
           list<dag> pattern = []>
    : F_L<(outs GPROpnd:$rd), (ins GPROpnd:$rs1, GPROpnd:$rs2),
          !if(comp, !strconcat(asm, ".c"), asm),
          "$rd, $rs1, $rs2", pattern> {
    bits<5> rs2;
    let Inst{31-26} = 0b111101;
    let Inst{15-11} = func;
    let Inst{10}    = comp;
    let Inst{9-5}   = 0b00000;
    let Inst{4-0}   = rs2;
}
```

This is a rich class. Let's dissect every aspect:

**Parameters:**
- `func` (5 bits): The function code that distinguishes different operations within this format (e.g., AND vs OR vs XOR).
- `comp` (1 bit): A flag indicating whether the second source operand should be bitwise-complemented before the operation.
- `asm`: The mnemonic string (e.g., `"and"`).
- `pattern`: Optional instruction selection pattern.

**Superclass initialization (`F_L<...>`):**

The output operands are `(outs GPROpnd:$rd)` — meaning this instruction produces one output, which is a general-purpose register referred to as `$rd`. The `outs` keyword marks this as the output list.

The input operands are `(ins GPROpnd:$rs1, GPROpnd:$rs2)` — two GPR inputs.

The mnemonic is computed conditionally: if `comp` is 1, the mnemonic gets a `.c` suffix (e.g., `and.c`), indicating the complement variant. Otherwise, it's just the plain mnemonic.

The operand string is fixed: `"$rd, $rs1, $rs2"`.

**Understanding DAGs in TableGen:**

A `dag` in TableGen is a tree-like structure with an operator and arguments. For example, `(outs GPROpnd:$rd)` has `outs` as the operator and `GPROpnd:$rd` as the single argument. The argument `GPROpnd:$rd` has two parts:
- `GPROpnd` is the **type** (the register operand class we defined earlier)
- `$rd` is the **name**, used to link the operand to the assembly string, the bit encoding, and the instruction selection pattern.

**Bit encoding:**
- Bits 31-26: Fixed to `0b111101` (the opcode for this instruction group).
- Bits 25-21: The `rd` field (inherited from `F_L`).
- Bits 20-16: The `rs1` field (inherited from `F_L`).
- Bits 15-11: The `func` code.
- Bit 10: The `comp` flag.
- Bits 9-5: Fixed to `0b00000` (reserved/unused).
- Bits 4-0: The `rs2` field (second source register).

**Important check**: All 32 bits are now accounted for. This is crucial — if any bits are unassigned, the encoding will be incorrect and the assembler/disassembler will malfunction.

#### Defining Instructions with Multiclass

Since each logical operation has two variants (with and without complement), a `multiclass` is used to define both at once:

```tablegen
multiclass Logic<bits<5> Fun, string OpcStr, SDNode OpNode> {
    let isCommutable = 1 in
    def rr : F_LR<Fun, /*comp=*/0b0, OpcStr,
                   [(set i32:$rd,
                         (OpNode GPROpnd:$rs1, GPROpnd:$rs2))]>;
    def rrc : F_LR<Fun, /*comp=*/0b1, OpcStr,
                    [(set i32:$rd,
                          (OpNode GPROpnd:$rs1, (not GPROpnd:$rs2)))]>;
}
```

**Key concepts:**

- **`isCommutable = 1`**: Tells LLVM that the operation is commutative (e.g., `AND(a, b) == AND(b, a)`). This allows the instruction selector to freely swap operands if that produces a better match.

- **Instruction selection patterns**: The pattern `[(set i32:$rd, (OpNode GPROpnd:$rs1, GPROpnd:$rs2))]` reads: "If the IR contains an `OpNode` operation on two 32-bit register values, assign the result to `$rd` and generate this instruction." For the complement variant, the pattern wraps the second operand in a `not` operation: `(not GPROpnd:$rs2)`.

- **Naming convention**: `rr` means "register-register" and `rrc` means "register-register-complement." When the multiclass is instantiated, these become suffix parts of the full name.

#### Instantiating the Instructions

```tablegen
defm AND : Logic<0b01000, "and", and>;
defm XOR : Logic<0b01010, "xor", xor>;
defm OR  : Logic<0b01011, "or", or>;
```

`defm` instantiates a multiclass. For `AND`, this creates two records:
- `ANDrr` — The plain `and` instruction
- `ANDrrc` — The `and.c` complement variant

The three parameters are:
1. `0b01000` — The function code in the encoding
2. `"and"` — The assembler mnemonic
3. `and` — The LLVM IR operation node (defined in `TargetSelectionDAG.td`)

#### Design Principle: No Repetition

The guiding philosophy is **DRY (Don't Repeat Yourself)**. The function code `0b01000` appears exactly once. Without the multiclass, you would have to type it twice (once for each variant) and repeat the patterns, which is error-prone. The class hierarchy ensures that each piece of information is specified in exactly one place.

#### Naming Conventions

The chapter emphasizes establishing a consistent naming scheme. `ANDrr` tells you it is an AND instruction with two register operands. `ANDrrc` adds the complement suffix. These names end up in the generated C++ code and in debug output, so good names make debugging much easier.

---

### Section 4.3: Creating the Top-Level File for the Target Description

**File: `M88k.td`**

This is the master file that includes everything and defines global instances:

```tablegen
include "llvm/Target/Target.td"

include "M88kRegisterInfo.td"
include "M88kInstrFormats.td"
include "M88kInstrInfo.td"
```

The order matters: LLVM's base definitions come first, then registers (which instructions depend on), then instruction formats, then instructions.

**Global instances:**

```tablegen
def M88kInstrInfo : InstrInfo;
```

This creates a record that references all defined instructions.

```tablegen
def M88kAsmParser : AsmParser;
def M88kAsmParserVariant : AsmParserVariant {
    let RegisterPrefix = "%";
}
```

This tells LLVM that register names in M88k assembly are prefixed with `%` (e.g., `%r1`). This is essential for the assembler parser to distinguish register references from other identifiers.

```tablegen
def M88k : Target {
    let InstructionSet = M88kInstrInfo;
    let AssemblyParsers = [M88kAsmParser];
    let AssemblyParserVariants = [M88kAsmParserVariant];
}
```

This is the top-level target definition that ties everything together.

---

## Part 5: Adding the M88k Backend to LLVM

### Directory Structure

Each LLVM backend has its own directory under `llvm/lib/Target/`. You create `llvm/lib/Target/M88k/` and place all your files there. The directory structure typically has subdirectories:

- Root: Target description files (`.td`), `CMakeLists.txt`, top-level `.cpp` files
- `TargetInfo/`: Target registration
- `MCTargetDesc/`: Machine-code level descriptions
- `AsmParser/`: Assembler parser
- `Disassembler/`: Disassembler

### Target Registration

LLVM uses a **registry pattern** — backends register themselves through global functions that are called during initialization. Each target holds a static instance of the `Target` class.

#### TargetInfo (`TargetInfo/M88kTargetInfo.cpp`)

```cpp
using namespace llvm;
Target &llvm::getTheM88kTarget() {
    static Target TheM88kTarget;
    return TheM88kTarget;
}
```

This is a **Meyer's Singleton**: the `Target` object is created once, the first time this function is called, and is returned by reference thereafter.

The registration function has **C linkage** because it is also used by the LLVM C API:

```cpp
extern "C" LLVM_EXTERNAL_VISIBILITY void
LLVMInitializeM88kTargetInfo() {
    RegisterTarget<Triple::m88k, /*HasJIT=*/false> X(
        getTheM88kTarget(), "m88k", "M88k", "M88k");
}
```

`RegisterTarget` registers the target instance in LLVM's global registry. `HasJIT=false` means this backend does not support Just-In-Time compilation.

The header file (`M88kTargetInfo.h`) simply declares `getTheM88kTarget()` for other files to use.

#### MCTargetDesc (`MCTargetDesc/M88kMCTargetDesc.cpp`)

This file populates the target instance with **machine-code-level** information. It includes TableGen-generated code fragments:

```cpp
#define GET_INSTRINFO_MC_DESC
#include "M88kGenInstrInfo.inc"

#define GET_SUBTARGETINFO_MC_DESC
#include "M88kGenSubtargetInfo.inc"

#define GET_REGINFO_MC_DESC
#include "M88kGenRegisterInfo.inc"
```

Each `#define` selects a specific section from the generated `.inc` file. This is how LLVM's code generation works: TableGen produces a single large `.inc` file with many sections guarded by `#define` macros, and each consuming `.cpp` file selects the section it needs.

**Factory methods** are defined for each type of information:

For instruction info:
```cpp
static MCInstrInfo *createM88kMCInstrInfo() {
    MCInstrInfo *X = new MCInstrInfo();
    InitM88kMCInstrInfo(X);  // Generated function
    return X;
}
```

For register info:
```cpp
static MCRegisterInfo *
createM88kMCRegisterInfo(const Triple &TT) {
    MCRegisterInfo *X = new MCRegisterInfo();
    InitM88kMCRegisterInfo(X, M88k::R1);  // R1 holds return address
    return X;
}
```

The `M88k::R1` parameter tells LLVM that register `r1` holds the return address. This is an ABI convention of the M88k architecture — when a function is called, the return address is placed in `r1`.

For subtarget info:
```cpp
static MCSubtargetInfo *
createM88kMCSubtargetInfo(const Triple &TT,
                          StringRef CPU, StringRef FS) {
    return createM88kMCSubtargetInfoImpl(TT, CPU,
                                          /*TuneCPU*/ CPU, FS);
}
```

**Registration:**

```cpp
extern "C" LLVM_EXTERNAL_VISIBILITY void
LLVMInitializeM88kTargetMC() {
    TargetRegistry::RegisterMCInstrInfo(
        getTheM88kTarget(), createM88kMCInstrInfo);
    TargetRegistry::RegisterMCRegInfo(
        getTheM88kTarget(), createM88kMCRegisterInfo);
    TargetRegistry::RegisterMCSubtargetInfo(
        getTheM88kTarget(), createM88kMCSubtargetInfo);
}
```

Each `RegisterXxx` call associates a factory method with the target. When LLVM tools need an `MCInstrInfo` object for the M88k target, they call the registered factory.

The header file (`M88kMCTargetDesc.h`) makes the generated enumerations available:

```cpp
#define GET_REGINFO_ENUM
#include "M88kGenRegisterInfo.inc"

#define GET_INSTRINFO_ENUM
#include "M88kGenInstrInfo.inc"

#define GET_SUBTARGETINFO_ENUM
#include "M88kGenSubtargetInfo.inc"
```

This gives other files access to names like `M88k::R0`, `M88k::ANDrr`, etc.

#### The TargetMachine Stub

To prevent a linker error, a stub function is needed:

```cpp
extern "C" LLVM_EXTERNAL_VISIBILITY void
LLVMInitializeM88kTarget() {
    // TODO Register the target machine. See chapter 12.
}
```

The full `TargetMachine` implementation is deferred to the next chapter on instruction selection. For now, an empty function satisfies the linker.

### Integrating into the Build System

You modify `llvm/CMakeLists.txt` to add M88k to the list of experimental targets:

```cmake
set(LLVM_ALL_EXPERIMENTAL_TARGETS ARC … M88k …)
```

Then configure the build:

```bash
$ cmake -DLLVM_EXPERIMENTAL_TARGETS_TO_BUILD=M88k ../llvm-m88k/llvm
```

The output should show `-- Targeting M88k`.

After building, you can verify:

```bash
$ bin/llc --version
Registered Targets:
    m88k    - M88k
```

### A Note on a Compile Error

The chapter mentions a specific bug in LLVM 17.0.2: the TableGen emitter for subtarget information uses the removed `llvm::None` instead of `std::nullopt`. The fix is to cherry-pick commit `a587f429` from the LLVM 18 development branch.

---

## Part 6: Implementing the Assembler Parser

The assembler parser takes assembly text (e.g., `and %r1, %r2, %r3`) and converts it to LLVM's internal `MCInst` representation, and from there to binary machine code. LLVM provides a framework for this, and TableGen generates much of the matcher logic.

### How the Assembler Framework Works

The process has two phases:

1. **Parsing**: The `ParseInstruction()` method reads tokens from the input and builds an **operand vector** — a list of parsed operands (tokens, registers, immediates).

2. **Matching**: A generated matcher compares the operand vector against all instruction definitions. If a match is found, an `MCInst` object is created. If not, an error is emitted.

For example, `and %r1, %r2, %r3` produces four operands: a token "and", a register `%r1`, a register `%r2`, and a register `%r3`. The matcher recognizes this as the `ANDrr` instruction.

### Support Classes Required

Several support classes are needed before the assembler parser itself can be built.

#### MCAsmInfo (`MCTargetDesc/M88kMCAsmInfo.h` and `.cpp`)

This class configures the assembler for the target. It inherits from `MCAsmInfoELF` because M88k uses the ELF format (and therefore shares characteristics with other ELF-based systems).

```cpp
class M88kMCAsmInfo : public MCAsmInfoELF {
public:
    explicit M88kMCAsmInfo(const Triple &TT);
};
```

The constructor sets platform-specific defaults:

```cpp
M88kMCAsmInfo::M88kMCAsmInfo(const Triple &TT) {
    IsLittleEndian = false;            // M88k is big-endian
    UseDotAlignForAlignment = true;    // Use .align directive
    MinInstAlignment = 4;              // Instructions are 4-byte aligned
    CommentString = "|";               // | starts a comment
    ZeroDirective = "\t.space\t";      // How to emit zero-filled space
    Data64bitsDirective = "\t.quad\t"; // How to emit 64-bit data
    UsesELFSectionDirectiveForBSS = true;
    SupportsDebugInformation = false;  // No debug info support yet
    ExceptionsType = ExceptionHandling::SjLj;  // setjmp/longjmp exceptions
}
```

The comment string is interesting: M88k assembly uses `|` for comments (not `#` or `;` as in many other architectures). The `#` symbol is only allowed as a comment delimiter at the first column, which is a quirk of the M88k assembler.

#### MCCodeEmitter (`MCTargetDesc/M88kMCCodeEmitter.cpp`)

This class converts an `MCInst` (LLVM's internal representation of an instruction) into binary bytes. It is one of the two output paths from `MCInst` — the other being the `InstPrinter` which produces text.

The class structure:

```cpp
class M88kMCCodeEmitter : public MCCodeEmitter {
    const MCInstrInfo &MCII;
    MCContext &Ctx;

public:
    void encodeInstruction(const MCInst &MI, raw_ostream &OS,
                           SmallVectorImpl<MCFixup> &Fixups,
                           const MCSubtargetInfo &STI) const override;

    uint64_t getBinaryCodeForInstr(const MCInst &MI,
                                    SmallVectorImpl<MCFixup> &Fixups,
                                    const MCSubtargetInfo &STI) const;

    unsigned getMachineOpValue(const MCInst &MI, const MCOperand &MO,
                               SmallVectorImpl<MCFixup> &Fixups,
                               const MCSubtargetInfo &STI) const;
};
```

**`getBinaryCodeForInstr()`** is generated by TableGen — it uses the bit assignments from the target description to assemble the instruction encoding.

**`encodeInstruction()`** calls the generated method and writes the result:

```cpp
void M88kMCCodeEmitter::encodeInstruction(
        const MCInst &MI, raw_ostream &OS,
        SmallVectorImpl<MCFixup> &Fixups,
        const MCSubtargetInfo &STI) const {
    uint64_t Bits = getBinaryCodeForInstr(MI, Fixups, STI);
    ++MCNumEmitted;
    support::endian::write<uint32_t>(OS, Bits, support::big);
}
```

Since M88k is big-endian, the 4-byte instruction is written in big-endian byte order using LLVM's `support::endian::write` utility.

A `STATISTIC` counter `MCNumEmitted` tracks how many instructions are emitted — this is LLVM's built-in statistics infrastructure, accessible with `--stats` on the command line.

**`getMachineOpValue()`** is called by the generated code to get the binary value for each operand:

```cpp
unsigned M88kMCCodeEmitter::getMachineOpValue(
        const MCInst &MI, const MCOperand &MO,
        SmallVectorImpl<MCFixup> &Fixups,
        const MCSubtargetInfo &STI) const {
    if (MO.isReg())
        return Ctx.getRegisterInfo()->getEncodingValue(MO.getReg());
    if (MO.isImm())
        return static_cast<uint64_t>(MO.getImm());
    return 0;
}
```

For a register operand, it looks up the hardware encoding value (the 5-bit number we defined in the target description). For an immediate, it returns the value directly.

#### InstPrinter (`MCTargetDesc/M88kInstPrinter.h` and `.cpp`)

This is the inverse of the code emitter: it takes an `MCInst` and produces assembly text.

The key method is `printOperand()`:

```cpp
void M88kInstPrinter::printOperand(
        const MCInst *MI, int OpNum,
        const MCSubtargetInfo &STI, raw_ostream &O) {
    const MCOperand &MO = MI->getOperand(OpNum);
    if (MO.isReg()) {
        if (!MO.getReg())
            O << '0';
        else
            O << '%' << getRegisterName(MO.getReg());
    } else if (MO.isImm())
        O << MO.getImm();
    else
        llvm_unreachable("Invalid operand");
}
```

For registers, it prints `%` followed by the register name (obtained from the generated `getRegisterName()` method). For register 0 (no register), it prints `0`. For immediates, it prints the numeric value.

The `printInst()` method is trivially simple:

```cpp
void M88kInstPrinter::printInst(
        const MCInst *MI, uint64_t Address, StringRef Annot,
        const MCSubtargetInfo &STI, raw_ostream &O) {
    printInstruction(MI, Address, STI, O);  // Generated
    printAnnotation(O, Annot);
}
```

`printInstruction()` is generated by TableGen from the `AsmString` fields in the target description. `printAnnotation()` handles optional annotations (like comments).

#### Registering the Support Classes

All these new classes need factory methods registered in `LLVMInitializeM88kTargetMC()`:

```cpp
extern "C" LLVM_EXTERNAL_VISIBILITY void
LLVMInitializeM88kTargetMC() {
    // ... previous registrations ...
    TargetRegistry::RegisterMCAsmInfo(
        getTheM88kTarget(), createM88kMCAsmInfo);
    TargetRegistry::RegisterMCCodeEmitter(
        getTheM88kTarget(), createM88kMCCodeEmitter);
    TargetRegistry::RegisterMCInstPrinter(
        getTheM88kTarget(), createM88kMCInstPrinter);
}
```

### The Assembler Parser Itself

**File: `AsmParser/M88kAsmParser.cpp`**

This file contains two classes in an anonymous namespace:

#### The `M88kOperand` Class

This represents a single parsed operand. It can be one of three kinds:

```cpp
enum OperandKind { OpKind_Token, OpKind_Reg, OpKind_Imm };
```

- **Token**: An instruction mnemonic like `and`
- **Register**: A register reference like `%r1`
- **Immediate**: A numeric value or expression

The value is stored in a union (since an operand is exactly one of these):

```cpp
union {
    StringRef Token;
    unsigned RegNo;
    const MCExpr *Imm;
};
```

Using `MCExpr` for immediates (rather than just a plain integer) is a forward-looking design decision. `MCExpr` can represent not just constants but also symbolic expressions (like label references), which are needed when the assembler handles address operands.

For each operand kind, four methods are needed:

1. **`isXxx()`**: Type check (e.g., `isReg()`)
2. **`getXxx()`**: Value accessor (e.g., `getReg()`)
3. **`createXxx()`**: Static factory method
4. **`addXxxOperands()`**: Called by generated code to add the operand to an `MCInst`

For example, for registers:

```cpp
bool isReg() const override { return Kind == OpKind_Reg; }
unsigned getReg() const override { return RegNo; }

static std::unique_ptr<M88kOperand>
createReg(unsigned Num, SMLoc StartLoc, SMLoc EndLoc) {
    auto Op = std::make_unique<M88kOperand>(
        OpKind_Reg, StartLoc, EndLoc);
    Op->RegNo = Num;
    return Op;
}

void addRegOperands(MCInst &Inst, unsigned N) const {
    assert(N == 1 && "Invalid number of operands");
    Inst.addOperand(MCOperand::createReg(getReg()));
}
```

The `SMLoc` (Source Manager Location) values track where in the source text this operand appears, which is essential for generating meaningful error messages.

#### The `M88kAsmParser` Class

This is the main assembler parser. It includes a generated header:

```cpp
class M88kAsmParser : public MCTargetAsmParser {
    #define GET_ASSEMBLER_HEADER
    #include "M88kGenAsmMatcher.inc"
    // ...
};
```

And later includes the generated implementation:

```cpp
#define GET_REGISTER_MATCHER
#define GET_MATCHER_IMPLEMENTATION
#include "M88kGenAsmMatcher.inc"
```

**`ParseInstruction()`** is the entry point when the framework expects an instruction:

```cpp
bool M88kAsmParser::ParseInstruction(
        ParseInstructionInfo &Info, StringRef Name,
        SMLoc NameLoc, OperandVector &Operands) {
    // Add mnemonic as a token operand
    Operands.push_back(M88kOperand::createToken(Name, NameLoc));

    // Parse operands if not at end of statement
    if (getLexer().isNot(AsmToken::EndOfStatement)) {
        if (parseOperand(Operands, Name)) {
            return Error(getLexer().getLoc(), "expected operand");
        }
        while (getLexer().is(AsmToken::Comma)) {
            Parser.Lex();  // Consume the comma
            if (parseOperand(Operands, Name)) {
                return Error(getLexer().getLoc(), "expected operand");
            }
        }
        if (getLexer().isNot(AsmToken::EndOfStatement))
            return Error(getLexer().getLoc(),
                         "unexpected token in argument list");
    }
    Parser.Lex();  // Consume end of statement
    return false;
}
```

**Important convention**: In LLVM's assembler parser, `return true` means **error** and `return false` means **success**. This is the opposite of what you might expect!

The parsing logic is straightforward:
1. Add the mnemonic as a token operand.
2. If there are more tokens, parse the first operand.
3. While there are commas, consume each comma and parse the next operand.
4. Verify we've reached the end of the statement.

**`parseOperand()`** distinguishes between register and immediate operands:

```cpp
bool M88kAsmParser::parseOperand(
        OperandVector &Operands, StringRef Mnemonic) {
    if (Parser.getTok().is(AsmToken::Percent)) {
        MCRegister RegNo;
        SMLoc StartLoc, EndLoc;
        if (parseRegister(RegNo, StartLoc, EndLoc,
                          /*RestoreOnFailure=*/false))
            return true;
        Operands.push_back(
            M88kOperand::createReg(RegNo, StartLoc, EndLoc));
        return false;
    }
    if (Parser.getTok().is(AsmToken::Integer)) {
        SMLoc StartLoc = Parser.getTok().getLoc();
        const MCExpr *Expr;
        if (Parser.parseExpression(Expr))
            return true;
        SMLoc EndLoc = Parser.getTok().getLoc();
        Operands.push_back(
            M88kOperand::createImm(Expr, StartLoc, EndLoc));
        return false;
    }
    return true;  // Error: unrecognized operand
}
```

If the token starts with `%`, it is a register. If it is an integer, it is an immediate (parsed as a general expression for future extensibility).

**`parseRegister()`** handles the `%identifier` pattern:

```cpp
bool M88kAsmParser::parseRegister(
        MCRegister &RegNo, SMLoc &StartLoc, SMLoc &EndLoc,
        bool RestoreOnFailure) {
    StartLoc = Parser.getTok().getLoc();
    if (Parser.getTok().isNot(AsmToken::Percent))
        return true;
    const AsmToken &PercentTok = Parser.getTok();
    Parser.Lex();  // Consume %
    if (Parser.getTok().isNot(AsmToken::Identifier) ||
        (RegNo = MatchRegisterName(
            Parser.getTok().getIdentifier())) == 0) {
        if (RestoreOnFailure)
            Parser.getLexer().UnLex(PercentTok);
        return Error(StartLoc, "invalid register");
    }
    Parser.Lex();  // Consume register name
    EndLoc = Parser.getTok().getLoc();
    return false;
}
```

The `MatchRegisterName()` function is **generated by TableGen** — it maps strings like `"r1"` to register enum values like `M88k::R1`. The `RestoreOnFailure` parameter controls whether to "un-lex" (put back) the `%` token if the register parse fails — this is important for the `tryParseRegister()` variant where we're not sure if it is a register.

**`MatchAndEmitInstruction()`** drives the matching:

```cpp
bool M88kAsmParser::MatchAndEmitInstruction(
        SMLoc IdLoc, unsigned &Opcode,
        OperandVector &Operands, MCStreamer &Out,
        uint64_t &ErrorInfo, bool MatchingInlineAsm) {
    MCInst Inst;
    SMLoc ErrorLoc;
    switch (MatchInstructionImpl(
                Operands, Inst, ErrorInfo, MatchingInlineAsm)) {
    case Match_Success:
        Out.emitInstruction(Inst, SubtargetInfo);
        Opcode = Inst.getOpcode();
        return false;
    case Match_MissingFeature:
        return Error(IdLoc, "Instruction use requires "
                            "option to be enabled");
    case Match_MnemonicFail:
        return Error(IdLoc, "Unrecognized instruction mnemonic");
    case Match_InvalidOperand: {
        ErrorLoc = IdLoc;
        if (ErrorInfo != ~0U) {
            if (ErrorInfo >= Operands.size())
                return Error(IdLoc, "Too few operands for instruction");
            ErrorLoc = ((M88kOperand &)*Operands[ErrorInfo])
                        .getStartLoc();
            if (ErrorLoc == SMLoc())
                ErrorLoc = IdLoc;
        }
        return Error(ErrorLoc, "Invalid operand for instruction");
    }
    default:
        break;
    }
    llvm_unreachable("Unknown match type detected!");
}
```

`MatchInstructionImpl()` is the **generated matcher**. It compares the operand vector against all instruction definitions and returns a match result. On success, it fills in the `MCInst` object. On failure, it provides error information (which operand caused the mismatch).

The `ErrorInfo` value contains the index of the operand that failed to match, which allows generating a precise error message pointing to the right location in the source.

#### Registration

```cpp
extern "C" LLVM_EXTERNAL_VISIBILITY void
LLVMInitializeM88kAsmParser() {
    RegisterMCAsmParser<M88kAsmParser> X(getTheM88kTarget());
}
```

### Testing the Assembler

After building, you can test with the `llvm-mc` tool:

```bash
$ echo 'and %r1,%r2,%r3' | \
    bin/llvm-mc --triple m88k-openbsd --show-encoding
    .text
    and %r1, %r2, %r3  | encoding: [0xf4,0x22,0x40,0x03]
```

Notice that the comment uses `|` — this is the comment character we configured in `M88kMCAsmInfo`.

**Debugging tip**: Use `--debug-only=asm-matcher` to see why a parsed instruction fails to match. This dumps the matching process to stderr, showing which alternatives were tried and why they were rejected.

---

## Part 7: Creating the Disassembler

The disassembler does the inverse of the assembler: it takes binary machine code and produces assembly text. While optional, implementing it is recommended because:

1. It is relatively low effort.
2. Generating the disassembler tables can **catch encoding errors** in the target description that other generators don't detect.

**File: `Disassembler/M88kDisassembler.cpp`**

### The Disassembler Class

```cpp
class M88kDisassembler : public MCDisassembler {
public:
    M88kDisassembler(const MCSubtargetInfo &STI, MCContext &Ctx)
        : MCDisassembler(STI, Ctx) {}

    DecodeStatus
    getInstruction(MCInst &instr, uint64_t &Size,
                   ArrayRef<uint8_t> Bytes,
                   uint64_t Address,
                   raw_ostream &CStream) const override;
};
```

The only method to implement is `getInstruction()`.

### The Decoder Function

Before the generated tables are included, you need to provide a function that the generated code calls to decode register operands:

```cpp
static const uint16_t GPRDecoderTable[] = {
    M88k::R0, M88k::R1, M88k::R2, M88k::R3,
    // … all 32 registers
};

static DecodeStatus
decodeGPRRegisterClass(MCInst &Inst, uint64_t RegNo,
                       uint64_t Address,
                       const void *Decoder) {
    if (RegNo > 31)
        return MCDisassembler::Fail;
    unsigned Register = GPRDecoderTable[RegNo];
    Inst.addOperand(MCOperand::createReg(Register));
    return MCDisassembler::Success;
}
```

This is the **inverse** of `getMachineOpValue()` in the code emitter. Given a 5-bit register number from the instruction encoding, it looks up the corresponding LLVM register enum value and adds it as an operand to the `MCInst`.

The function name `decodeGPRRegisterClass` matches the `DecoderMethod` field we set in the `M88kRegisterOperand` class in the target description. This is the connection point between the TableGen description and the C++ implementation.

### Including Generated Tables

```cpp
#include "M88kGenDisassemblerTables.inc"
```

This pulls in the generated decoder tables that map bit patterns to instructions.

### The `getInstruction()` Method

```cpp
DecodeStatus M88kDisassembler::getInstruction(
        MCInst &MI, uint64_t &Size, ArrayRef<uint8_t> Bytes,
        uint64_t Address, raw_ostream &CS) const {
    if (Bytes.size() < 4) {
        Size = 0;
        return MCDisassembler::Fail;
    }
    Size = 4;
    uint32_t Inst = 0;
    for (uint32_t I = 0; I < Size; ++I)
        Inst = (Inst << 8) | Bytes[I];
    if (decodeInstruction(DecoderTableM88k32, MI, Inst,
                          Address, this, STI) !=
        MCDisassembler::Success) {
        return MCDisassembler::Fail;
    }
    return MCDisassembler::Success;
}
```

The logic:

1. Check if we have at least 4 bytes available. If not, fail.
2. Set `Size = 4` (all M88k instructions are 4 bytes).
3. Read 4 bytes in big-endian order into a 32-bit integer. The loop shifts each byte into position: `(Inst << 8) | Bytes[I]`.
4. Call the generated `decodeInstruction()` function with the table name `DecoderTableM88k32` (generated from the `DecoderNamespace` field we set to `"M88k"` and the instruction size of 32 bits).

### Registration

```cpp
extern "C" LLVM_EXTERNAL_VISIBILITY void
LLVMInitializeM88kDisassembler() {
    TargetRegistry::RegisterMCDisassembler(
        getTheM88kTarget(), createM88kDisassembler);
}
```

### Testing the Disassembler

```bash
$ echo "0xf4,0x22,0x40,0x03" | \
    bin/llvm-mc --triple m88k-openbsd --disassemble
    .text
    and %r1, %r2, %r3
```

The bytes `0xf4,0x22,0x40,0x03` are correctly decoded back to `and %r1, %r2, %r3`, confirming the round-trip: assembly → binary → assembly.

You can also now use `llvm-objdump` to disassemble M88k ELF object files, though for full usefulness you would need to add all instructions (not just the logical operations shown in this chapter).

---

## Final Synthesis: How Everything Connects

Let's trace the entire data flow to see how all the pieces fit together:

### Assembly → Binary (Assembler Path)

1. **Input**: `and %r1, %r2, %r3`
2. **Lexer** (LLVM framework): Tokenizes into `[Identifier:"and", Percent, Identifier:"r1", Comma, Percent, Identifier:"r2", Comma, Percent, Identifier:"r3"]`
3. **`ParseInstruction()`** (our code): Builds operand vector: `[Token:"and", Reg:R1, Reg:R2, Reg:R3]`
4. **`MatchInstructionImpl()`** (generated): Matches against `ANDrr` definition, creates `MCInst{opcode=ANDrr, operands=[R1, R2, R3]}`
5. **`encodeInstruction()`** (our code + generated): Calls `getBinaryCodeForInstr()` which:
   - Looks up that `ANDrr` has `Inst{31-26} = 0b111101`, `func = 0b01000`, `comp = 0`, etc.
   - Calls `getMachineOpValue()` to get register encodings: R1=1, R2=2, R3=3
   - Assembles the 32-bit value
6. **Output**: `[0xf4, 0x22, 0x40, 0x03]`

### Binary → Assembly (Disassembler Path)

1. **Input**: `[0xf4, 0x22, 0x40, 0x03]`
2. **`getInstruction()`** (our code): Reads 4 bytes into `uint32_t`
3. **`decodeInstruction()`** (generated): Looks up the bit pattern in the decoder table, identifies `ANDrr`, calls `decodeGPRRegisterClass()` for each register field
4. **Result**: `MCInst{opcode=ANDrr, operands=[R1, R2, R3]}`
5. **`printInst()`** (our code + generated): Calls `printInstruction()` which formats the assembly string
6. **Output**: `and %r1, %r2, %r3`

### The Class Hierarchy Summary

The target description (`.td` files) form a hierarchy:

```
Target.td (LLVM base definitions)
  └── M88k.td (top-level, includes all others)
       ├── M88kRegisterInfo.td
       │     M88kReg → Register (per-register definitions)
       │     M88kRegisterClass → RegisterClass (grouping)
       │     M88kRegisterOperand → RegisterOperand (operand info)
       │
       ├── M88kInstrFormats.td
       │     InstM88k → Instruction (base instruction)
       │     F_L → InstM88k (logical format)
       │     F_LR → F_L (triadic register format)
       │
       └── M88kInstrInfo.td
              Logic (multiclass, instantiates F_LR twice)
              AND, XOR, OR (final instruction records)
```

### The C++ Files Summary

```
llvm/lib/Target/M88k/
├── M88k.td, M88kRegisterInfo.td, M88kInstrFormats.td, M88kInstrInfo.td
├── M88kTargetMachine.cpp          (stub for now)
├── TargetInfo/
│   ├── M88kTargetInfo.h/cpp       (target singleton + registration)
│   └── CMakeLists.txt
├── MCTargetDesc/
│   ├── M88kMCTargetDesc.h/cpp     (MC-level factory registration)
│   ├── M88kMCAsmInfo.h/cpp        (assembler configuration)
│   ├── M88kMCCodeEmitter.cpp      (MCInst → binary)
│   ├── M88kInstPrinter.h/cpp      (MCInst → text)
│   └── CMakeLists.txt
├── AsmParser/
│   ├── M88kAsmParser.cpp          (text → MCInst)
│   └── CMakeLists.txt
├── Disassembler/
│   ├── M88kDisassembler.cpp       (binary → MCInst)
│   └── CMakeLists.txt
└── CMakeLists.txt
```

### Key Takeaways

1. **The target description is central.** Most C++ code is either generated from it or is thin glue that connects generated pieces. Getting the `.td` files right is the most important part of backend development.

2. **LLVM uses a registry/factory pattern extensively.** Every component is registered via a factory method. This allows lazy initialization and modular compilation — backends can be included or excluded at build time.

3. **TableGen's X-macro pattern** (`#define GET_XXX` / `#include "M88kGenXxx.inc"`) is used throughout to extract specific code fragments from generated files.

4. **The design philosophy is DRY.** Information appears in exactly one place. The register encoding is defined once in the `.td` file and used automatically by both the assembler (for encoding) and the disassembler (for decoding).

5. **Incremental development is key.** The chapter starts with the simplest possible step (extending the Triple class) and builds up incrementally to a working assembler and disassembler. The TargetMachine (needed for full code generation) is deferred to the next chapter.

6. **Everything is wired through naming conventions.** The `DecoderMethod` field in the target description specifies the name of a C++ function that you must implement. The `LLVMInitialize<Name>XxxYyy()` functions follow a naming convention that LLVM's initialization macros rely on. Breaking these conventions breaks the build.

This chapter establishes the **foundation** for a complete backend. The next chapters (12 and 13) build on this foundation by adding instruction selection (IR → machine instructions), register allocation, and all the other passes needed for a full compiler backend. But even at this stage, you have a functional assembler and disassembler — tools that are useful in their own right for working with M88k machine code.
