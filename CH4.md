
# Chapter 4: Data Structures in Rust — A Comprehensive Explanation

---

## Preface: Why This Chapter Matters

Before Chapters 4 onward, the book focused on Rust's tooling. Now we shift entirely to the **language itself**. The author argues that data structures are *the most important part of Rust after its basic syntax*. This is not hyperbole — understanding how Rust models data is the gateway to understanding ownership, borrowing, lifetimes, and memory safety, because in Rust, **your types define how memory is managed**. That single insight is the thread that runs through the entire chapter.

The chapter covers: strings, vectors, maps, primitive types, structs, enums, aliases, error handling with `Result`, type conversion with `From`/`Into`, and FFI compatibility with C. Let's go through every piece.

---

## Section 1: Demystifying String, str, &str, and &'static str (§4.1)

### 1.1 The Core Confusion and How to Resolve It

If you've come to Rust from C#, Java, or Python, you're used to having **one** string type. Rust has **two core string types** — `String` and `str` — plus two common reference forms (`&str` and `&'static str`). The author's key insight is:

> Separate the **underlying data** (a contiguous sequence of UTF-8 characters) from the **interface** you use to interact with it. There is only one kind of string in Rust, but multiple ways to handle its allocation and references.

Both `String` and `str` represent a UTF-8 sequence of characters stored contiguously in memory. The **only practical difference** is how memory is managed.

### 1.2 String vs str — The Memory Model

- **`str`** — A **stack-allocated** UTF-8 string. It can be **borrowed** but cannot be **moved** or **mutated**. (Important nuance: `&str` *can* point to heap-allocated data — more on this below.)
- **`String`** — A **heap-allocated** UTF-8 string. It can be both **borrowed** and **mutated**.

In Rust, memory allocation is **explicit** — the type itself usually tells you how memory is allocated and how many elements it contains. This is a massive departure from C, where a `char*` gives you zero information about whether the memory is on the stack or heap.

### 1.3 The C Analogy

The author draws a direct comparison to C to ground this:

```c
char *stack_string = "stack-allocated string";
char *heap_string = strndup("heap-allocated string");
```

In C, both are `char*` — the type is identical. You have no type-level signal about where the memory lives. The first is a pointer to a string literal in the binary's read-only data segment (stack pointer). The second calls `strndup()`, which internally calls `malloc()` to allocate heap memory, copies the input, and returns the address.

**Pedantic note the author includes:** The literal `"heap-allocated string"` in the second line is *initially* stack-allocated (it exists in the binary). `strndup()` copies it to the heap. You could verify this by examining the compiled binary, which would contain that literal string.

In C, all strings are "just" contiguous memory terminated by `0x00` (null). There is no type-level distinction. Rust's `str` maps conceptually to `stack_string`, and `String` maps to `heap_string`. The author calls this an "oversimplification" but a "good model."

### 1.4 Using Strings Effectively (§4.1.2)

In practice, you almost always work with **`String`** or **`&str`**, never a bare `str`. Why?

- It's **not possible to create a `str` directly** — you can only borrow a reference to one.
- The standard library's **immutable** string functions are implemented on `&str`.
- The **mutable** string functions are only on `String`.
- `&str` is the "lowest common denominator" for function arguments because you can always borrow a `String` as `&str`.

This means: if you're writing a function that reads a string but doesn't modify it, accept `&str`. This way callers can pass either a `String` (borrowed as `&str`) or a string literal (which is already `&str`).

### 1.5 Static Lifetimes

`'static` is a special lifetime specifier meaning a reference is valid for the **entire life of the process**. A `&'static str` is a reference to a `str` that lives as long as the program runs — typically string literals baked into the binary.

Key distinction: While a `String` can be borrowed as `&str`, a `String` can **never** be borrowed as `&'static str`. Why? Because a `String` has a finite lifetime — when it goes out of scope, Rust calls its `Drop` trait to deallocate. Its lifetime is inherently shorter than `'static`.

### 1.6 The Decision Flowchart

The author provides a simple decision tree:

1. Do you need a UTF-8 string? (Yes.)
2. Do you need it to be **mutable**?
   - **No** → Use `let my_string = "my string";` (this gives you a `&'static str`)
   - **Yes** → Use `let my_string = String::new("my string");`

### 1.7 Under the Hood

- A `String` is actually **a `Vec<u8>`** — a vector of UTF-8 bytes. (Vectors are covered later in the chapter.)
- A `str` is **a slice** of UTF-8 characters. (Slices are covered in the next section.)

### 1.8 Summary Table

