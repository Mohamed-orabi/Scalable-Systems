# Comprehensive Explanation: Optimizing IR (Chapter on LLVM Pass Manager, Pass Implementation, and Optimization Pipelines)

---

## Part 1: The LLVM Pass Manager — The Orchestrator of Optimization

### 1.1 What Is a Pass and Why Do We Need a Pass Manager?

When your compiler's frontend finishes its work, it hands off LLVM IR (Intermediate Representation) to the LLVM core libraries. These libraries have two enormous jobs: **optimize** the IR and then **convert** it into machine code (object code). This giant task is not done in one monolithic step — it's decomposed into many small, focused transformation steps called **passes**.

Think of it like an assembly line in a factory. Each station (pass) does one specific thing to the product (IR): one station removes dead code, another inlines functions, another simplifies expressions, and so on. The **pass manager** is the factory foreman — it decides which stations operate, in what order, and makes sure they share information efficiently.

**Why not just hard-code the order?** Because different situations demand different levels of optimization:

- During **development**, a programmer wants **fast compilation**. They don't care if the resulting binary is perfectly optimized — they just want to test quickly. So you might run very few passes.
- For a **release build**, the programmer wants the **fastest possible executable**, and they're willing to wait longer for the compiler to do its job. So you run many sophisticated passes, accepting longer compilation time.

This means the number and order of passes changes depending on the **optimization level** (like `-O0`, `-O1`, `-O2`, `-O3`, `-Os`, `-Oz`). Hard-coding the order would make this flexibility impossible.

### 1.2 Custom Passes — Why a Compiler Writer Needs Them

As a compiler writer, you may have **special knowledge about your source language** that LLVM's built-in passes don't have. For example, you might know that a certain well-known library function in your language can be replaced with inlined IR code, or even with a precomputed constant result. For the C language, LLVM already includes such a pass (it knows about `strlen`, `memcpy`, etc.), but for a custom language, you'd need to write this yourself.

Once you introduce your own pass, you may need to **reorder** or **add additional passes** around it. For instance, if your custom pass makes some code unreachable, you'd want to schedule the **dead code removal** pass right after yours to clean up. The pass manager handles all of this orchestration.

### 1.3 Pass Categories by Scope

Passes are categorized by the **scope of IR** they operate on. This is a hierarchy from broadest to narrowest:

**Module pass** — Takes an entire LLVM module as input. A module is essentially a single translation unit: all the functions, global variables, type definitions, etc. A module pass can see everything and perform **inter-procedural** analysis (looking across function boundaries). For example, it could decide to remove an unused global variable, or insert declarations of new functions across the module.

**Call graph pass** — Operates on the **strongly connected components (SCCs)** of the program's call graph. An SCC is a set of functions that are mutually recursive (they call each other in a cycle). The call graph pass traverses these components in **bottom-up order** — it processes callees before callers. This is critical for analyses like inlining decisions: you want to know what a callee looks like before deciding whether to inline it into its caller.

**Function pass** — Takes a **single function** as input and works only on that function. It cannot look at other functions or global state. Most optimization passes are function passes (constant folding, dead code elimination within a function, etc.).

**Loop pass** — Operates on a **single loop** within a function. This is the narrowest scope. Passes like loop unrolling, loop vectorization, and loop-invariant code motion work at this level.

The scope determines what the pass can see and modify, which directly affects what analysis information it needs and what it might invalidate.

### 1.4 Analysis Results — The Information Economy

Passes don't just transform IR — they often need **analysis information** to make smart decisions. For example, an optimization pass might need:

- **Alias analysis**: Do these two pointers point to the same memory? If not, we can reorder their accesses.
- **Dominator tree**: Which basic blocks dominate which others? This is essential for knowing where it's safe to move code.

This information is expensive to compute. The pass manager uses an **analysis manager** to handle this efficiently through a caching system:

1. A pass **requests** an analysis result from the analysis manager.
2. If the result is already **cached** (computed by a previous pass and still valid), the cached result is returned immediately — no recomputation needed.
3. If not cached, the analysis manager **computes** it on demand.

After a pass runs and **modifies** the IR, it must declare which analysis results are **preserved** (still valid). Any analysis not explicitly preserved gets **invalidated** in the cache, so subsequent passes that need it will trigger a recomputation. This is why passes return a `PreservedAnalyses` object.

### 1.5 What the Pass Manager Does Under the Hood

Two critical responsibilities:

**Sharing analysis results among passes.** The pass manager tracks which pass requires which analysis, and the current state (valid/invalid) of each cached analysis. The goal is twofold: avoid needless recomputation (don't run an analysis if no one needs it or if a valid result is already cached), and free memory held by analysis results as soon as they're no longer needed by any future pass.

**Pipeline execution for cache efficiency.** When multiple function passes need to run in sequence, the pass manager doesn't run all passes on all functions one pass at a time. Instead, it runs **all function passes on function #1**, then **all function passes on function #2**, and so on. This is a deliberate strategy to improve **CPU cache behavior**: the compiler is working on a small, localized chunk of data (one function's IR), performing all transformations on it while it's still hot in cache, before moving on. If it instead ran pass #1 on all functions, then pass #2 on all functions, it would repeatedly evict and reload different functions' IR from cache.

---

## Part 2: Implementing a New Pass — The `ppprofiler` Instrumentation Pass

### 2.1 The Goal: A Simple Profiler Through Instrumentation

The chapter demonstrates pass creation through a practical example: building a **basic profiler** called the "poor person's profiler" (`ppprofiler`). The idea is **instrumentation** — automatically inserting code into every function to collect performance data.

Specifically, the pass inserts:
- A call to `__ppp_enter()` at the **entry** of each function
- A call to `__ppp_exit()` before each **return instruction**

Both functions receive the **name of the current function** as a parameter. At runtime, these functions can count how many times each function is called and measure how long each invocation takes.

The pass is developed so it can work as a **standalone plugin** (loaded dynamically at runtime) or be **added directly to the LLVM source tree** (compiled into LLVM itself). Both approaches are shown.

