# Chapter 12: Instruction Selection — A Comprehensive Explanation

## Overview and Context

This chapter sits at the very heart of the LLVM compiler backend. Everything discussed in prior chapters — defining registers, instruction formats, and encoding in the target description — was preparation for **this** moment: taking LLVM Intermediate Representation (IR) and turning it into actual machine instructions that a physical CPU can execute.

The chapter uses the **Motorola 88000 (M88k)** architecture as its running example. This is a historical RISC processor family, but the principles apply universally to any backend you'd write for LLVM.

The chapter covers two entirely different approaches to instruction selection: the traditional **Selection DAG** method and the newer **Global Instruction Selection (GlobalISel)** method. Both achieve the same end goal but through fundamentally different mechanisms.

---

## Part I: Defining the Rules of the Calling Convention

### What Is a Calling Convention and Why Does It Matter?

A **calling convention** is an agreement between the caller and callee of a function about how data is exchanged. It answers questions like:

- Which physical registers hold function arguments?
- What happens when there are more arguments than available registers?
- Which register holds the return value?
- Which registers must the callee preserve (callee-saved registers)?

This is critical because when LLVM IR says `define i32 @f1(i32 %a, i32 %b)`, the backend needs to know that `%a` lives in physical register `R2`, `%b` lives in `R3`, and the return value goes back into `R2`. Without this mapping, the backend cannot produce correct code.

### How Calling Convention Rules Work in LLVM

LLVM's calling convention system uses a **rule-based pattern matching** approach. The rules are defined as a sequence of **conditions** and **actions**:

1. A condition is checked (e.g., "is this parameter a 32-bit integer?").
2. If the condition holds, an action is executed (e.g., "assign it to the next free register from this list").
3. If the action succeeds (a register was available), we're done for that parameter.
4. If the action fails (all registers are used up), the next rule is tried.

The **state** of which registers have already been assigned is tracked by a framework class called `CCState`. You don't implement the loop yourself — LLVM's framework does the looping; you just provide the rules.

### The Target Description Syntax

In TableGen (the target description language), the rule for passing 32-bit integers in registers is written as:

```tablegen
CCIfType<[i32],
         CCAssignToReg<[R2, R3, R4, R5, R6, R7, R8, R9]>>,
```

This reads as: "If the parameter type is `i32`, try to assign it to one of registers R2 through R9." Since there are 8 registers listed, the first 8 integer parameters go into registers. If a function has a 9th parameter, this rule fails (the register list is exhausted), and the next rule kicks in:

```tablegen
CCAssignToStack<4, 4>
```

This is a **catch-all rule** with no condition. It says: "Put the parameter on the stack, using a 4-byte slot with 4-byte alignment." Since it has no condition, it always matches.

### Additional Predefined Conditions and Actions

LLVM provides many built-in building blocks:

- **`CCIfInReg`**: Checks if the argument has the `inreg` attribute (a hint that it should be in a register).
- **`CCIfVarArg`**: Evaluates to true if the function uses a variable argument list (like C's `printf`).
- **`CCPromoteToType`**: Widens a narrow type to a larger one. For example, an `i8` gets promoted to `i32` before being assigned to a register.
- **`CCPassIndirect`**: Stores the actual value on the stack and passes a *pointer* to it as the argument. This is used for large structs.

All of these are defined in `llvm/include/llvm/Target/TargetCallingConv.td`.

### The Complete M88k Calling Convention Definition

The file `M88kCallingConv.td` contains three definitions:

**1. Parameter passing rules (`CC_M88k`):**

```tablegen
def CC_M88k : CallingConv<[
  CCIfType<[i8, i16], CCPromoteToType<i32>>,
  CCIfType<[i32,f32],
           CCAssignToReg<[R2, R3, R4, R5, R6, R7, R8, R9]>>,
  CCAssignToStack<4, 4>
]>;
```

The logic flows like a chain:
- First rule: If the type is `i8` or `i16`, promote it to `i32`. After promotion, the analysis continues with the promoted type — it will now match the second rule.
- Second rule: If the type is `i32` or `f32`, try to put it in one of R2–R9.
- Third rule (catch-all): If we get here, put it on the stack.

**2. Return value rules (`RetCC_M88k`):**

```tablegen
def RetCC_M88k : CallingConv<[
  CCIfType<[i32], CCAssignToReg<[R2]>>
]>;
```

Only 32-bit integers are supported, and the return value always goes in R2. This is intentionally simple for the example.

**3. Callee-saved registers (`CSR_M88k`):**

```tablegen
def CSR_M88k :
    CalleeSavedRegs<(add R1, R30,
                         (sequence "R%d", 25, 14))>;
```

This defines which registers the called function must save and restore if it uses them. `R1` (the return address register) and `R30` (the frame pointer) are explicitly listed. The `(sequence "R%d", 25, 14)` is a TableGen operator that expands to `R25, R24, R23, ..., R14` — a compact way to list a contiguous range of registers in descending order instead of writing them all out by hand.

### Why Define This in the Target Description?

The key benefit is **reuse**. The same calling convention definition can be used by both the Selection DAG instruction selector and GlobalISel. If you hardcoded the rules in C++, you'd need to write them twice.

---

## Part II: Instruction Selection via the Selection DAG

### The Big Picture — The Four-Step Process

The Selection DAG is the traditional and most widely used instruction selection method in LLVM. Here is the high-level pipeline:

**Step 1: Build the DAG from IR.** Each LLVM IR basic block is converted into a Directed Acyclic Graph (DAG). Each node represents an operation (like `and`, `add`, `load`), and the edges represent data flow and control dependencies between operations.

**Step 2: Legalize types and operations.** The DAG may contain types or operations that the target hardware doesn't directly support. Legalization transforms them into supported equivalents. For example:
- A 64-bit value on a 32-bit machine gets **split** into two 32-bit values.
- A 64-bit multiply might be **lowered to a library call** (e.g., `__muldi3`).
- A complex operation like `ctpop` (count population / popcount) can be **expanded** into a sequence of simpler bit-manipulation instructions.

**Step 3: Pattern matching.** DAG nodes are matched against patterns defined in the target description and replaced with actual machine instructions. This is where the `and` DAG node becomes `M88k::ANDrr`.

**Step 4: Instruction scheduling.** The resulting machine instructions are reordered for better performance (e.g., to avoid pipeline stalls or to exploit instruction-level parallelism).

### Trade-offs of the Selection DAG

The Selection DAG generates high-quality code, but it has a cost: constructing the DAG data structure itself is expensive in terms of compile time. This motivated the development of alternative approaches like FastISel (fast but poor code quality, and it doesn't share code with the DAG) and GlobalISel (discussed later in this chapter).

### The Goal IR for This Chapter

The chapter aims to translate this minimal IR function:

```llvm
define i32 @f1(i32 %a, i32 %b) {
  %res = and i32 %a, %b
  ret i32 %res
}
```

This is a function that takes two 32-bit integers, performs a bitwise AND, and returns the result.

### The Two Key Classes

To implement instruction selection via the Selection DAG, two new classes are needed:

1. **`M88kISelLowering`** (also referred to as `M88kTargetLowering`): Customizes the DAG. It defines which types are legal, which operations are legal, and contains the code for lowering function arguments and return values. Think of it as the "configuration and policy" class.

2. **`M88kDAGToDAGISel`**: Performs the actual DAG-to-DAG transformation — replacing abstract DAG nodes with machine instruction nodes. Most of its implementation is *generated* from the target description patterns; you only need to add code for cases that can't be expressed as patterns.

### The Class Hierarchy (Figure 12.1)

The chapter describes a class hierarchy that connects everything:

- **`LLVMTargetMachine`** is the top-level base class.
  - **`M88kTargetMachine`** derives from it and provides `getSubtargetImpl()`.
    - It references **`M88kGenSubtargetInfo`** (generated from the target description).
- **`M88kSubtarget`** is the central hub. It owns:
  - **`M88kFrameLowering`** (stack frame management)
  - **`M88kInstrInfo`** (instruction information + hooks)
  - **`M88kTargetLowering`** (DAG customization)
- **`M88kInstrInfo`** owns **`M88kRegisterInfo`**.
- Generated classes (**`M88kGenInstrInfo`**, **`M88kGenRegisterInfo`**) provide the auto-generated code from the target description.

This hierarchy matters because the framework expects to find these classes through specific getter methods, and they all need to be wired up correctly.

---

### Implementing DAG Lowering — Handling Legal Types and Setting Operations

The `M88kTargetLowering` constructor is where you tell the framework about your hardware's capabilities.

**Step 1: Constructor signature.**

```cpp
M88kTargetLowering::M88kTargetLowering(
    const TargetMachine &TM, const M88kSubtarget &STI)
    : TargetLowering(TM), Subtarget(STI) {
```

It takes the overall `TargetMachine` (general backend configuration, like which passes run) and the `M88kSubtarget` (characteristics of the specific CPU we're targeting).

**Step 2: Register classes for machine value types.**

```cpp
addRegisterClass(MVT::i32, &M88k::GPRRegClass);
```

This tells the framework: "32-bit integer values should use the General Purpose Register class." If the target also had floating-point registers, you'd add `addRegisterClass(MVT::f32, &M88k::FPRRegClass)` here.

**Step 3: Compute derived register properties.**

```cpp
computeRegisterProperties(Subtarget.getRegisterInfo());
```

After declaring all register classes, this call lets the framework pre-compute various properties (like spill sizes, register costs, etc.) that are used later in register allocation.

**Step 4: Stack pointer register.**

```cpp
setStackPointerRegisterToSaveRestore(M88k::R31);
```

This tells the framework which register is the stack pointer. This is needed for prologue/epilogue generation and stack frame management.

**Step 5: Boolean representation.**

```cpp
setBooleanContents(ZeroOrOneBooleanContent);
```

Different architectures represent boolean values differently. On M88k, a boolean `true` is stored as `1` in bit 0, with all other bits cleared. Other options include `ZeroOrNegativeOneBooleanContent` (where true is all-ones / -1).

**Step 6: Function alignment.**

```cpp
setMinFunctionAlignment(Align(4));
setPrefFunctionAlignment(Align(4));
```

The minimum alignment is what the hardware *requires* for correct execution (M88k needs 4-byte aligned instructions). The preferred alignment is an optimization hint (e.g., aligning to cache line boundaries).

**Step 7: Legal operations.**

```cpp
setOperationAction(ISD::AND, MVT::i32, Legal);
setOperationAction(ISD::OR, MVT::i32, Legal);
setOperationAction(ISD::XOR, MVT::i32, Legal);
```

This declares that the AND, OR, and XOR operations on 32-bit integers are directly supported by the hardware. The framework will not try to lower or transform these — they can be directly matched to machine instructions.

**Step 8: Non-legal operations.**

```cpp
setOperationAction(ISD::CTPOP, MVT::i32, Expand);
```

The `CTPOP` (population count) operation is *not* directly supported by M88k hardware, so we request that it be **expanded** into a sequence of simpler operations.

The available action types are:
- **`Legal`**: Directly supported by hardware.
- **`Promote`**: Widen the type (e.g., `i16` AND becomes `i32` AND).
- **`Expand`**: Replace with a sequence of other operations.
- **`LibCall`**: Lower to a runtime library call.
- **`Custom`**: Call the `LowerOperation()` hook method for you to implement your own lowering logic.

### The Connection Between Target Description and C++ Code

This is a crucial conceptual point. In the target description (`M88kInstrInfo.td`), a machine instruction was defined with a pattern:

```tablegen
let isCommutable = 1 in
  def ANDrr : F_LR<0b01000, Func, /*comp=*/0b0, "and",
                    [(set i32:$rd,
                          (and GPROpnd:$rs1, GPROpnd:$rs2))]>;
```

Here's how the pieces connect:

- The pattern uses `and`, which is a DAG node type. In C++, this is `ISD::AND`.
- In the C++ code, we called `setOperationAction(ISD::AND, MVT::i32, Legal)` to declare this operation as legal.
- During instruction selection, when the DAG contains an `and` node with `i32` operands from the GPR register class, the pattern matcher will replace it with `M88k::ANDrr`.
- The string `"and"` in the instruction definition is the assembly mnemonic.

So the most important task when developing instruction selection is: **(a)** define the correct legalization actions in C++, and **(b)** attach correct patterns to instruction definitions in the target description.

---

### Implementing DAG Lowering — Lowering Formal Arguments

When the framework encounters a function, it needs to know how to get the function's arguments from their physical locations (registers or stack slots) into virtual registers that the DAG can work with. This is done in `LowerFormalArguments()`.

**The method signature:**

```cpp
SDValue M88kTargetLowering::LowerFormalArguments(
    SDValue Chain, CallingConv::ID CallConv,
    bool IsVarArg,
    const SmallVectorImpl<ISD::InputArg> &Ins,
    const SDLoc &DL, SelectionDAG &DAG,
    SmallVectorImpl<SDValue> &InVals) const {
```

Let's understand each parameter:

- **`Chain`**: Represents control flow. DAG nodes can be chained together to enforce ordering. The method returns an updated chain.
- **`CallConv`**: Identifies which calling convention is used (e.g., C calling convention, fast call, etc.).
- **`IsVarArg`**: True if the function has a variable argument list.
- **`Ins`**: The list of arguments that need to be handled, with their types and flags.
- **`DL`**: Debug location information.
- **`DAG`**: Access to the `SelectionDAG` object (the actual graph data structure).
- **`InVals`**: Output vector where we store the DAG values representing each argument.

**Step-by-step implementation:**

1. **Get references to machine-level objects:**

```cpp
MachineFunction &MF = DAG.getMachineFunction();
MachineRegisterInfo &MRI = MF.getRegInfo();
```

`MachineFunction` is the machine-level representation of a function. `MachineRegisterInfo` manages virtual and physical register tracking.

2. **Invoke the generated calling convention code:**

```cpp
SmallVector<CCValAssign, 16> ArgLocs;
CCState CCInfo(CallConv, IsVarArg, MF, ArgLocs, *DAG.getContext());
CCInfo.AnalyzeFormalArguments(Ins, CC_M88k);
```

`CCState` is the framework class that tracks register usage. `AnalyzeFormalArguments` runs the rules we defined in `CC_M88k` against the actual function parameters. The result goes into `ArgLocs` — a vector of `CCValAssign` objects, each describing where a particular argument ended up (which register, or which stack offset).

The name `CC_M88k` directly corresponds to the `def CC_M88k : CallingConv<[...]>` from the `.td` file. The generated C++ function has the same name.

3. **Loop over argument locations and create DAG nodes:**

For each argument, we check if it was assigned to a register:

```cpp
for (unsigned I = 0, E = ArgLocs.size(); I != E; ++I) {
    SDValue ArgValue;
    CCValAssign &VA = ArgLocs[I];
    EVT LocVT = VA.getLocVT();
```

4. **Handle register arguments:**

If the argument is in a register, we need to:
- Determine the correct register class (for `i32`, it's `GPRRegClass`).
- Create a virtual register.
- Mark the physical register as "live-in" (meaning the function expects a value in it upon entry).
- Create a `CopyFromReg` DAG node that represents moving the value from the physical register to the virtual register.

```cpp
if (VA.isRegLoc()) {
    const TargetRegisterClass *RC;
    switch (LocVT.getSimpleVT().SimpleTy) {
    default:
        llvm_unreachable("Unexpected argument type");
    case MVT::i32:
        RC = &M88k::GPRRegClass;
        break;
    }
    Register VReg = MRI.createVirtualRegister(RC);
    MRI.addLiveIn(VA.getLocReg(), VReg);
    ArgValue = DAG.getCopyFromReg(Chain, DL, VReg, LocVT);
```

5. **Handle type promotion:**

Remember that in `CC_M88k`, we said `i8` and `i16` should be promoted to `i32`. When a value is promoted, we need to ensure the promotion is reflected in the DAG. If the calling convention says the value was sign-extended (`SExt`) or zero-extended (`ZExt`), we insert assertion nodes, then truncate back to the original type:

```cpp
if (VA.getLocInfo() == CCValAssign::SExt)
    ArgValue = DAG.getNode(ISD::AssertSext, DL, LocVT, ArgValue,
                           DAG.getValueType(VA.getValVT()));
else if (VA.getLocInfo() == CCValAssign::ZExt)
    ArgValue = DAG.getNode(ISD::AssertZext, DL, LocVT, ArgValue,
                           DAG.getValueType(VA.getValVT()));

if (VA.getLocInfo() != CCValAssign::Full)
    ArgValue = DAG.getNode(ISD::TRUNCATE, DL, VA.getValVT(), ArgValue);
```

The `AssertSext`/`AssertZext` nodes are hints to the optimizer that the upper bits have a known pattern. Then `TRUNCATE` narrows the value back to the original type (e.g., from `i32` back to `i8`).

6. **Store the result and handle edge cases:**

```cpp
    InVals.push_back(ArgValue);
}
```

Stack-located arguments (`VA.isMemLoc()`) are handled with loads, but since load/store instructions aren't defined yet, this case triggers `llvm_unreachable`. Variable argument handling is similarly deferred.

7. **Return the chain:**

```cpp
return Chain;
```

---

### Implementing DAG Lowering — Lowering Return Values

Return value handling is the mirror image of argument handling, with one key addition: we need a special DAG node type to glue return values together.

**Why gluing?** When a function returns a value, the return instruction must see the physical register holding the return value as "live." If the instruction scheduler were to reorder the register copy and the return instruction, the value could be lost. Gluing prevents this reordering.

**Target description additions:**

A new DAG node type `RET_GLUE` is defined:

```tablegen
def retglue : SDNode<"M88kISD::RET_GLUE", SDTNone,
                     [SDNPHasChain, SDNPOptInGlue, SDNPVariadic]>;
```

The properties mean:
- `SDNPHasChain`: This node participates in the control flow chain.
- `SDNPOptInGlue`: This node can optionally accept a glue operand (to enforce ordering with the preceding copy-to-register).
- `SDNPVariadic`: The node can have a variable number of operands (because the number of return values varies).

A pseudo-instruction `RET` matches this node:

```tablegen
let isReturn = 1, isTerminator = 1, isBarrier = 1,
    AsmString = "RET" in
  def RET : Pseudo<(outs), (ins), [(retglue)]>;
```

This is a **pseudo-instruction** — it doesn't correspond to a real hardware instruction. It's a placeholder that will be expanded later (in `expandPostRAPseudo`) into the real `jmp %r1` instruction. The flags `isReturn`, `isTerminator`, and `isBarrier` tell the framework about the control flow properties.

**The `LowerReturn()` implementation:**

The structure mirrors `LowerFormalArguments()`:

1. Analyze return values using `RetCC_M88k`:

```cpp
SmallVector<CCValAssign, 16> RetLocs;
CCState RetCCInfo(CallConv, IsVarArg, DAG.getMachineFunction(),
                  RetLocs, *DAG.getContext());
RetCCInfo.AnalyzeReturn(Outs, RetCC_M88k);
```

2. Loop over return value locations, copying values to physical registers with gluing:

```cpp
SDValue Glue;
SmallVector<SDValue, 4> RetOps(1, Chain);
for (unsigned I = 0, E = RetLocs.size(); I != E; ++I) {
    CCValAssign &VA = RetLocs[I];
    Register Reg = VA.getLocReg();
    Chain = DAG.getCopyToReg(Chain, DL, Reg, OutVals[I], Glue);
    Glue = Chain.getValue(1);
    RetOps.push_back(DAG.getRegister(Reg, VA.getLocVT()));
}
```

The `getCopyToReg` creates a node that copies a value to a physical register. The `Glue` value chains these copies together so they can't be reordered. `Chain.getValue(1)` extracts the glue output from the copy node.

3. Build and return the `RET_GLUE` node:

```cpp
RetOps[0] = Chain;
if (Glue.getNode())
    RetOps.push_back(Glue);
return DAG.getNode(M88kISD::RET_GLUE, DL, MVT::Other, RetOps);
```

---

### Implementing DAG-to-DAG Transformations

The `M88kDAGToDAGISel` class is the pass that performs the actual pattern matching — replacing abstract DAG nodes with concrete machine instruction nodes. Importantly, **most of the code is generated** from the target description patterns.

**Class definition:**

```cpp
class M88kDAGToDAGISel : public SelectionDAGISel {
public:
    static char ID;
    M88kDAGToDAGISel(M88kTargetMachine &TM, CodeGenOpt::Level OptLevel)
        : SelectionDAGISel(ID, TM, OptLevel) {}
    void Select(SDNode *Node) override;
    #include "M88kGenDAGISel.inc"  // Generated pattern matching code
};
```

The `#include "M88kGenDAGISel.inc"` pulls in the auto-generated code that implements the pattern matching logic derived from all the instruction patterns in the `.td` files.

**Pass registration:**

LLVM backends still use the legacy pass manager for code generation:

```cpp
char M88kDAGToDAGISel::ID = 0;
INITIALIZE_PASS(M88kDAGToDAGISel, DEBUG_TYPE, PASS_NAME, false, false)
```

The static `ID` member uniquely identifies the pass. `INITIALIZE_PASS` is a macro that generates the registration boilerplate.

A factory method creates instances:

```cpp
FunctionPass *llvm::createM88kISelDag(M88kTargetMachine &TM,
                                       CodeGenOpt::Level OptLevel) {
    return new M88kDAGToDAGISel(TM, OptLevel);
}
```

**The `Select()` method:**

```cpp
void M88kDAGToDAGISel::Select(SDNode *Node) {
    SelectCode(Node);
}
```

`SelectCode` is the generated method that does the pattern matching. If you have a transformation too complex to express in TableGen patterns, you'd add your own C++ code *before* calling `SelectCode`. For example, you might check the node's opcode and manually construct a machine instruction node, then return early — only falling through to `SelectCode` for the standard cases.

---

## Part III: Adding Register and Instruction Information

### M88kRegisterInfo

The `M88kRegisterInfo` class provides access to register information from the target description and implements hooks that can't be expressed declaratively.

**Header file pattern:**

```cpp
#define GET_REGINFO_HEADER
#include "M88kGenRegisterInfo.inc"
```

This is a pattern used throughout LLVM backends. The generated `.inc` file contains different sections of code, controlled by preprocessor macros. By defining `GET_REGINFO_HEADER` before including, you get just the class declarations.

**Constructor:**

```cpp
M88kRegisterInfo::M88kRegisterInfo()
    : M88kGenRegisterInfo(M88k::R1) {}
```

The parameter `M88k::R1` tells the superclass which register holds the **return address**. On M88k, when a function is called, the return address is placed in R1.

**Callee-saved registers:**

```cpp
const MCPhysReg *M88kRegisterInfo::getCalleeSavedRegs(
    const MachineFunction *MF) const {
    return CSR_M88k_SaveList;
}
```

`CSR_M88k_SaveList` is generated from the `CSR_M88k` definition in the target description. It's an array of physical register numbers.

**Reserved registers:**

```cpp
BitVector M88kRegisterInfo::getReservedRegs(
    const MachineFunction &MF) const {
    BitVector Reserved(getNumRegs());
    Reserved.set(M88k::R0);   // Always reads as 0
    Reserved.set(M88k::R28);  // Reserved for linker
    Reserved.set(M88k::R29);  // Reserved for linker
    Reserved.set(M88k::R31);  // Stack pointer
    return Reserved;
}
```

Reserved registers cannot be used by the register allocator. This list is **dynamic** — it can depend on the function being compiled (e.g., if a frame pointer is needed, R30 would also be reserved). This dynamic nature is exactly why it can't be generated from the static target description.

**Frame register and frame index elimination:**

```cpp
Register M88kRegisterInfo::getFrameRegister(
    const MachineFunction &MF) const {
    return M88k::R30;
}

bool M88kRegisterInfo::eliminateFrameIndex(
    MachineBasicBlock::iterator MI, int SPAdj,
    unsigned FIOperandNum, RegScavenger *RS) const {
    return false;
}
```

These are declared as `pure virtual` in the base class, so they *must* be implemented even if frame support isn't complete yet. `eliminateFrameIndex()` is called to replace abstract frame index operands with concrete stack pointer + offset addressing. Since load/store instructions aren't implemented yet, it just returns `false`.

### M88kInstrInfo

This class provides instruction-level hooks and owns the register information.

**Header file:**

```cpp
#define GET_INSTRINFO_HEADER
#include "M88kGenInstrInfo.inc"

class M88kInstrInfo : public M88kGenInstrInfo {
    const M88kRegisterInfo RI;
    [[maybe_unused]] M88kSubtarget &STI;
    virtual void anchor();
public:
    explicit M88kInstrInfo(M88kSubtarget &STI);
    const M88kRegisterInfo &getRegisterInfo() const { return RI; }
    bool expandPostRAPseudo(MachineInstr &MI) const override;
};
```

Key points:
- The class **owns** an `M88kRegisterInfo` instance. Other parts of the framework access register info through instruction info.
- The `anchor()` method is a common LLVM pattern — it's a virtual method whose definition is placed in the `.cpp` file to "pin" the vtable to that translation unit, preventing duplicate vtable issues across shared libraries.
- `[[maybe_unused]]` is a C++17 attribute that suppresses "unused variable" warnings for the `STI` member, which isn't used yet but will be needed as the backend grows.

**The `expandPostRAPseudo()` method:**

This is called **after register allocation** (hence "PostRA") to expand pseudo-instructions into real ones:

```cpp
bool M88kInstrInfo::expandPostRAPseudo(MachineInstr &MI) const {
    MachineBasicBlock &MBB = *MI.getParent();
    switch (MI.getOpcode()) {
    default:
        return false;
    case M88k::RET: {
        MachineInstrBuilder MIB =
            BuildMI(MBB, &MI, MI.getDebugLoc(), get(M88k::JMP))
                .addReg(M88k::R1, RegState::Undef);
        for (auto &MO : MI.operands()) {
            if (MO.isImplicit())
                MIB.add(MO);
        }
        break;
    }
    }
    MBB.erase(MI);
    return true;
}
```

Here's what happens:
1. If the instruction is the `RET` pseudo, we build a real `JMP` (jump) instruction targeting R1 (the return address).
2. `RegState::Undef` indicates that R1's value comes from elsewhere (it was set when the function was called).
3. We copy all **implicit operands** from the pseudo to the real instruction. These implicit operands represent the return values — they tell the register allocator that those registers are live at the return point.
4. The pseudo-instruction is erased from the basic block.
5. Returning `true` indicates we handled the instruction; `false` means "not a pseudo I know about."

---

## Part IV: Putting an Empty Frame Lowering in Place

### What Is Frame Lowering?

The **stack frame** is the region of the stack that a function uses for local variables, spilled registers, saved return addresses, etc. The **prologue** is the code at the beginning of a function that sets up the frame (allocating stack space, saving registers). The **epilogue** is the code at the end that tears it down.

### Why an Empty Implementation?

At the current development stage, the backend doesn't support load/store instructions, so it can't actually create a prologue or epilogue (which require stack manipulation). However, the instruction selection framework **requires** that a `TargetFrameLowering` subclass exists. The solution is to provide a minimal implementation.

**Constructor:**

```cpp
M88kFrameLowering::M88kFrameLowering()
    : TargetFrameLowering(
          TargetFrameLowering::StackGrowsDown, Align(8),
          0, Align(8), false /* StackRealignable */) {}
```

The parameters describe the stack layout:
- **`StackGrowsDown`**: The stack grows toward lower addresses (standard for most architectures).
- **`Align(8)`**: The stack is aligned on 8-byte boundaries.
- **`0`**: The offset of the local area from the stack pointer. Zero means locals start immediately below the caller's stack pointer.
- **`Align(8)`**: The stack should remain 8-byte aligned even during function calls.
- **`false`**: The stack cannot be realigned (would need special support).

The `emitPrologue()`, `emitEpilogue()`, and `hasFP()` methods are all empty/trivial since they're pure virtual in the base class but not yet functional.

---

## Part V: Emitting Machine Instructions

### The Two-Level Instruction Representation

LLVM has two levels of instruction representation after instruction selection:

1. **`MachineInstr`**: A rich representation that carries metadata like labels, flags, debug information, and implicit operands. Used during register allocation and other late passes.

2. **`MCInst`**: A lean representation used by the machine code (MC) layer for writing to object files or printing assembly text. Contains only the opcode and explicit operands.

The assembly printer's job is to **lower** `MachineInstr` instances to `MCInst` instances and emit them.

### M88kAsmPrinter

This class is responsible for emitting an entire compilation unit:

```cpp
class M88kAsmPrinter : public AsmPrinter {
public:
    explicit M88kAsmPrinter(TargetMachine &TM,
                             std::unique_ptr<MCStreamer> Streamer)
        : AsmPrinter(TM, std::move(Streamer)) {}
    StringRef getPassName() const override { return "M88k Assembly Printer"; }
    void emitInstruction(const MachineInstr *MI) override;
};
```

**Registration:**

```cpp
extern "C" LLVM_EXTERNAL_VISIBILITY void LLVMInitializeM88kAsmPrinter() {
    RegisterAsmPrinter<M88kAsmPrinter> X(getTheM88kTarget());
}
```

The `extern "C"` and `LLVM_EXTERNAL_VISIBILITY` ensure this function can be found when the backend is loaded dynamically.

**Instruction emission:**

```cpp
void M88kAsmPrinter::emitInstruction(const MachineInstr *MI) {
    MCInst LoweredMI;
    M88kMCInstLower Lower(MF->getContext(), *this);
    Lower.lower(MI, LoweredMI);
    EmitToStreamer(*OutStreamer, LoweredMI);
}
```

The flow is: `MachineInstr` → `M88kMCInstLower::lower()` → `MCInst` → `EmitToStreamer()`.

The base class `AsmPrinter` provides many customization hooks:
- `emitStartOfAsmFile()` / `emitEndOfAsmFile()`: For target-specific code at the beginning/end of a file.
- `emitFunctionBodyStart()` / `emitFunctionBodyEnd()`: For per-function customization.

### M88kMCInstLower

This class handles the actual conversion:

**Lowering operands:**

```cpp
MCOperand M88kMCInstLower::lowerOperand(const MachineOperand &MO) const {
    switch (MO.getType()) {
    case MachineOperand::MO_Register:
        return MCOperand::createReg(MO.getReg());
    case MachineOperand::MO_Immediate:
        return MCOperand::createImm(MO.getImm());
    default:
        llvm_unreachable("Operand type not handled");
    }
}
```

Only registers and immediates are handled. When expressions (like symbol references for function calls or global variable addresses) are added, this method would need to create `MCExpr`-based operands.

**Lowering instructions:**

```cpp
void M88kMCInstLower::lower(const MachineInstr *MI, MCInst &OutMI) const {
    OutMI.setOpcode(MI->getOpcode());
    for (auto &MO : MI->operands()) {
        if (!MO.isReg() || !MO.isImplicit())
            OutMI.addOperand(lowerOperand(MO));
    }
}
```

The opcode is copied directly. Operands are lowered one by one, but **implicit operands are skipped**. Implicit operands exist for the benefit of the register allocator and are not part of the actual encoded instruction. The condition `!MO.isReg() || !MO.isImplicit()` means: "include this operand if it's not a register, OR if it is a register but not implicit." In other words, only implicit register operands are filtered out.

---

## Part VI: Creating the Target Machine and the Sub-Target

### M88kSubtarget — Describing Hardware Features

The `M88kSubtarget` class captures the specific hardware configuration:

**Declaration highlights:**

```cpp
class M88kSubtarget : public M88kGenSubtargetInfo {
    Triple TargetTriple;
    M88kInstrInfo InstrInfo;
    M88kTargetLowering TLInfo;
    M88kFrameLowering FrameLowering;
    // ...
};
```

The sub-target **owns** the instruction info, target lowering, and frame lowering objects. It also includes auto-generated getter methods for features:

```cpp
#define GET_SUBTARGETINFO_MACRO(ATTRIBUTE, DEFAULT, GETTER) \
    bool GETTER() const { return ATTRIBUTE; }
#include "M88kGenSubtargetInfo.inc"
```

This macro expansion creates methods like `hasFeatureX()` for each feature defined in the target description.

**Constructor:**

```cpp
M88kSubtarget::M88kSubtarget(const Triple &TT,
                              const std::string &CPU,
                              const std::string &FS,
                              const TargetMachine &TM)
    : M88kGenSubtargetInfo(TT, CPU, /*TuneCPU*/ CPU, FS),
      TargetTriple(TT), InstrInfo(*this),
      TLInfo(TM, *this), FrameLowering() {}
```

The generated superclass takes two CPU parameters: one for the instruction set and one for scheduling tuning. The use case is: you might want to generate code compatible with an older CPU (instruction set) but optimized for a newer CPU's pipeline (scheduling). Since the M88k backend doesn't support this distinction yet, the same CPU name is used for both.

### M88kTargetMachine — Configuring the Backend

This is the top-level class that ties everything together.

**Key members:**

```cpp
class M88kTargetMachine : public LLVMTargetMachine {
    std::unique_ptr<TargetLoweringObjectFile> TLOF;
    mutable StringMap<std::unique_ptr<M88kSubtarget>> SubtargetMap;
    // ...
};
```

- **`TLOF`**: A `TargetLoweringObjectFile` instance that provides information about the object file format (like section names). For M88k, this is `TargetLoweringObjectFileELF` since the target uses ELF.
- **`SubtargetMap`**: A map from configuration strings to sub-target instances. Different functions in the same module can target different CPUs/features, so multiple sub-targets may exist simultaneously. The `mutable` keyword allows modification in `const` methods.

**The data layout string:**

The `computeDataLayout()` function constructs a string describing the target's memory layout:

```cpp
std::string Ret;
Ret += "E";                                              // Big-endian
Ret += DataLayout::getManglingComponent(TT);             // ELF symbol mangling
Ret += "-p:32:32:32";                                    // Pointers: 32-bit, 32-bit aligned, 32-bit store size
Ret += "-i1:8:8-i8:8:8-i16:16:16-i32:32:32-i64:64:64"; // Integer types: naturally aligned
Ret += "-f32:32:32-f64:64:64";                           // Float types: naturally aligned
Ret += "-a:8:16";                                        // Aggregates: 8-bit ABI alignment, 16-bit preferred
Ret += "-n32";                                           // Native integer width: 32-bit
return Ret;
```

This string is the same format used by the LLVM `DataLayout` class. The frontend (e.g., Clang) uses this to make decisions about struct layout, which must match the backend's expectations.

**Constructor:**

```cpp
M88kTargetMachine::M88kTargetMachine(
    const Target &T, const Triple &TT, StringRef CPU,
    StringRef FS, const TargetOptions &Options,
    std::optional<Reloc::Model> RM,
    std::optional<CodeModel::Model> CM,
    CodeGenOpt::Level OL, bool JIT)
    : LLVMTargetMachine(T, computeDataLayout(TT, CPU, FS), TT, CPU,
                         FS, Options, !RM ? Reloc::Static : *RM,
                         getEffectiveCodeModel(CM, CodeModel::Medium), OL),
      TLOF(std::make_unique<TargetLoweringObjectFileELF>()) {
    initAsmInfo();
}
```

Notable points:
- If no relocation model is specified, it defaults to `Reloc::Static`.
- If no code model is specified, it defaults to `CodeModel::Medium`.
- `initAsmInfo()` must be called in the constructor body — it initializes many data members in the base class.
- The `JIT` parameter indicates whether this target machine is being used for just-in-time compilation.

**Sub-target caching (`getSubtargetImpl`):**

```cpp
const M88kSubtarget *M88kTargetMachine::getSubtargetImpl(
    const Function &F) const {
    Attribute CPUAttr = F.getFnAttribute("target-cpu");
    Attribute FSAttr = F.getFnAttribute("target-features");
    std::string CPU = !CPUAttr.hasAttribute(Attribute::None)
                          ? CPUAttr.getValueAsString().str()
                          : TargetCPU;
    std::string FS = !FSAttr.hasAttribute(Attribute::None)
                         ? FSAttr.getValueAsString().str()
                         : TargetFS;
    auto &I = SubtargetMap[CPU + FS];
    if (!I) {
        resetTargetOptions(F);
        I = std::make_unique<M88kSubtarget>(TargetTriple, CPU, FS, *this);
    }
    return I.get();
}
```

Each function can have different `target-cpu` and `target-features` attributes. The method checks these attributes, falls back to default values if absent, and caches sub-target instances by their configuration string.

**Pass pipeline configuration:**

```cpp
class M88kPassConfig : public TargetPassConfig {
public:
    M88kPassConfig(M88kTargetMachine &TM, PassManagerBase &PM)
        : TargetPassConfig(TM, PM) {}
    bool addInstSelector() override;
};

bool M88kPassConfig::addInstSelector() {
    addPass(createM88kISelDag(getTM<M88kTargetMachine>(), getOptLevel()));
    return false;
}
```

The `addInstSelector()` method is called by the framework to add instruction selection passes. We add our `M88kDAGToDAGISel` pass. Returning `false` indicates that this pass converts LLVM IR to machine instructions (as opposed to returning `true`, which would mean "I didn't add anything, use the default").

**Registration:**

```cpp
extern "C" LLVM_EXTERNAL_VISIBILITY void LLVMInitializeM88kTarget() {
    RegisterTargetMachine<M88kTargetMachine> X(getTheM88kTarget());
    auto &PR = *PassRegistry::getPassRegistry();
    initializeM88kDAGToDAGISelPass(PR);
}
```

Both the target machine and the DAG-to-DAG pass must be registered.

### Running the Example

With everything in place, the simple IR:

```llvm
define i32 @f1(i32 %a, i32 %b) {
  %res = and i32 %a, %b
  ret i32 %res
}
```

Produces this assembly:

```asm
f1:
    and %r2, %r2, %r3
    jmp %r1
```

This is correct:
- `%a` arrives in `%r2`, `%b` in `%r3` (per `CC_M88k`).
- The `and` instruction operates on `%r2` and `%r3`, storing the result in `%r2`.
- The result is already in `%r2` (per `RetCC_M88k`), so no copy is needed.
- `jmp %r1` returns to the caller (R1 holds the return address).

---

## Part VII: Global Instruction Selection (GlobalISel)

### Motivation — Why Another Approach?

The Selection DAG is a **monolithic algorithm**: it builds an entire graph, legalizes it, pattern-matches it, and schedules it — all as one big step per basic block. This is expensive. At optimization level 0 (where compile speed matters most), this overhead is painful.

**FastISel** was the first attempt at a faster alternative. It works at the instruction level without building a DAG, producing code quickly but of poor quality. It also doesn't share code with the Selection DAG, meaning each target that supports FastISel has to maintain two completely separate instruction selection implementations. Not all targets support it.

**GlobalISel** takes a different approach:

1. Instead of building a new data structure (the DAG), it works directly with **machine functions** and **machine basic blocks** — the same data structures used by all other backend passes.
2. It breaks instruction selection into **small, independent passes**, rather than one monolithic step.
3. Because it uses passes, you can insert optimization passes (combiners) between the steps. Turning them off gives faster compilation; turning them on gives better code. This allows **scaling** between speed and quality.
4. Unlike the Selection DAG (which works basic-block-by-basic-block), GlobalISel works at the **function level**, enabling cross-basic-block optimizations — hence "global" instruction selection.

### The Four-Step GlobalISel Pipeline

**Step 1: IR Translator.** LLVM IR is lowered into **generic machine instructions**. These are target-independent instructions like `G_AND`, `G_ADD`, `G_LOAD` that represent the most common operations found in real hardware.

**Step 2: Legalizer.** Generic instructions with unsupported types or operations are transformed into legal forms (e.g., splitting 64-bit operations into pairs of 32-bit operations).

**Step 3: Register Bank Selector.** Each operand is mapped to a **register bank** (e.g., general-purpose vs. floating-point). This is important because moving values between register banks can be expensive.

**Step 4: Instruction Selector.** Generic instructions are replaced with real machine instructions, using the same patterns defined in the target description.

### Lowering Arguments and Return Values (M88kCallLowering)

The `M88kCallLowering` class handles function entry and exit:

```cpp
class M88kCallLowering : public CallLowering {
public:
    M88kCallLowering(const M88kTargetLowering &TLI);
    bool lowerReturn(...) const override;
    bool lowerFormalArguments(...) const override;
    bool enableBigEndian() const override { return true; }
};
```

**Critical note**: The `enableBigEndian()` override is essential. Without it, the framework would assume little-endian, producing incorrect machine code for the big-endian M88k.

**Value handlers:**

The generated calling convention code tells us *where* each parameter is (which register or stack offset), but we need custom code to generate the actual machine instructions for the transfer. This is done via **value handler** classes:

- **`IncomingValueHandler`**: For function arguments (values coming *in*).
- **`OutgoingValueHandler`**: For return values (values going *out*).

For the `FormalArgHandler` (incoming arguments), the key method is `assignValueToReg()`:

```cpp
void FormalArgHandler::assignValueToReg(
    Register ValVReg, Register PhysReg, CCValAssign VA) {
    MIRBuilder.getMRI()->addLiveIn(PhysReg);
    MIRBuilder.getMBB().addLiveIn(PhysReg);
    CallLowering::IncomingValueHandler::assignValueToReg(
        ValVReg, PhysReg, VA);
}
```

This marks the physical register as live-in (both at the function level and basic block level) and delegates to the base class to generate the copy instruction.

**`lowerFormalArguments()` implementation:**

1. Convert IR function parameters into `ArgInfo` objects:

```cpp
SmallVector<ArgInfo, 8> SplitArgs;
for (const auto &[I, Arg] : llvm::enumerate(F.args())) {
    ArgInfo OrigArg{VRegs[I], Arg.getType(), static_cast<unsigned>(I)};
    setArgFlags(OrigArg, I + AttributeList::FirstArgIndex, DL, F);
    splitToValueTypes(OrigArg, SplitArgs, DL, F.getCallingConv());
}
```

`splitToValueTypes()` handles the case where a single IR parameter needs multiple virtual registers (e.g., a 64-bit value on a 32-bit machine).

2. Use the framework to generate machine code:

```cpp
IncomingValueAssigner ArgAssigner(CC_M88k);
FormalArgHandler ArgHandler(MIRBuilder, MRI);
return determineAndHandleAssignments(
    ArgHandler, ArgAssigner, SplitArgs, MIRBuilder,
    F.getCallingConv(), F.isVarArg());
```

The framework runs the calling convention rules (`CC_M88k`), determines assignments, and calls the handler for each one. Note how the same `CC_M88k` from the target description is reused — this is the benefit of defining calling conventions declaratively.

### Legalizing Generic Machine Instructions (M88kLegalizerInfo)

The legalizer defines what's legal:

```cpp
getActionDefinitionsBuilder({G_AND, G_OR, G_XOR})
    .legalFor({S32})
    .clampScalar(0, S32, S32);
```

This reads as: "The G_AND, G_OR, and G_XOR instructions are legal when all operands are 32-bit scalars (`S32`). If any operand is wider or narrower, clamp it to 32-bit."

The `clampScalar(0, S32, S32)` means: for operand index 0 (which in LLVM convention is the result), the minimum type is `S32` and the maximum type is `S32`. Values smaller than 32 bits are widened, values larger are split.

**What "legal" means in GlobalISel** is slightly more flexible than in the Selection DAG. A generic instruction is legal if the instruction selector can translate it. You could declare an 8-bit operation as legal even though the hardware works on 32-bit values, as long as the instruction selector knows how to handle it correctly.

### Selecting a Register Bank (M88kRegisterBankInfo)

**What is a register bank?** It's a set of related registers. Common banks include general-purpose registers and floating-point registers. Moving values within a bank is cheap; moving between banks is expensive (or impossible on some architectures).

**Target description addition:**

```tablegen
def GRRegBank : RegisterBank<"GRRB", [GPR, GPR64]>;
```

This defines a single register bank named "GRRB" that encompasses both the 32-bit and 64-bit general-purpose register classes.

**Partial mappings:**

These describe how values map to register banks:

```cpp
RegisterBankInfo::PartialMapping M88kGenRegisterBankInfo::PartMappings[]{
    {0, 32, M88k::GRRegBank},  // PMI_GR32: 32-bit value starting at bit 0
    {0, 64, M88k::GRRegBank},  // PMI_GR64: 64-bit value starting at bit 0
};
```

**Value mappings:**

For three-address instructions (one result, two sources), we need three partial mappings per width:

```cpp
RegisterBankInfo::ValueMapping M88kGenRegisterBankInfo::ValMappings[]{
    {nullptr, 0},                                              // Invalid (index 0)
    {&PartMappings[PMI_GR32], 1},  // 32-bit operand 1
    {&PartMappings[PMI_GR32], 1},  // 32-bit operand 2
    {&PartMappings[PMI_GR32], 1},  // 32-bit operand 3
    {&PartMappings[PMI_GR64], 1},  // 64-bit operand 1
    {&PartMappings[PMI_GR64], 1},  // 64-bit operand 2
    {&PartMappings[PMI_GR64], 1},  // 64-bit operand 3
};
```

The accessor function uses arithmetic to index into this array:

```cpp
const RegisterBankInfo::ValueMapping *
M88kGenRegisterBankInfo::getValueMapping(PartialMappingIdx RBIdx) {
    return &ValMappings[1 + 3*RBIdx];
}
```

For `PMI_GR32` (value 0), this returns `&ValMappings[1]`, pointing to three consecutive 32-bit mappings. For `PMI_GR64` (value 1), it returns `&ValMappings[4]`, pointing to three consecutive 64-bit mappings.

**The chapter notes that this code could and should be generated from the target description** — a comment in the LLVM source even says so — but as of this writing, it must be created manually. This is a known area for improvement.

**The `getInstrMapping()` method:**

```cpp
const RegisterBankInfo::InstructionMapping &
M88kRegisterBankInfo::getInstrMapping(const MachineInstr &MI) const {
    const ValueMapping *OperandsMapping = nullptr;
    switch (MI.getOpcode()) {
    case TargetOpcode::G_AND:
    case TargetOpcode::G_OR:
    case TargetOpcode::G_XOR:
        OperandsMapping = getValueMapping(PMI_GR32);
        break;
    default:
        MI.dump();  // Debug aid: print the unhandled instruction
        return getInvalidInstructionMapping();
    }
    return getInstructionMapping(DefaultMappingID, /*Cost=*/1,
                                 OperandsMapping, MI.getNumOperands());
}
```

The `MI.dump()` in the default case is a practical debugging aid — when you forget to add a mapping for a new instruction, the error message would otherwise be unhelpful. However, `dump()` is only available in debug builds, hence the `#if !defined(NDEBUG)` guard.

### Translating Generic Machine Instructions (M88kInstructionSelector)

The final step replaces generic instructions with real machine instructions:

```cpp
bool M88kInstructionSelector::select(MachineInstr &I) {
    if (selectImpl(I, *CoverageInfo))
        return true;
    return false;
}
```

`selectImpl()` is the generated method that applies patterns from the target description. The same patterns that work for the Selection DAG also work here because of a mapping from DAG node types to generic machine instructions (e.g., the DAG `and` node maps to `G_AND`).

If `selectImpl()` returns `false` (pattern didn't match), you'd add custom C++ code before returning `false` to handle special cases.

### Running GlobalISel

To use GlobalISel instead of the Selection DAG:

```bash
$ llc -mtriple m88k-openbsd -global-isel < and.ll
```

The output is identical to the Selection DAG version. To verify that GlobalISel is being used and to debug the pipeline, you can stop after specific passes:

```bash
$ llc -mtriple m88k-openbsd -global-isel < and.ll -stop-after=legalizer
```

This prints the machine IR after legalization, showing the generic instructions. The ability to inspect intermediate states between passes is a key advantage of GlobalISel's modular design.

---

## Part VIII: How to Further Evolve the Backend

The chapter concludes with a practical roadmap for developing the backend further:

1. **Choose your primary instruction selection method**: GlobalISel is easier to understand and develop; the Selection DAG is more mature and all existing LLVM targets implement it.

2. **Add integer arithmetic**: `add` and `sub` instructions follow the same pattern as `and`.

3. **Implement load/store instructions**: This is more complex because you need to handle different addressing modes (base+offset, base+index, etc.).

4. **Fully implement frame and call lowering**: With loads and stores available, you can create prologue/epilogue code and handle stack-based arguments. At this point, a "Hello, world!" program can be compiled.

5. **Implement branch instructions**: This enables loops and conditionals. For good code quality, you need to implement branch analysis methods in the instruction information class.

After these steps, the backend can translate simple algorithms, and you have enough experience to continue development based on your priorities.

---

## Final Synthesis: Connecting All the Pieces

Here's how everything fits together, from a 10,000-foot view:

1. **The Target Description** (`.td` files) is the foundation. It declaratively defines registers, register classes, instruction formats, instruction patterns, and calling conventions using TableGen. The `llvm-tblgen` tool generates C++ code from these descriptions.

2. **The Calling Convention** bridges the abstract world of LLVM IR (where arguments are named values) and the concrete world of machine code (where arguments live in specific physical registers or stack slots). It's defined once in the target description and reused by both Selection DAG and GlobalISel.

3. **Instruction Selection** (via either the Selection DAG or GlobalISel) is the core transformation. It takes LLVM IR and produces machine instructions. The Selection DAG does this through a monolithic graph-based approach; GlobalISel does it through a pipeline of smaller passes working on machine IR directly.

4. **Support Classes** (`RegisterInfo`, `InstrInfo`, `FrameLowering`) provide the infrastructure that instruction selection depends on — register allocation needs to know about reserved registers, instruction expansion needs the ability to construct new instructions, and the framework needs frame information even if it's empty.

5. **The Assembly Printer** is the final pass, lowering `MachineInstr` to `MCInst` and writing them to the output stream. It bridges the gap between the rich machine instruction representation used during compilation and the lean representation used for output.

6. **The Target Machine and Sub-Target** tie everything together. The target machine configures the pass pipeline, the data layout, and the object file format. The sub-target captures per-function hardware features and owns the key support classes.

The key insight is that LLVM's backend is designed as a **framework with hooks**. You implement specific classes, override specific methods, and provide specific data — but the framework handles the orchestration. The target description generates as much code as possible, and you only write C++ for what can't be expressed declaratively. This design makes it possible to create a working (if minimal) backend in a surprisingly small amount of code — as demonstrated by the fact that a single `and` instruction and a return can be translated with the code shown in this chapter.