| Type | Kind | Components | Use |
|------|------|-----------|-----|
| `str` | Stack-allocated UTF-8 string slice | Pointer to array of chars + length | Immutable strings like logging/debug statements |
| `String` | Heap-allocated UTF-8 string | A vector of characters | Mutable, resizable strings |
| `&str` | Immutable string reference | Pointer to borrowed `str` or `String` + length | Anywhere you want to borrow a string immutably |
| `&'static str` | Immutable static string reference | Pointer to a `str` + length | A reference with explicit static lifetime |

### 1.9 Movability — A Critical Distinction

`String` can be **moved**; `str` **cannot**. You cannot own a variable of type `str` — you can only hold a reference to one.

The code example (Listing 4.1) demonstrates this perfectly:

```rust
fn print_String(s: String) { println!("print_String: {}", s); }
fn print_str(s: &str)      { println!("print_str: {}", s); }

fn main() {
    // let s: str = "impossible str";          // DOES NOT COMPILE — size unknown at compile time
    print_String(String::from("String"));      // OK: moves String into function
    print_str(&String::from("String"));        // OK: borrows String as &str
    print_str("str");                          // OK: string literal is already &str
    // print_String("str");                    // DOES NOT COMPILE — type mismatch: expected String, got &str
}
```

Walking through each line:

- `let s: str = ...` fails because `str` is unsized — the compiler cannot determine how much stack space to allocate.
- `print_String(String::from("String"))` works because we create a heap-allocated `String` and **move** ownership into the function.
- `print_str(&String::from("String"))` works because we create a `String` and then **borrow** it as `&str` — this is the `Deref` coercion at work.
- `print_str("str")` works because a string literal is already `&'static str`, which satisfies `&str`.
- `print_String("str")` fails because the function expects an owned `String`, but we're giving it a `&str`. These are different types.

---

## Section 2: Understanding Slices and Arrays (§4.2)

### 2.1 What Are Slices?

Slices and arrays represent a **sequence of values of the same type**. They can be multidimensional (slices of slices, arrays of arrays, etc.).

The term "slice" is relatively modern. In Java, C, C++, Python, Ruby — you typically use "array" or "list." Slices as a **first-class language concept** are a feature of Rust and Go. C++ has `std::span` and `std::string_view` which provide equivalent behavior, but the C++ community doesn't use the word "slice."

**Historical note the author includes:** The term "slices" appears to have originated with Go, as described in a 2013 blog post by Rob Pike.

### 2.2 The Key Difference: Array vs Slice

- **Array**: A **fixed-length** sequence. The length is known at **compile time** and is part of the type signature (e.g., `[u8; 64]`).
- **Slice**: A **variable-length** sequence. The length is determined at **run time**. A slice is a "view" into contiguous memory.

Additionally, slices can be **destructured** into nonoverlapping subslices — useful for divide-and-conquer or recursive algorithms.

### 2.3 Arrays Can Be Tricky

Because array length must be known at compile time, it must be embedded in the type signature. As of Rust 1.51, **const generics** allow defining generic arrays of arbitrary (but still compile-time-known) length. This is covered in detail in chapter 10.

### 2.4 Code Walkthrough (Listing 4.2)

```rust
let array = [0u8; 64];          // Type: [u8; 64] — 64 bytes, all zero
let slice: &[u8] = &array;      // Borrow the array as a slice
```

`0u8` is shorthand: the value `0` with type `u8` (unsigned 8-bit integer). The `;` syntax `[value; count]` initializes an array with `count` copies of `value`.

On the second line, we borrow the array as a slice. The slice type `&[u8]` no longer carries the length in the type — the length is a runtime value stored alongside the pointer (a "fat pointer").

### 2.5 Destructuring Slices — Borrowing Multiple Times

```rust
let (first_half, second_half) = slice.split_at(32);
```

`split_at()` is a core library function that works on all slices, arrays, and vectors. It returns two **nonoverlapping** subslices. This is critical in Rust because:

- You can borrow the same array or slice **multiple times** using this pattern.
- Subslices don't overlap, so there's no aliasing violation.
- Common use case: parsing or decoding text/binary data.

The string parsing example:

```rust
let wordlist = "one,two,three,four";
for word in wordlist.split(',') {
    println!("word={}", word);
}
```

Here, we split a string on commas and iterate over the resulting subslices. **No heap allocation occurs.** All memory is stack-allocated, with fixed length known at compile time, and no `malloc()` under the hood. This is equivalent to working with raw C pointers in performance — no reference counting, no garbage collection — but the code is safe, succinct, and not verbose.