### 2.2 Developing the Pass as a Plugin (Out-of-Tree)

The entire pass lives in a single file: `PPProfiler.cpp`.

**Step 1: Include files.** Six headers are needed:

```cpp
#include "llvm/ADT/Statistic.h"      // For the STATISTIC macros
#include "llvm/IR/Function.h"         // For the Function class
#include "llvm/IR/PassManager.h"      // For PassInfoMixin and pass infrastructure
#include "llvm/Passes/PassBuilder.h"  // For PassBuilder (pipeline construction)
#include "llvm/Passes/PassPlugin.h"   // For the plugin interface
#include "llvm/Support/Debug.h"       // For debug output infrastructure
```

Each header serves a specific purpose. `Statistic.h` gives us counters to track how many functions we instrument. `PassPlugin.h` is what makes our code loadable as a dynamic shared library.

**Step 2: Using the `llvm` namespace.** Simply `using namespace llvm;` to avoid writing `llvm::` everywhere. This is standard practice in LLVM pass code.

**Step 3: Defining the debug type.** LLVM has a built-in debug infrastructure. You define a string identifier with:

```cpp
#define DEBUG_TYPE "ppprofiler"
```

This string appears in printed statistics and debug output, so you can filter debug messages by pass name. When you see statistics output, it will say things like `1 ppprofiler - Number of instrumented functions`.

**Step 4: Defining a statistic counter.**

```cpp
ALWAYS_ENABLED_STATISTIC(NumOfFunc, "Number of instrumented functions.");
```

This creates a counter variable called `NumOfFunc`. Every time we instrument a function, we increment it. The text string is what gets printed in the statistics report.

> **Important note about the two macros:** There are two ways to define counters:
> - `STATISTIC` — the counter only works in debug builds (with assertions enabled) or if the CMake option `LLVM_FORCE_ENABLE_STATS` is set to `ON`. Statistics can be printed with the `--stats` command-line option.
> - `ALWAYS_ENABLED_STATISTIC` — the counter is **always** collected, even in release builds. However, the `--stats` flag only works with the first macro. If you use `ALWAYS_ENABLED_STATISTIC`, you'd need to explicitly call `llvm::PrintStatistics(llvm::raw_ostream)` to print the values.
>
> The chapter uses `ALWAYS_ENABLED_STATISTIC` so the count is always available.

**Step 5: Declaring the pass class.**

```cpp
namespace {
class PPProfilerIRPass
    : public llvm::PassInfoMixin<PPProfilerIRPass> {
public:
    llvm::PreservedAnalyses
    run(llvm::Module &M, llvm::ModuleAnalysisManager &AM);
private:
    void instrument(llvm::Function &F,
                    llvm::Function *EnterFn,
                    llvm::Function *ExitFn);
};
}
```

Several critical things to understand here:

The class is in an **anonymous namespace** (`namespace { ... }`). This is a C++ idiom that gives the class internal linkage — it's only visible within this translation unit. This avoids symbol conflicts if another file defines a class with the same name.

The class inherits from `PassInfoMixin<PPProfilerIRPass>`. This is CRTP (Curiously Recurring Template Pattern) — the class passes itself as a template argument. The `PassInfoMixin` template adds boilerplate code like a `name()` method that returns the pass's name. **Crucially, this template does NOT determine the type of the pass.** Whether this is a module pass, function pass, etc. is determined by the **signature of the `run()` method** and how the pass is registered.

The `run()` method takes a `Module &` and a `ModuleAnalysisManager &`, which makes this a **module pass**. If it took `Function &` and `FunctionAnalysisManager &`, it would be a function pass.

The private `instrument()` method is a helper that does the actual work of inserting calls into a single function.

### 2.3 The `instrument()` Method — How IR Is Modified

```cpp
void PPProfilerIRPass::instrument(llvm::Function &F,
                                   Function *EnterFn,
                                   Function *ExitFn) {
```

This receives three things: the function to instrument, and pointers to the `__ppp_enter` and `__ppp_exit` function declarations.

**Incrementing the counter:**

```cpp
++NumOfFunc;
```

Each call to `instrument()` means one more function is being instrumented, so we increment.

**Creating an IRBuilder:**

```cpp
IRBuilder<> Builder(&*F.getEntryBlock().begin());
```

The `IRBuilder` is LLVM's convenience class for inserting new IR instructions. You set it to a specific **insertion point**, and then every instruction you create through the builder gets inserted at that point.

Here, we set it to the **beginning of the entry block**. `F.getEntryBlock()` returns the first basic block of the function (the entry point), and `.begin()` gives an iterator to the first instruction in that block. The `&*` dereferences the iterator to get a pointer to the instruction.

**Creating a global string constant:**

```cpp
GlobalVariable *FnName = Builder.CreateGlobalString(F.getName());
```

This creates a new **global constant** in the module that holds the name of the function as a null-terminated C string. For example, if the function is called `main`, this creates a global like `@0 = private unnamed_addr constant [5 x i8] c"main\00"`. We need this string so we can pass the function name as an argument to `__ppp_enter` and `__ppp_exit`.

**Inserting the `__ppp_enter` call:**

```cpp
Builder.CreateCall(EnterFn->getFunctionType(), EnterFn, {FnName});
```

This inserts a `call void @__ppp_enter(ptr @0)` instruction right at the beginning of the function. The `{FnName}` is the argument — the pointer to the global string we just created.

**Inserting `__ppp_exit` calls before every return:**

```cpp
for (BasicBlock &BB : F) {
    for (Instruction &Inst : BB) {
        if (Inst.getOpcode() == Instruction::Ret) {
            Builder.SetInsertPoint(&Inst);
            Builder.CreateCall(ExitFn->getFunctionType(),
                               ExitFn, {FnName});
        }
    }
}
```

This is a nested loop: iterate over every basic block in the function, then over every instruction in each block. When we find a return instruction (`Instruction::Ret`), we move the builder's insertion point to **just before** that return (`SetInsertPoint(&Inst)` inserts before the given instruction), and insert a call to `__ppp_exit`.

Why loop over all basic blocks? A function can have **multiple return points**. Consider:

```c
int foo(int x) {
    if (x > 0) return 1;   // ret instruction #1
    return -1;              // ret instruction #2
}
```

We need to instrument **every** return path.

### 2.4 The `run()` Method — The Pass Entry Point

```cpp
PreservedAnalyses
PPProfilerIRPass::run(Module &M, ModuleAnalysisManager &AM) {
```

LLVM calls this method when it's time to execute the pass. It receives the module and an analysis manager (in case we need analysis results, though this simple pass doesn't).

**Guard against infinite recursion:**

```cpp
if (M.getFunction("__ppp_enter") ||
    M.getFunction("__ppp_exit")) {
    return PreservedAnalyses::all();
}
```

This is a critical safety check. If the module being compiled **already contains definitions** of `__ppp_enter` or `__ppp_exit` (meaning we're compiling the runtime support library itself), we must do **nothing**. Otherwise, we'd insert calls to `__ppp_enter` inside `__ppp_enter`, creating infinite recursion.

Returning `PreservedAnalyses::all()` tells the pass manager that no IR was modified, so all cached analyses remain valid.

**Declaring the runtime functions:**

```cpp
Type *VoidTy = Type::getVoidTy(M.getContext());
PointerType *PtrTy = PointerType::getUnqual(M.getContext());
FunctionType *EnterExitFty = FunctionType::get(VoidTy, {PtrTy}, false);
Function *EnterFn = Function::Create(
    EnterExitFty, GlobalValue::ExternalLinkage, "__ppp_enter", M);
Function *ExitFn = Function::Create(
    EnterExitFty, GlobalValue::ExternalLinkage, "__ppp_exit", M);
```

We're building the function type `void(ptr)` — a function that returns void and takes a single opaque pointer argument. Then we create two function **declarations** (not definitions — no body) with external linkage. This tells LLVM "these functions exist somewhere else (in the runtime library), and we'll link against them later."

Let's break this down line by line:

- `Type::getVoidTy(M.getContext())` — Gets the `void` type from the module's LLVM context.
- `PointerType::getUnqual(M.getContext())` — Gets an opaque (unqualified) pointer type. In modern LLVM (opaque pointers), all pointers are just `ptr` without a pointee type.
- `FunctionType::get(VoidTy, {PtrTy}, false)` — Creates a function type: returns `void`, takes one `ptr` parameter, and `false` means it's not variadic (no `...`).
- `Function::Create(...)` — Creates a function declaration with external linkage (the definition will come from another translation unit), the given name, and adds it to module `M`.

**Instrumenting all functions:**

```cpp
for (auto &F : M.functions()) {
    if (!F.isDeclaration() && F.hasName())
        instrument(F, EnterFn, ExitFn);
}
```

We iterate over all functions in the module. We skip:

- **Declarations** (functions with no body — just prototypes like `puts`, `printf`). There's nothing to instrument in a declaration.
- **Unnamed functions** (anonymous functions that don't have a meaningful name). Our approach relies on passing the function name as a string, so unnamed functions don't work.

**Returning the analysis preservation status:**

```cpp
return PreservedAnalyses::none();
```

We return `PreservedAnalyses::none()`, meaning we've invalidated **all** analyses. The text acknowledges this is "most likely too pessimistic" — in reality, some analyses might still be valid after our transformation. But being conservative (saying "nothing is preserved") is always safe. Being wrong the other way (claiming an analysis is preserved when it isn't) would cause incorrect optimization later.

### 2.5 Registering the Pass — Static vs. Dynamic Linking

Once the pass logic is implemented, it needs to be **registered** so the LLVM toolchain knows about it. Two mechanisms exist:

**Dynamic linking (plugin):** The pass is compiled into a shared library (`.so` file) and loaded at runtime. For this, the function `llvmGetPassPluginInfo()` must be provided — it's called when the plugin is loaded.

**Static linking:** The pass is compiled directly into LLVM tools like `opt` or `clang`. For this, a function `get<PluginName>PluginInfo()` must be provided.

Both return a `PassPluginLibraryInfo` struct containing: the API version, the plugin name, a version string, and a **pointer to a registration callback function**.

**The registration callback (`RegisterCB`):**

```cpp
void RegisterCB(PassBuilder &PB) {
    PB.registerPipelineParsingCallback(
        [](StringRef Name, ModulePassManager &MPM,
           ArrayRef<PassBuilder::PipelineElement>) {
            if (Name == "ppprofiler") {
                MPM.addPass(PPProfilerIRPass());
                return true;
            }
            return false;
        });
}
```

This registers a **pipeline parsing callback**. When the user specifies `--passes="ppprofiler"` on the command line, LLVM's `PassBuilder` parses that string and calls all registered callbacks to see if any of them recognize the name. Our callback checks if the name is `"ppprofiler"`. If so, it adds an instance of our pass to the module pass manager and returns `true` (meaning "I handled this name"). If the name doesn't match, it returns `false` (meaning "not my pass, ask someone else").

**The static-link function:**

```cpp
llvm::PassPluginLibraryInfo getPPProfilerPluginInfo() {
    return {LLVM_PLUGIN_API_VERSION, "PPProfiler", "v0.1", RegisterCB};
}
```

**The dynamic-link function:**

```cpp
#ifndef LLVM_PPPROFILER_LINK_INTO_TOOLS
extern "C" LLVM_ATTRIBUTE_WEAK ::llvm::PassPluginLibraryInfo
llvmGetPassPluginInfo() {
    return getPPProfilerPluginInfo();
}
#endif
```

The `#ifndef` guard is important: when the code is being **statically linked** into a tool, the macro `LLVM_PPPROFILER_LINK_INTO_TOOLS` is defined, and `llvmGetPassPluginInfo()` is **not compiled**. This prevents linker errors from multiple definitions of that symbol (since other plugins also define `llvmGetPassPluginInfo()`).

The `extern "C"` ensures no C++ name mangling. `LLVM_ATTRIBUTE_WEAK` makes the symbol a weak symbol so it doesn't cause conflicts during linking.

---

## Part 3: Adding the Pass to the LLVM Source Tree

There are two ways to integrate a pass into the LLVM source tree itself, rather than keeping it as an external plugin.

### 3.1 Approach A: Using the Plugin Mechanism Inside the Tree

This is the **simpler** approach — minimal changes to existing files.

**File placement:** Copy `PPProfiler.cpp` into a new directory: `llvm-project/llvm/lib/Transforms/PPProfiler/`.

**No source changes needed** to the `.cpp` file itself!

**Build integration:** Create `CMakeLists.txt` in the new directory:

```cmake
add_llvm_pass_plugin(PPProfiler PPProfiler.cpp)
```

And add to the parent directory's `CMakeLists.txt`:

```cmake
add_subdirectory(PPProfiler)
```

**Building:** When you run `ninja install`, CMake detects the new build file and outputs:

```
-- Registering PPProfiler as a pass plugin (static build: OFF)
```

By default, it builds as a **shared library** (`PPProfiler.so`) installed in the `lib` directory.

**Static linking option:** To statically link instead, rerun CMake with:

```
-DLLVM_PPPROFILER_LINK_INTO_TOOLS=ON
```

CMake will now report:

```
-- Registering PPProfiler as a pass plugin (static build: ON)
```

After building, three things change:

1. The pass is compiled into a static library (`libPPProfiler.a`)
2. Tools like `opt` are linked against this library
3. The plugin is registered as an extension in `Extension.def`, which now contains `HANDLE_EXTENSION(PPProfiler)`

This approach minimizes merge conflicts when syncing with the main LLVM repository because the new code lives in its own directory and only one existing file was modified.

### 3.2 Approach B: Full Integration into the Pass Registry

This is the approach used by LLVM's own built-in passes. It requires more changes but fully integrates the pass into the LLVM infrastructure.

**Step 1: Create a header file.** The class definition must be in a header file at `llvm-project/llvm/include/llvm/Transforms/PPProfiler/PPProfiler.h`.

The class is now in the `llvm` namespace (not an anonymous namespace), because it needs to be visible from other translation units:

```cpp
#ifndef LLVM_TRANSFORMS_PPPROFILER_PPPROFILER_H
#define LLVM_TRANSFORMS_PPPROFILER_PPPROFILER_H

#include "llvm/IR/PassManager.h"

namespace llvm {
class PPProfilerIRPass
    : public llvm::PassInfoMixin<PPProfilerIRPass> {
public:
    llvm::PreservedAnalyses
    run(llvm::Module &M, llvm::ModuleAnalysisManager &AM);
private:
    void instrument(llvm::Function &F,
                    llvm::Function *EnterFn,
                    llvm::Function *ExitFn);
};
} // namespace llvm

#endif
```

The header uses standard include guards (`#ifndef`/`#define`/`#endif`) to prevent multiple inclusion. The main reason this restructuring is necessary is that the **pass registry** needs to call the pass's constructor, which requires the class declaration to be accessible from a header file.

**Step 2: Update the source file.** Copy it to `llvm-project/llvm/lib/Transforms/PPProfiler/PPProfiler.cpp` and make two changes:

1. Remove the class definition (it's now in the header) and add `#include "llvm/Transforms/PPProfiler/PPProfiler.h"`
2. Remove `llvmGetPassPluginInfo()` — not needed since the pass won't be its own shared library

**Step 3: Create the build file.** Use `add_llvm_component_library` instead of `add_llvm_pass_plugin`:

```cmake
add_llvm_component_library(LLVMPPProfiler
    PPProfiler.cpp

    LINK_COMPONENTS
    Core
    Support
)
```

This declares the pass as a full LLVM **component**, which is a more formal unit of organization in LLVM's build system.

**Step 4: Add the subdirectory.** In the parent directory's `CMakeLists.txt`:

```cmake
add_subdirectory(PPProfiler)
```

**Step 5: Update the pass registry.** Edit `llvm/lib/Passes/PassRegistry.def` and add in the module passes section:

```
MODULE_PASS("ppprofiler", PPProfilerIRPass())
```

This `.def` file is a database of all available passes. The `MODULE_PASS` macro maps the string name `"ppprofiler"` to the constructor `PPProfilerIRPass()`. When `PassBuilder` parses a pipeline string containing `"ppprofiler"`, it looks up this database.

**Step 6: Update `PassBuilder.cpp`.** Add the include:

```cpp
#include "llvm/Transforms/PPProfiler/PPProfiler.h"
```

**Step 7: Add the link dependency.** In `llvm/lib/Passes/CMakeLists.txt`, under `LINK_COMPONENTS`, add `PPProfiler`.

After building and installing, the pass is now fully integrated: compiled into `libLLVMPPProfiler.a`, available as the `PPProfiler` component, and accessible by name `"ppprofiler"` to all LLVM tools.

---

## Part 4: Using the `ppprofiler` Pass with LLVM Tools

### 4.1 Running the Pass with `opt`

First, create a test IR file from a simple C program:

```c
#include <stdio.h>
int main(int argc, char *argv[]) {
    puts("Hello");
    return 0;
}
```

Compile to LLVM IR:

```bash
$ clang -S -emit-llvm -O1 hello.c
```

The `-S` flag says produce text output (not binary), `-emit-llvm` says produce LLVM IR instead of assembly, and `-O1` applies optimization level 1 to get clean, readable IR. The result is `hello.ll`.

The generated IR looks like:

```llvm
@.str = private unnamed_addr constant [6 x i8] c"Hello\00", align 1

define dso_local i32 @main(
        i32 noundef %0, ptr nocapture noundef readnone %1) {
    %3 = tail call i32 @puts(
            ptr noundef nonnull dereferenceable(1) @.str)
    ret i32 0
}
```

This is a very simple module with one global string constant and one function.

**Running the pass with `opt`:**

```bash
$ opt --load-pass-plugin=./PPProfiler.so \
      --passes="ppprofiler" --stats hello.ll -o hello_inst.bc
```

- `--load-pass-plugin` loads the shared library plugin
- `--passes="ppprofiler"` specifies which pass(es) to run
- `--stats` requests statistics output
- Output goes to `hello_inst.bc` (LLVM bitcode format)

If statistics are enabled, you see:

```
===--------------------------------------------------------===
                ... Statistics Collected ...
===--------------------------------------------------------===
1 ppprofiler - Number of instrumented functions.
```

If not enabled (release build without the force flag), you see:

```
Statistics are disabled.  Build with asserts or with
-DLLVM_FORCE_ENABLE_STATS
```

**Examining the instrumented IR:**

```bash
$ llvm-dis hello_inst.bc -o -
```

`llvm-dis` converts bitcode back to readable IR. The output now shows:

```llvm
@.str = private unnamed_addr constant [6 x i8] c"Hello\00", align 1
@0 = private unnamed_addr constant [5 x i8] c"main\00", align 1

define dso_local i32 @main(i32 noundef %0,
                            ptr nocapture noundef readnone %1) {
    call void @__ppp_enter(ptr @0)
    %3 = tail call i32 @puts(
            ptr noundef nonnull dereferenceable(1) @.str)
    call void @__ppp_exit(ptr @0)
    ret i32 0
}
```

Key observations:
- A new global string `@0` containing `"main\00"` was created
- A `call void @__ppp_enter(ptr @0)` was inserted at the function entry
- A `call void @__ppp_exit(ptr @0)` was inserted before the `ret` instruction

This confirms the pass is working correctly.

### 4.2 Specifying a Pass Pipeline

> **Important note:** The `--passes` option accepts not just single pass names but full **pipeline descriptions**. For example:
>
> ```
> --passes="ppprofiler,default<O2>"
> ```
>
> This runs the `ppprofiler` pass first, then the entire default O2 optimization pipeline. The pass names in a pipeline description must be of the **same type** (e.g., all module passes at the top level). The default pipeline for optimization level 2 is named `default<O2>`.

### 4.3 The Runtime Support Library

The instrumented code calls `__ppp_enter` and `__ppp_exit`, but these functions don't exist yet — we need to provide implementations. This is the **runtime support** part.

The `runtime.c` file implements these functions:

**File descriptor and cleanup:**

```c
static FILE *FileFD = NULL;

static void cleanup() {
    if (FileFD == NULL) {  // Note: This is actually a bug - should be != NULL
        fclose(FileFD);
        FileFD = NULL;
    }
}
```

A static file descriptor and an `atexit` cleanup function ensure the file is properly closed when the program terminates.

**Initialization:**

```c
static void init() {
    if (FileFD == NULL) {
        FileFD = fopen("ppprofile.csv", "w");
        atexit(&cleanup);
    }
}
```

Lazily opens the output file on first use and registers the cleanup function.

**Timestamp function:**

```c
typedef unsigned long long Time;

static Time get_time() {
    struct timespec ts;
    clock_gettime(CLOCK_PROCESS_CPUTIME_ID, &ts);
    return 1000000000L * ts.tv_sec + ts.tv_nsec;
}
```

Uses `clock_gettime` with `CLOCK_PROCESS_CPUTIME_ID` to get the CPU time consumed by the process, converted to nanoseconds. The text notes that not all systems support this clock — `CLOCK_REALTIME` is an alternative.

**The enter and exit functions:**

```c
void __ppp_enter(const char *FnName) {
    init();
    Time T = get_time();
    void *Frame = __builtin_frame_address(1);
    fprintf(FileFD,
            // "enter|name|clock|frame"
            "enter|%s|%llu|%p\n", FnName, T, Frame);
}

void __ppp_exit(const char *FnName) {
    init();
    Time T = get_time();
    void *Frame = __builtin_frame_address(1);
    fprintf(FileFD,
            // "exit|name|clock|frame"
            "exit|%s|%llu|%p\n", FnName, T, Frame);
}
```

Each function writes a pipe-delimited record: event type, function name, timestamp, and frame address. The `__builtin_frame_address(1)` gets the frame pointer of the **caller** (level 1, not level 0 which would be the current function). This helps distinguish between different active invocations of the same function (e.g., in recursion).

**Known limitations (all explicitly noted in the text):**

1. **Not thread-safe.** A single global file descriptor with no locking means multithreaded programs will produce corrupted output.
2. **No error checking on I/O.** If `fopen` or `fprintf` fails, data is silently lost.
3. **Imprecise timestamps.** The function call overhead plus the I/O within the instrumentation functions means the recorded times include the profiling overhead itself. The enter/exit events can be matched to compute a function's runtime, but that value includes the time spent doing I/O in the profiling functions. The text bluntly warns: "do not trust the times recorded here." However, the **call counts** should be accurate.

**Building and running the instrumented program:**

```bash
$ clang hello_inst.bc runtime.c
$ ./a.out
$ cat ppprofile.csv
enter|main|3300868|0x1
exit|main|3760638|0x1
```

The instrumented binary produces a CSV file showing that `main` was entered and exited once.

### 4.4 Plugging the Pass into `clang`

Using `opt` is useful for debugging a pass, but for real compilation, you want the pass to run automatically as part of the compilation pipeline.

**Extension points** are specific places in the default pass pipeline where custom passes can be inserted. The `PassBuilder` class provides registration methods for each extension point. Key extension points include:

- **Pipeline start**: Before any optimization passes run
- **Peephole**: After each instance of the instruction combiner
- Various others for loop optimization, vectorization, etc.

For `ppprofiler`, the **pipeline start** is ideal. Why? During optimization, functions may be **inlined** (their code is copied into their callers and the original function may be removed). If we instrument after inlining, we'd miss those inlined functions. By instrumenting before any optimization, we guarantee every function gets instrumented.

**Extending `RegisterCB()`:**

```cpp
PB.registerPipelineStartEPCallback(
    [](ModulePassManager &PM, OptimizationLevel Level) {
        PM.addPass(PPProfilerIRPass());
    });
```

This registers a lambda that adds the `PPProfilerIRPass` to the module pass manager whenever the default pipeline is being constructed. The callback is invoked at the pipeline start extension point, so our pass runs before any other optimization passes.

**Using with `clang`:**

```bash
$ clang -fpass-plugin=./PPProfiler.so hello.c runtime.c
```

The `-fpass-plugin` option tells `clang` to load the plugin. The `runtime.c` file is compiled alongside — and importantly, it is **not instrumented** because the pass detects that `__ppp_enter` and `__ppp_exit` are already defined in that module and skips it.

### 4.5 Scaling to Larger Programs

The chapter demonstrates scaling by suggesting you build the `tinylang` compiler itself with instrumentation. You pass CMake flags to add the plugin to all compiler invocations:

```
-DCMAKE_CXX_FLAGS="-fpass-plugin=<PluginPath>/PPProfiler.so"
-DCMAKE_EXE_LINKER_FLAGS="<RuntimePath>/runtime.o"
```

You also need to set the environment variables to ensure `clang` is the build compiler:

```bash
export CC=clang
export CXX=clang++
```

Running the instrumented `tinylang` on a test file (`Gcd.mod`) produces over **44,000 lines** in the CSV file.

### 4.6 Analyzing the Profiling Data with `awk`

Three `awk` scripts process the raw CSV data:

**`join.awk`** — Matches enter events with exit events by function name:

```awk
BEGIN { FS = "|"; OFS = "|" }
/enter/ { record[$2] = $0 }
/exit/ { split(record[$2],val,"|")
         print val[2], val[3], $3, $3-val[3], val[4] }
```

When it sees an `enter` event, it stores the entire line in an associative array keyed by function name (`$2`). When it sees an `exit` event, it looks up the stored enter event, splits it, and emits a joined record with the function name, the enter timestamp, the exit timestamp, the difference (elapsed time), and the frame address.

**`avg.awk`** — Aggregates the joined records:

```awk
BEGIN { FS = "|"; count[""] = 0; sum[""] = 0 }
{ count[$1]++; sum[$1] += $4 }
END { for (i in count) {
        if (i != "") {
            print count[i], sum[i], sum[i]/count[i], i }
} }
```

Counts calls per function and sums execution time. At the end, prints count, total time, average time, and function name for each.

**`demangle.awk`** — Passes mangled C++ names through `llvm-cxxfilt` to produce human-readable names:

```awk
{ cmd = "llvm-cxxfilt " $4
  (cmd) | getline name
  close(cmd); $4 = name; print }
```

For example, `_ZN8charinfo7isASCIIEc` becomes `charinfo::isASCII(char)`.

**The full pipeline:**

```bash
$ cat ppprofile.csv | awk -f join.awk | awk -f avg.awk | \
      sort -nr | head -15 | awk -f demangle.awk
```

Sample output:

```
446 1545581 3465.43 charinfo::isASCII(char)
409 826261 2020.2  llvm::StringRef::StringRef()
382 899471 2354.64 tinylang::Token::is(tinylang::tok::TokenKind) const
171 1561532 9131.77 charinfo::isIdentifierHead(char)
```

The first column is call count (reliable), the second is total nanoseconds (unreliable due to overhead), and the third is average time per call (also unreliable). As explained previously, do not trust the time values, though the call counts should be accurate.

---

## Part 5: Adding an Optimization Pipeline to Your Compiler

### 5.1 Architecture Overview

The `PassBuilder` class is the central orchestrator. It knows about all registered passes and can construct a pipeline either from a textual description (like `"default<O2>"`) or programmatically. It populates a `ModulePassManager` which then runs the pipeline.

Code generation passes (converting IR to machine code) still use the **legacy pass manager** — a separate, older system. So the compiler driver must manage two pass managers: the new one for IR optimization and the old one for code generation.

### 5.2 Implementation in the Compiler Driver

The implementation extends `tools/driver/Driver.cpp`:

**New includes:**

```cpp
#include "llvm/Passes/PassBuilder.h"
#include "llvm/Passes/PassPlugin.h"
#include "llvm/Analysis/TargetTransformInfo.h"
```

The `TargetTransformInfo` header provides a pass that bridges IR-level transformations with target-specific information (e.g., how expensive is a division on this CPU? How wide are the SIMD registers?).

**Command-line options (mirroring `opt`):**

```cpp
static cl::opt<bool>
    DebugPM("debug-pass-manager", cl::Hidden,
            cl::desc("Print PM debugging information"));
static cl::opt<std::string> PassPipeline(
    "passes",
    cl::desc("A description of the pass pipeline"));
static cl::list<std::string> PassPlugins(
    "load-pass-plugin",
    cl::desc("Load passes from plugin library"));
```

- `--debug-pass-manager` — Prints which passes run, in what order
- `--passes` — Specifies a custom pipeline as text
- `--load-pass-plugin` — Loads a pass plugin shared library

**Optimization levels:**

```cpp
static cl::opt<signed char> OptLevel(
    cl::desc("Setting the optimization level:"),
    cl::ZeroOrMore,
    cl::values(
        clEnumValN(3, "O", "Equivalent to -O3"),
        clEnumValN(0, "O0", "Optimization level 0"),
        clEnumValN(1, "O1", "Optimization level 1"),
        clEnumValN(2, "O2", "Optimization level 2"),
        clEnumValN(3, "O3", "Optimization level 3"),
        clEnumValN(-1, "Os",
                   "Like -O2 with extra optimizations for size"),
        clEnumValN(-2, "Oz",
                   "Like -Os but reduces code size further")),
    cl::init(0));
```

Six levels: O0 (no optimization), O1-O3 (increasing speed optimization), Os (optimize for size like O2 but with size reduction), Oz (aggressive size reduction). The default is O0. Note the use of `signed char` with negative values for size optimizations — a simple encoding trick.

**Static plugin registry:** LLVM generates an `Extension.def` file during configuration that lists all statically linked plugins. Using this file, function prototypes are generated:

```cpp
#define HANDLE_EXTENSION(Ext) \
    llvm::PassPluginLibraryInfo get##Ext##PluginInfo();
#include "llvm/Support/Extension.def"
```

The `##` is the C preprocessor token-pasting operator. If `PPProfiler` is registered, this generates `llvm::PassPluginLibraryInfo getPPProfilerPluginInfo();`.

### 5.3 The `emit()` Function — Setting Up and Running the Pipeline

**Creating the PassBuilder:**

```cpp
bool emit(StringRef Argv0, llvm::Module *M,
          llvm::TargetMachine *TM,
          StringRef InputFilename) {
    PassBuilder PB(TM);
```

The `TargetMachine *TM` is passed so the `PassBuilder` can access target-specific information when constructing passes.

**Loading dynamic plugins:**

```cpp
for (auto &PluginFN : PassPlugins) {
    auto PassPlugin = PassPlugin::Load(PluginFN);
    if (!PassPlugin) {
        WithColor::error(errs(), Argv0)
            << "Failed to load passes from '" << PluginFN
            << "'. Request ignored.\n";
        continue;
    }
    PassPlugin->registerPassBuilderCallbacks(PB);
}
```

Each plugin library is loaded, and its registration callback is invoked to register its passes with our `PassBuilder`. If loading fails, an error is printed but compilation continues (graceful degradation).

**Loading static plugins:**

```cpp
#define HANDLE_EXTENSION(Ext) \
    get##Ext##PluginInfo().RegisterPassBuilderCallbacks(PB);
#include "llvm/Support/Extension.def"
```

Same pattern but for statically linked plugins. The preprocessor expands this to call the registration function for each static plugin.

**Setting up analysis managers:**

```cpp
LoopAnalysisManager LAM(DebugPM);
FunctionAnalysisManager FAM(DebugPM);
CGSCCAnalysisManager CGAM(DebugPM);
ModuleAnalysisManager MAM(DebugPM);
```

Four analysis managers, one for each scope level. The debug flag enables verbose output about analysis computations.

**Populating and cross-registering:**

```cpp
FAM.registerPass(
    [&] { return PB.buildDefaultAAPipeline(); });
PB.registerModuleAnalyses(MAM);
PB.registerCGSCCAnalyses(CGAM);
PB.registerFunctionAnalyses(FAM);
PB.registerLoopAnalyses(LAM);
PB.crossRegisterProxies(LAM, FAM, CGAM, MAM);
```

The first line registers the **default alias analysis pipeline** with the function analysis manager. Alias analysis is so commonly needed that it gets special treatment.

The `register*Analyses` calls populate each manager with the default analysis passes and trigger any registration callbacks from plugins.

`crossRegisterProxies` is crucial: it sets up **proxy objects** so that, for example, a function pass can request a module-level analysis result. Without cross-registration, analysis managers at different scopes couldn't communicate.

**Constructing the module pass manager:**

```cpp
ModulePassManager MPM(DebugPM);
```

Two paths for populating it:

1. If `--passes` was specified, parse that description:

```cpp
if (!PassPipeline.empty()) {
    if (auto Err = PB.parsePassPipeline(MPM, PassPipeline)) {
        WithColor::error(errs(), Argv0)
            << toString(std::move(Err)) << "\n";
        return false;
    }
}
```

2. Otherwise, use the optimization level to select a default pipeline:

```cpp
else {
    StringRef DefaultPass;
    switch (OptLevel) {
        case 0: DefaultPass = "default<O0>"; break;
        case 1: DefaultPass = "default<O1>"; break;
        case 2: DefaultPass = "default<O2>"; break;
        case 3: DefaultPass = "default<O3>"; break;
        case -1: DefaultPass = "default<Os>"; break;
        case -2: DefaultPass = "default<Oz>"; break;
    }
    if (auto Err = PB.parsePassPipeline(MPM, DefaultPass)) {
        WithColor::error(errs(), Argv0)
            << toString(std::move(Err)) << "\n";
        return false;
    }
}
```

**Opening the output file:**

```cpp
std::error_code EC;
sys::fs::OpenFlags OpenFlags = sys::fs::OF_None;
CodeGenFileType FileType = codegen::getFileType();
if (FileType == CGFT_AssemblyFile)
    OpenFlags |= sys::fs::OF_Text;
auto Out = std::make_unique<llvm::ToolOutputFile>(
    outputFilename(InputFilename), EC, OpenFlags);
if (EC) {
    WithColor::error(errs(), Argv0) << EC.message() << '\n';
    return false;
}
```

The `OF_Text` flag is set for assembly and LLVM IR output, which are text-based formats. This matters on platforms like Windows where text and binary modes differ (line ending conversion).

**Code generation (legacy pass manager):**

```cpp
legacy::PassManager CodeGenPM;
CodeGenPM.add(createTargetTransformInfoWrapperPass(
    TM->getTargetIRAnalysis()));
```

The `TargetTransformInfoWrapperPass` makes target-specific cost information available to any IR passes that run in the code generation pipeline.

For LLVM IR output:

```cpp
if (FileType == CGFT_AssemblyFile && EmitLLVM) {
    CodeGenPM.add(createPrintModulePass(Out->os()));
}
```

For machine code (assembly or object):

```cpp
else {
    if (TM->addPassesToEmitFile(CodeGenPM, Out->os(),
                                 nullptr, FileType)) {
        WithColor::error() << "No support for file type\n";
        return false;
    }
}
```

`addPassesToEmitFile` returns `true` on error (if the target doesn't support the requested file type).

**Executing everything:**

```cpp
MPM.run(*M, MAM);      // Run optimization pipeline
CodeGenPM.run(*M);      // Run code generation
Out->keep();            // Don't delete the output file
return true;
```

The order matters: optimize first, then generate code. `Out->keep()` signals the `ToolOutputFile` to preserve the file (by default, it would be deleted on destruction, which is useful for cleanup when errors occur).

### 5.4 Build System Changes

The `CMakeLists.txt` must list all LLVM components needed:

```cmake
set(LLVM_LINK_COMPONENTS ${LLVM_TARGETS_TO_BUILD}
    AggressiveInstCombine Analysis AsmParser
    BitWriter CodeGen Core Coroutines IPO IRReader
    InstCombine Instrumentation MC ObjCARCOpts Remarks
    ScalarOpts Support Target TransformUtils Vectorize
    Passes)
```

Each name corresponds to an LLVM library. The component names roughly match the directory names where the LLVM source lives. During configuration, CMake translates these into the actual link library names.

The tool declaration:

```cmake
add_tinylang_tool(tinylang Driver.cpp SUPPORT_PLUGINS)
```

The `SUPPORT_PLUGINS` flag enables dynamic plugin loading support (sets up proper linker flags so plugins can access symbols from the main executable).

And linking against the project's own libraries:

```cmake
target_link_libraries(tinylang
    PRIVATE tinylangBasic tinylangCodeGen
    tinylangLexer tinylangParser tinylangSema)
```

Building with `$ ninja` will automatically detect the CMakeLists.txt changes and re-run CMake before compiling.

### 5.5 Extending the Pass Pipeline with Extension Points

Beyond just using `--passes` for a full custom pipeline, the extension point mechanism lets users **add passes at specific strategic locations** within the default pipeline.

During the construction of the pass pipeline, the pass builder allows user-contributed passes to be added at predefined places called **extension points**:

- The **pipeline start** extension point — adds passes at the beginning of the pipeline
- The **peephole** extension point — adds passes after each instance of the instruction combiner pass
- Other extension points exist for loop optimization, vectorization, etc.

The implementation adds a command-line option:

```cpp
static cl::opt<std::string> PipelineStartEPPipeline(
    "passes-ep-pipeline-start",
    cl::desc("Pipeline start extension point"));
```

And registers a callback:

```cpp
PB.registerPipelineStartEPCallback(
    [&PB, Argv0](ModulePassManager &PM) {
        if (auto Err = PB.parsePassPipeline(
                PM, PipelineStartEPPipeline)) {
            WithColor::error(errs(), Argv0)
                << "Could not parse pipeline "
                << PipelineStartEPPipeline.ArgStr << ": "
                << toString(std::move(Err)) << "\n";
        }
    });
```

This callback is invoked during pipeline construction. It parses the user-provided pipeline description and adds those passes at the pipeline start. This must be added after the call to `crossRegisterProxies()` but before the pipeline is created (before `parsePassPipeline()` is called).

> **Tip from the text:** To allow the user to add passes at every extension point, you need to add the preceding code snippet for each extension point.

### 5.6 Important Notes and Tips

**Alternative to `parsePassPipeline` for default pipelines:** Instead of parsing the string `"default<O2>"`, you can call `PB.buildPerModuleDefaultPipeline(OLevel)`. This method builds the default optimization pipeline programmatically for any level except O0.

**O0 level special handling:** At O0, no optimization passes are added, and extension point callbacks are **not invoked**. If you have passes that should run even at O0 (like `AlwaysInlinerPass` for functions with the `always_inline` attribute), you must add them manually:

```cpp
PassBuilder::OptimizationLevel OLevel = …;
if (OLevel == PassBuilder::OptimizationLevel::O0)
    MPM.addPass(AlwaysInlinerPass());
else
    MPM = PB.buildPerModuleDefaultPipeline(OLevel, DebugPM);
```

Of course, it is possible to add more than one pass to the pass manager in this fashion. `PassBuilder` also uses the `addPass()` method when constructing the pass pipeline.

> **Running extension point callbacks at O0:** Because the pass pipeline is not populated for optimization level O0, the registered extension points are not called. If you use the extension points to register passes that should also run at O0 level, this is problematic. You can call the `runRegisteredEPCallbacks()` method to run the registered extension point callbacks, resulting in a pass manager populated only with the passes that were registered through the extension points.

**Debugging tools:**

- `--debug-pass-manager` shows which passes execute and in which order
- `--print-before-all` / `--print-after-all` prints the full IR before/after each pass
- `--print-changed` prints the IR **only when it changed** compared to the previous pass — extremely useful for tracking which pass caused a specific transformation. The greatly reduced output makes it much easier to follow IR transformations.
- Insert `print` into a custom pipeline: `--passes="print,inline,print"` to see the IR before and after inlining

---

## Final Synthesis: The Big Picture

Everything in this chapter connects through a coherent architecture:

1. **The Pass Manager** is the central orchestration layer. It manages the execution order of passes, shares analysis results through caching, and optimizes for CPU cache behavior through its pipeline execution strategy.

2. **Passes** are modular, scoped transformations. They operate at module, call-graph, function, or loop level. They request analyses from analysis managers and declare what analyses they preserve after running.

3. **The `ppprofiler` pass** demonstrates the full lifecycle: implementing the pass logic (using `IRBuilder` to insert instrumentation calls), handling edge cases (infinite recursion guard, skipping declarations and unnamed functions), declaring analysis preservation, and registering the pass for both static and dynamic linking.

4. **Integration paths** range from external plugin (maximum isolation, minimum source changes) through in-tree plugin (moderate integration) to full registry integration (maximum integration, used by LLVM's own passes).

5. **The `PassBuilder`** ties everything together in a compiler: it creates analysis managers, cross-registers them, loads plugins, parses pipeline descriptions, provides extension points for strategic pass insertion, and handles optimization levels.

6. **The dual pass manager reality** — new pass manager for IR optimization, legacy pass manager for code generation — reflects LLVM's incremental migration and is something every compiler writer using LLVM must handle in their driver code.

The key takeaway is that LLVM's pass infrastructure is designed for **flexibility** (custom passes, multiple integration modes, extension points, user-specified pipelines) while maintaining **efficiency** (analysis caching, cache-friendly execution order, lazy computation). Understanding this architecture is essential for anyone building a compiler on top of LLVM or extending LLVM's optimization capabilities.