### 2.6 Optimized Slice Operations (Listing 4.3)

The standard library's `copy_from_slice()` uses `memcpy()` internally:

```rust
pub fn copy_from_slice(&mut self, src: &[T]) where T: Copy {
    unsafe {
        ptr::copy_nonoverlapping(src.as_ptr(), self.as_mut_ptr(), self.len());
    }
}
```

`ptr::copy_nonoverlapping()` is a wrapper around C's `memcpy()`. On some platforms, `memcpy()` has hardware-level optimizations (SIMD, etc.) beyond what normal Rust code can achieve. Similarly, `fill()` and `fill_with()` use `memset()`.

### 2.7 Summary of Arrays and Slices

- Arrays are fixed-length, compile-time-known.
- Slices are pointers to contiguous memory + a length, representing arbitrary-length sequences.
- Both can be recursively destructured into nonoverlapping subslices.

---

## Section 3: Vectors (§4.3)

### 3.1 Why Vectors Matter

`Vec` is **arguably Rust's most important data type**. (`String` is the next most important — and it's based on `Vec`.) Whenever you need a **resizable sequence of values**, you reach for `Vec`.

If you're coming from C++, Rust's `Vec` is conceptually very similar to `std::vector`. Vectors are a general-purpose container.

Vectors are one of the ways to allocate heap memory in Rust (another being smart pointers like `Box`, covered in chapter 5). They have internal optimizations to limit excessive allocations, such as allocating memory in blocks. In nightly Rust, you can supply a custom allocator.

### 3.2 Diving Deeper into Vec (§4.3.1)

**Vec inherits the methods of slices.** This is possible because you can obtain a slice reference from a vector. Rust doesn't have OOP inheritance, but `Vec` is a "special type that is both a `Vec` and a slice at the same time."

Here's how `as_slice()` works:

```rust
pub fn as_slice(&self) -> &[T] {
    self  // This looks like it shouldn't work — but it does!
}
```

This takes `self` (which is `Vec<T>`) and returns it as `&[T]` (a slice). If you tried to write the same code yourself, it would fail. So how does it work?

**Answer: The `Deref` trait.**

Rust provides `Deref` (and `DerefMut`), which the compiler uses to **coerce** one type into another implicitly. Once implemented for a type, that type automatically gets all the methods of the dereferenced type.

The standard library implementation (Listing 4.5):

```rust
impl<T, A: Allocator> ops::Deref for Vec<T, A> {
    type Target = [T];
    fn deref(&self) -> &[T] {
        unsafe { slice::from_raw_parts(self.as_ptr(), self.len) }
    }
}

impl<T, A: Allocator> ops::DerefMut for Vec<T, A> {
    fn deref_mut(&mut self) -> &mut [T] {
        unsafe { slice::from_raw_parts_mut(self.as_mut_ptr(), self.len) }
    }
}
```

Dereferencing the vector coerces it into a slice from its raw pointer and length. This operation is **temporary** — a slice can't be resized, and its length is fixed at the time of dereferencing.

### 3.3 The Borrow Checker Protects You

If you took a slice of a vector and then tried to resize the vector, the slice's size wouldn't change — and you'd have a dangling reference. But **this is only possible in unsafe code**. The borrow checker prevents it:

```rust
let mut vec = vec![1, 2, 3];
let slice = vec.as_slice();   // immutable borrow
vec.resize(10, 0);            // mutable borrow — FAILS
println!("{}", slice[0]);     // immutable borrow used again
```

Error: "cannot borrow `vec` as mutable because it is also borrowed as immutable." This is Rust's ownership system at work — the same pattern you've been implementing as a runtime-enforced library in C#, but here it's a compile-time guarantee.

### 3.4 Wrapping Vectors (§4.3.2)

Some types wrap a `Vec` internally. The most important example:

```rust
pub struct String {
    vec: Vec<u8>,
}
```

`String` is literally a `Vec<u8>` that dereferences (via `Deref`) into `str`. This wrapping pattern is common — `Vec` is the preferred way to implement a resizable sequence of any type.

### 3.5 Types Related to Vectors (§4.3.3)

The author's advice: **90% of the time, use `Vec`. 10% of the time, use `HashMap`.** Other container types exist but rarely provide meaningful performance improvements for typical cases.

The author includes the famous Donald Knuth quote about premature optimization (the full quote about programmers wasting enormous time on noncritical optimizations, concluding that premature optimization is the root of all evil, while noting we shouldn't pass up opportunities in the critical 3%).

**When memory is a concern:** If you're worried about allocating excessively large contiguous memory blocks, use `Vec<Box<T>>` — each element is a pointer to a heap-allocated value, so the vector itself is just a sequence of pointers.

**Other collection types in the standard library:**

| Type | Description |
|------|-------------|
| `VecDeque` | Double-ended queue (resizable, based on `Vec`) |
| `LinkedList` | Doubly linked list |
| `HashMap` | Hash map (discussed next) |
| `BTreeMap` | Map based on a B-tree |
| `HashSet` | Hash set (based on `HashMap`) |
| `BTreeSet` | B-tree set (based on `BTreeMap`) |
| `BinaryHeap` | Priority queue (binary heap, uses `Vec` internally) |

**Tip from the author:** You can build your own data structures on top of `Vec`. The `BinaryHeap` source code from the standard library is a complete example of how to do this.

---

## Section 4: Maps (§4.4)

### 4.1 HashMap Basics

`HashMap` is the preferred type for **constant-time lookups by key**. Rust's implementation is likely faster and safer than what you'd find in other libraries.

`HashMap` uses the **SipHash-1-3** hashing function, also used in Python (≥3.4), Ruby, Swift, and Haskell. SipHash provides good tradeoffs for common cases but may be inappropriate for very small keys (like single integers) or very large keys (like long strings).

You **can supply your own hash function**.

### 4.2 Custom Hashing Functions (§4.4.1)

To use a custom hash function, it must implement three traits:
- `std::hash::BuildHasher`
- `std::hash::Hasher`
- `std::default::Default`

Looking at HashMap's generic parameters (Listing 4.7):

```rust
impl<K, V, S> HashMap<K, V, S>
where
    K: Eq + Hash,       // Keys must be equatable and hashable
    S: BuildHasher,     // The hash strategy must implement BuildHasher
{ }
```

`BuildHasher` is a factory trait — it just creates `Hasher` instances (Listing 4.8):

```rust
pub trait BuildHasher {
    type Hasher: Hasher;
    // requires build_hasher() -> Self::Hasher
}
```

The `Hasher` trait requires only two methods:
- `write(&[u8])` — feed bytes to be hashed
- `finish() -> u64` — return the computed hash

The `Hasher` trait also provides blanket implementations (default implementations you get for free).

**Practical example (Listing 4.9):** Using MetroHash (an alternative to SipHash by J. Andrew Rogers):

```rust
use metrohash::MetroBuildHasher;
use std::collections::HashMap;

let mut map = HashMap::<String, String, MetroBuildHasher>::default();
map.insert("hello?".into(), "Hello!".into());
println!("{:?}", map.get("hello?"));  // prints Some("Hello!")
```

The MetroHash crate already implements `BuildHasher` and `Hasher`, so this "just works." The `.into()` calls use the `Into` trait to convert `&str` to `String`.

### 4.3 Creating Hashable Types (§4.4.2)

HashMap keys must implement `Eq` and `Hash`. Both can be **automatically derived**:

```rust
#[derive(Hash, Eq, PartialEq, Debug)]
struct CompoundKey {
    name: String,
    value: i32,
}
```

Why four derives?
- `Hash` — needed for hashing the key
- `Eq` — needed for key equality comparison
- `PartialEq` — required because `Eq` **depends on** `PartialEq` (you can't have `Eq` without `PartialEq`)
- `Debug` — provides debug printing, extremely convenient for development

**Important property of `#[derive]`:** It's **composable**. As long as all *member types* of your struct implement the traits, you can derive them for the struct. Since both `String` and `i32` implement `Hash`, `Eq`, `PartialEq`, and `Debug`, the compiler can automatically generate these implementations for `CompoundKey`.

---

## Section 5: Rust Types — Primitives, Structs, Enums, and Aliases (§4.5)

Rust is strongly typed and provides four categories of types:

1. **Primitives** — strings, arrays, tuples, integral types
2. **Structs** — compound types composed of arbitrary combinations of other types
3. **Enums** — mutually exclusive variant types (far more powerful than C/C++/Java enums)
4. **Aliases** — syntax sugar for creating new type names for existing types

### 5.1 Primitive Types (§4.5.1)

**Summary table:**

| Class | Kind | Description |
|-------|------|-------------|
| Scalar | Integers | Signed (`i`) or unsigned (`u`), 8–128 bits |
| Scalar | Sizes | Architecture-specific (`usize`, `isize`) |
| Scalar | Floating point | `f32` or `f64` |
| Compound | Tuples | Fixed-length collection of mixed types |
| Sequence | Arrays | Fixed-length sequence of a single type |

#### Integer Types

| Length | Signed | Unsigned | C Equivalent |
|--------|--------|----------|-------------|
| 8-bit | `i8` | `u8` | `char` / `unsigned char` |
| 16-bit | `i16` | `u16` | `short` / `unsigned short` |
| 32-bit | `i32` | `u32` | `int` / `unsigned int` |
| 64-bit | `i64` | `u64` | `long` / `long long` (platform-dependent) |
| 128-bit | `i128` | `u128` | Nonstandard (`__int128` in GCC/Clang) |

**Integer literal syntax:** The type can be appended directly — `0u8` means unsigned 8-bit with value 0. Prefixes for different bases: `0b` (binary), `0o` (octal), `0x` (hexadecimal), `b` (byte literal).

The code example (Listing 4.11) demonstrates all of these:

```rust
let value = 0u8;       // 0 as u8, size = 1 byte
let value = 0b1u16;    // binary 1 as u16, size = 2 bytes
let value = 0o2u32;    // octal 2 as u32, size = 4 bytes
let value = 0x3u64;    // hex 3 as u64, size = 8 bytes
let value = 4u128;     // 4 as u128, size = 16 bytes
```

`std::mem::size_of_val(&value)` returns the size in bytes. The output confirms: u8=1 byte, u16=2 bytes, u32=4 bytes, u64=8 bytes, u128=16 bytes.

**Underscore separators** for readability: `0b1111_1111` is the same as `0b11111111`. The output confirms: binary `0b1111_1111` = 255, octal `0o1111_1111` = 2,396,745, decimal `1111_1111` = 11,111,111, hex `0x1111_1111` = 286,331,153, byte literal `b'A'` = 65.

#### Size Types

`usize` and `isize` are platform-dependent — typically 32 bits on 32-bit systems, 64 bits on 64-bit systems. `usize` is equivalent to C's `size_t`. In the standard library, any function expecting or returning a length uses `usize`.

#### Arithmetic on Primitives — Checked by Default

This is a major contrast with C/C++. The author demonstrates with a C division-by-zero example:

```c
printf("%d\n", 1 / 0);  // Compiles! Prints a "random" value. Undefined behavior.
```

Both GCC and Clang compile this (with a warning). There's no runtime check.

In Rust, **all arithmetic is checked by default**. The author shows four attempts to divide by zero, and the compiler catches the first three at compile time:

```rust
// println!("{}", 1 / 0);           // Caught at compile time
// println!("{}", one / zero);       // Caught at compile time (constants folded)
// println!("{}", one / zero);       // Still caught (compiler traces constant values)
let one = { || 1 }();               // Trick: closure return value — compiler can't fold
let zero = { || 0 }();
println!("{}", one / zero);          // PANICS at runtime
```

The compiler is remarkably smart at catching constant expressions. You have to "trick" it with closures (or regular functions) to reach runtime. When you do, Rust **panics** — a controlled crash with a backtrace, not undefined behavior.

**Controlled arithmetic methods:**

For cases where you need finer control:

- `checked_div()` — returns `Option<T>`: `Some(result)` on success, `None` on division by zero
- **Wrapping** forms (`wrapping_add`, `wrapping_sub`, etc.) — perform modular arithmetic (C-compatible behavior for unsigned types; note: signed overflow in C is technically undefined)
- **Overflowing** forms — return `(result, bool)` indicating whether overflow occurred
- **Unchecked** forms — skip the check entirely (unsafe)

Examples:
```rust
assert_eq!((100i32).checked_div(1i32), Some(100i32));
assert_eq!((100i32).checked_div(0i32), None);

assert_eq!(0xffu8.wrapping_add(1), 0);            // 255 + 1 wraps to 0
assert_eq!(0xffffffffu32.wrapping_add(1), 0);     // u32::MAX + 1 wraps to 0
assert_eq!(0u32.wrapping_sub(1), 0xffffffff);     // 0 - 1 wraps to u32::MAX
assert_eq!(0x80000000u32.wrapping_mul(2), 0);     // Overflow wraps
```

### 5.2 Using Tuples (§4.5.2)

Tuples are **fixed-length sequences where each element can have a different type**. They are **not reflective** — unlike arrays, you can't iterate over them, slice them, or determine component types at runtime. They are essentially syntax sugar.

```rust
let tuple = (1, 2, 3);
println!("tuple = ({}, {}, {})", tuple.0, tuple.1, tuple.2);  // Access by position
```

**Destructuring with match:**
```rust
match tuple {
    (one, two, three) => println!("{}, {}, {}", one, two, three),
}
```

**Destructuring with let:**
```rust
let (one, two, three) = tuple;  // Moves values out of the tuple
```

**Most common use: returning multiple values from a function:**
```rust
fn swap<A, B>(a: A, b: B) -> (B, A) {
    (b, a)
}
// swap(1, 2) returns (2, 1)
```

**Practical limit:** Don't use tuples with more than 12 elements. There's no strict compiler limit, but the standard library only provides trait implementations for tuples up to 12 elements. Beyond 12, you'll lose access to derived traits like `Debug`, `Clone`, etc.

### 5.3 Using Structs (§4.5.3)

Structs are Rust's **primary building block** — composite data types containing any combination of types and values. They're similar to C structs or classes in OOP languages. They support generics (like C++ templates or Java/C# generics).

**When to use a struct:**
- You need **stateful functions** (methods operating on internal state)
- You need to **control access** to internal state (private variables)
- You need to **encapsulate state behind an API**

**You don't have to use structs.** You can write pure function APIs (C-style). Structs define **implementations**, not **interfaces** (unlike OOP languages where classes define both).

#### Three Forms of Structs

**1. Empty (unit) structs:**
```rust
struct EmptyStruct {}
struct AnotherEmptyStruct;  // Semicolon variant
```

**2. Tuple structs:**
```rust
struct TupleStruct(String);
let ts = TupleStruct("string value".into());
println!("{}", ts.0);  // Access by position, like tuples
```

Tuple structs have unnamed fields (types only, no names). They use a semicolon at the end. They save characters but create ambiguity.

**3. Regular structs:**
```rust
struct TypicalStruct {
    name: String,
    value: String,
    number: i32,
}
```

#### Visibility

Elements have **module visibility** by default (accessible anywhere in the current module, equivalent to `pub(self)`). Visibility is per-element:

```rust
pub struct MixedVisibilityStruct {
    pub name: String,           // Public outside crate
    pub(crate) value: String,   // Public within crate only
    pub(super) number: i32,     // Public within parent scope
}
```

For a struct to be usable outside its crate (as a library), it must be declared `pub struct`. This applies equally to functions, traits, and all other declarations.

#### Deriving Standard Traits

```rust
#[derive(Debug, Clone, Default)]
struct DebuggableStruct {
    string: String,
    number: i32,
}
```

- **`Debug`** — Provides `fmt()` for debug printing (`{:?}` format specifier)
- **`Clone`** — Provides `clone()` to create a copy
- **`Default`** — Provides `default()` to create a default (usually empty) instance

As long as all elements implement the trait, you can derive automatically. With these derives:

```rust
let ds = DebuggableStruct::default();
println!("{:?}", ds);          // Prints: DebuggableStruct { string: "", number: 0 }
println!("{:?}", ds.clone());  // Same output
```

#### Implementing Methods

```rust
impl DebuggableStruct {
    fn increment_number(&mut self) {   // Borrows mutably
        self.number += 1;
    }
}
```

An alternative that **consumes** the struct:

```rust
impl DebuggableStruct {
    fn incremented_number(mut self) -> Self {  // Takes ownership
        self.number += 1;
        self
    }
}
```

The difference is subtle but important: `&mut self` borrows (the caller keeps ownership), while `mut self` **moves** (the caller loses ownership; useful for "builder pattern" or when you want to prevent further use of the original).

### 5.4 Using Enums (§4.5.4)

Rust's enums are **fundamentally different** from enums in C, C++, Java, or C#. In those languages, enums are essentially named integer constants. In Rust, enums are **discriminated unions** — they can hold any type, and only one variant is active at a time.

The author makes the comparison: Rust's enums are more similar to C++'s `std::variant` than to C++'s `enum`.

**Key distinction from structs:** With a struct, **all elements are present simultaneously**. With an enum, **only one variant is present at any given time**. They are mutually exclusive.

#### Simple (C-like) Enums

```rust
#[derive(Debug)]
enum JapaneseDogBreeds {
    AkitaKen, HokkaidoInu, KaiKen, KishuInu, ShibaInu, ShikokuKen,
}

println!("{:?}", JapaneseDogBreeds::ShibaInu);         // Prints: ShibaInu
println!("{:?}", JapaneseDogBreeds::ShibaInu as u32);  // Prints: 4 (enumerated value)
```

Enum variants are auto-enumerated starting from 0. You can cast to an integer with `as u32`.

#### Converting from Integer to Enum

There's **no automatic conversion** from integer back to enum. You must implement it yourself, typically with the `From` trait:

```rust
impl From<u32> for JapaneseDogBreeds {
    fn from(other: u32) -> Self {
        match other {
            other if JapaneseDogBreeds::AkitaKen as u32 == other => JapaneseDogBreeds::AkitaKen,
            // ... each variant ...
            _ => panic!("Unknown breed!"),
        }
    }
}
```

This uses **match guards** — the `if` clause after the pattern. Each arm casts the enum variant to `u32` for comparison. The `_` wildcard panics on unknown values.

#### Explicit Discriminants

You can specify the integer values:
```rust
enum Numbers { One = 1, Two = 2, Three = 3 }
println!("one={}", Numbers::One as u32);  // Prints: one=1
```

Note: Without the `as u32` cast, this doesn't compile — enum variants don't implement `std::fmt::Display` by default.

#### Rich Enum Variants

Enums can contain tuples, structs, and anonymous types:

```rust
enum EnumTypes {
    NamedType,                        // Named unit-like type
    String,                           // Unnamed String type
    NamedString(String),              // Named, with tuple syntax (one item)
    StructLike { name: String },      // Struct-like variant
    TupleLike(String, i32),           // Tuple-like with two elements
}
```

- **Named variants** = new types within the enum, also corresponding to enumerated integer values.
- **Unnamed variants** = specified as a type, not with a name.

**Best practice:** Avoid mixing named and unnamed variants — it's confusing.

### 5.5 Using Aliases (§4.5.5)

Aliases provide an **alternative name** for an existing type. Equivalent to C/C++'s `typedef` or C++'s `using`. **An alias does not create a new type** — it's purely syntactic.

Two common uses:
1. **Ergonomics** — shorter names for complex type compositions
2. **Library API design** — export sensible defaults so users don't need to know implementation details

```rust
pub(crate) type MyMap = std::collections::HashMap<String, MyStruct>;
```

Now `MyMap` can be used instead of the full type everywhere.

**Real-world example from the dryoc crate (Listing 4.14):**
```rust
pub type Key = StackByteArray<CRYPTO_KDF_KEYBYTES>;
pub type Context = StackByteArray<CRYPTO_KDF_CONTEXTBYTES>;
```

Users of the library use `Key` and `Context` without worrying about the underlying `StackByteArray<N>` implementation.

---

## Section 6: Error Handling with Result (§4.6)

### 6.1 The Result Enum

Error handling in Rust revolves around the `Result` enum:

```rust
pub enum Result<T, E> {
    Ok(T),    // Success, carrying the result value
    Err(E),   // Failure, carrying the error value
}
```

`Result` is ubiquitous — you'll see it as the return type for most fallible operations.

### 6.2 Creating Your Own Error Type

You should create your own error type per crate. This makes it clear to consumers where errors originate.

Simple approach:
```rust
#[derive(Debug)]
struct Error {
    message: String,
}
```

### 6.3 The `?` Operator and `From` Conversions

The `?` operator is central to Rust error handling. It does two things:
1. On `Ok(value)`, extracts `value` and continues.
2. On `Err(e)`, **immediately returns** from the current function with the error.

```rust
fn read_file(name: &str) -> Result<String, Error> {
    let mut file = File::open(name)?;           // ? returns Error on failure
    let mut contents = String::new();
    file.read_to_string(&mut contents)?;        // ? again
    Ok(contents)
}
```

The catch: `File::open` returns `std::io::Error`, but our function returns our custom `Error`. The `?` operator uses the `From` trait to convert automatically. So we must provide:

```rust
impl From<std::io::Error> for Error {
    fn from(other: std::io::Error) -> Self {
        Self { message: other.to_string() }
    }
}
```

Now `?` can convert `std::io::Error` → `Error` transparently.

---

## Section 7: Converting Types with From/Into (§4.7)

### 7.1 The From and Into Traits

These are among Rust's most useful and frequently implemented traits. They provide a **standard way to convert between types**.

**Key rule: Only implement `From`. The compiler derives `Into` automatically.** (Exception: Rust versions before 1.41 had stricter rules requiring `Into` when the destination was an external type.)

`From` is preferred because it doesn't require specifying the destination type:

```rust
pub trait From<T>: Sized {
    fn from(_: T) -> Self;
}
```

**Example:**
```rust
struct StringWrapper(String);

impl From<&str> for StringWrapper {
    fn from(other: &str) -> Self {
        Self(other.into())  // Uses Into<String> for &str
    }
}

println!("{}", StringWrapper::from("Hello, world!").0);
```

### 7.2 From/Into with Error Handling

This is where `From`/`Into` becomes essential in practice. When using `?` with mismatched error types, the compiler tells you exactly which `From` implementation is missing:

```
error[E0277]: `?` couldn't convert the error to `Error`
  the trait `From<std::io::Error>` is not implemented for `Error`
```

The fix is always the same: implement `From<TheirError> for YourError`.

### 7.3 TryFrom and TryInto (§4.7.1)

`TryFrom` and `TryInto` are identical to `From`/`Into` except they return `Result` instead of the type directly. Use these when conversion **can fail** — the alternatives would be panicking (crashing the program), which is unacceptable in most real code.

### 7.4 Best Practices (§4.7.2)

1. Implement `From` for types that need conversion to/from other types.
2. Avoid custom conversion routines — rely on the well-known traits.

---

## Section 8: FFI Compatibility with Rust's Types (§4.8)

### 8.1 The Problem

You sometimes need to call C libraries from Rust (or vice versa). Rust structs are **not** C-compatible by default — the compiler is free to reorder fields, add padding differently, etc.

### 8.2 Making Structs C-Compatible

Two requirements:
1. Declare structs with **`#[repr(C)]`** — tells the compiler to use C-compatible memory layout.
2. Use **C types from the `libc` crate** — Rust types aren't always bitwise compatible with C types, even when they seem equivalent.

### 8.3 The rust-bindgen Tool

The Rust team provides **rust-bindgen**, which automatically generates Rust bindings from C header files. For most cases, use this tool.

### 8.4 Manual FFI — The zlib Example

For simple cases, you can map C structs manually:

**Original C struct:**
```c
struct gzFile_s {
    unsigned have;
    unsigned char *next;
    z_off64_t pos;
};
```

**Rust equivalent:**
```rust
#[repr(C)]
struct GzFileState {
    have: c_uint,
    next: *mut c_uchar,
    pos: i64,
}
```

**Declaring external C functions:**
```rust
type GzFile = *mut GzFileState;  // Type alias for the pointer

#[link(name = "z")]              // Link against libz
extern "C" {
    fn gzopen(path: *const c_char, mode: *const c_char) -> GzFile;
    fn gzread(file: GzFile, buf: *mut c_uchar, len: c_uint) -> c_int;
    fn gzclose(file: GzFile) -> c_int;
    fn gzeof(file: GzFile) -> c_int;
}
```

**The complete `read_gz_file` function** demonstrates:
- Converting Rust strings to C strings with `CString::new()`
- Calling C functions inside an `unsafe` block (required for all FFI calls)
- Reading in a loop until EOF
- Converting raw bytes back to Rust's UTF-8 strings
- Properly closing the file handle

Key detail: `CString::new()` can fail (if the input contains a null byte), so it uses `.expect()` to panic with a message on failure.

---

## Final Synthesis: How Everything Connects

The chapter builds a coherent mental model around one central idea: **in Rust, types define how memory is managed.**

1. **Strings** illustrate this directly — `str` (stack) vs `String` (heap) vs `&str` (borrowed reference to either). The type tells you the allocation strategy.

2. **Slices and arrays** extend this to sequences — arrays have compile-time-known lengths (embedded in the type), slices have runtime lengths (carried alongside the pointer). Both enable zero-cost abstractions over contiguous memory.

3. **Vectors** provide heap-allocated, resizable sequences. Through `Deref`, they seamlessly act as slices, giving you all slice methods for free. `String` is literally a `Vec<u8>` wrapper.

4. **HashMaps** provide constant-time key-value lookups, with a pluggable hashing system built on traits (`BuildHasher`, `Hasher`).

5. **Primitives, structs, enums, and aliases** form Rust's type system hierarchy. Primitives handle scalar data with checked arithmetic (unlike C). Structs compose types. Enums model mutually exclusive variants (far richer than C enums). Aliases provide ergonomic names.

6. **`Result` and `From`/`Into`** tie everything together for error handling and type conversion — `Result` models success/failure, `From` enables the `?` operator to automatically convert between error types, and `TryFrom` handles fallible conversions.

7. **FFI** closes the loop with the outside world — `#[repr(C)]` makes Rust structs C-compatible, the `libc` crate provides type mappings, and `extern "C"` blocks declare foreign functions.

The patterns repeat: ownership vs borrowing, stack vs heap, sized vs unsized, fallible vs infallible. Once you internalize these through the data structures, the rest of Rust becomes much clearer.
