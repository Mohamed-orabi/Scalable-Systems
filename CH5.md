# Chapter 5: Working with Memory in Rust — A Comprehensive Explanation

---

## Preface: Why This Chapter Matters

Chapter 4 covered Rust's data structures. Chapter 5 completes the picture by explaining **how those data structures interact with memory**. The core data types (`String`, `Vec`, `HashMap`) provide nice abstractions, but real-world applications often require more: custom allocators, reference counting, smart pointers, or system-level memory features beyond the language's built-in scope.

The author's key point: you *can* use Rust effectively without deep memory management knowledge. But knowing what happens under the hood makes you dramatically more effective, especially when debugging performance issues, writing systems software, or interfacing with C libraries — all areas directly relevant to your banking system modernization work.

This chapter covers: heap vs stack, ownership semantics, deep copying, avoiding unnecessary copies, smart pointers (`Box`), reference counting (`Rc`/`Arc`), interior mutability (`Cell`/`RefCell`), clone-on-write (`Cow`), and custom allocators.

---

## Section 1: Memory Management — Heap and Stack (§5.1)

### 1.1 The Big Picture

When you use a `String` or `Vec`, you're typically not thinking about *where* the memory lives — just like in Python or Ruby, the language abstracts it away. Under the hood, though, Rust's memory management is very similar to C/C++. The difference is that Rust keeps memory management out of your way *until you need to worry about it*, and then provides fine-grained tools to dial complexity up or down.

### 1.2 The Heap

The heap is a section of memory reserved for **dynamic allocation** — data whose size is only known at run time, or data that needs to grow and shrink during the program's lifetime.

Key properties of the heap:

- Managed by an **allocator** (usually the OS or C library provides one, e.g., `malloc()`)
- Data can be thought of as allocated "randomly" throughout the heap
- Data can grow and shrink throughout the process's life
- Allocation is slower than the stack (because it involves finding free space, bookkeeping, etc.)
- Not strictly required — embedded systems often operate without a heap

In Rust, you allocate on the heap by using any heap-allocated data structure:

```rust
let heap_integer = Box::new(1);                    // Single value, boxed on the heap
let heap_integer_vec = vec![0; 100];               // Vector of 100 zeros, on the heap
let heap_string = String::from("heap string");     // String (based on Vec), on the heap
```

The author notes that `String` is based on `Vec` (as we learned in Chapter 4), which is why it's heap-allocated.

### 1.3 The Stack

The stack is a **thread-local** memory space bound to function scope. It operates in **LIFO (Last In, First Out)** order:

- When a function is entered → a new **frame** is pushed onto the stack
- When a function exits → that frame is popped off
- The size of stack-allocated data must be known at **compile time**
- Allocation is essentially instantaneous (just moving a pointer)
- There is **one stack per thread** on operating systems that support threading
- The stack is managed entirely by the compiler-generated code — you don't manage it

The stack also has a nice secondary property: the function call stack itself can serve as a data structure through recursive calls, with no memory management overhead.

```rust
let stack_integer = 69420;                          // Primitive, on the stack
let stack_allocated_string = "stack string";        // &'static str, on the stack
```

### 1.4 Cross-Language Comparison

The author draws comparisons to ground this knowledge:

- **C/C++**: Heap allocation via `malloc()` or `new`; stack allocation by declaring variables within functions. No automatic cleanup of heap memory in C (you must call `free()`). C++ has destructors and RAII, closer to Rust's model.
- **Java**: Uses `new` for heap allocation, but memory is **garbage collected** — you never manually free anything.
- **Python/Ruby**: Memory management is completely abstracted away.
- **Rust**: Stack is managed by the compiler. Heap allocation can be customized (custom allocators). The compiler ensures heap memory is freed when ownership ends (via the `Drop` trait), without garbage collection.

### 1.5 What Can Live on the Stack?

The author explicitly lists the types that can be stack-allocated:

- Primitive types (integers, floats, booleans, characters)
- Compound types (tuples, structs)
- `str` (string slices)
- The container types *themselves* (but **not necessarily their contents** — a `Vec` struct lives on the stack, but the data it points to lives on the heap)

This last point is subtle and important. When you write `let v = vec![1, 2, 3];`, the variable `v` (which contains a pointer, length, and capacity) lives on the stack. The actual array `[1, 2, 3]` lives on the heap.

---

## Section 2: Understanding Ownership — Copies, Borrowing, References, and Moves (§5.2)

### 2.1 What Is Ownership?

Ownership is **the concept that makes Rust different from every other mainstream language**. It's where Rust's safety guarantees come from — it's how the compiler knows when memory is in scope, being shared, has gone out of scope, or is being misused.

The **borrow checker** enforces three rules:

1. Every value has exactly **one owner**.
2. There can only be **one owner at a time**.
3. When the owner goes out of scope, the **value is dropped** (memory freed).

### 2.2 How Ownership Relates to C/C++/Java

Rust's ownership is *somewhat* similar to C, C++, and Java, with critical differences:

- **No copy constructors**: In C++, assigning one object to another can invoke a copy constructor that duplicates the object. Rust has no such mechanism.
- **No raw pointers in normal use**: While Rust *has* C-like pointers, you almost never use them outside of FFI or `unsafe` code.
- **Assignment is a move**: `let a = b;` **transfers ownership** from `b` to `a`. After this, `b` is invalid. No copy is made (unless it's a primitive/`Copy` type, which gets a bitwise copy).

### 2.3 References and Borrowing

Instead of passing pointers around, Rust uses **references**, created by **borrowing**:

- Data can be passed by **value** (a move) or by **reference** (a borrow)
- Borrowed data is **immutable by default** — you can't modify what the reference points to
- Use the `mut` keyword to get a **mutable reference**, which allows modification
- You can have **multiple immutable references** simultaneously
- You can have **only one mutable reference** at a time (and no immutable references coexisting with it)

Borrowing operators:
- `&` — immutable borrow
- `&mut` — mutable borrow
- `as_ref()` / `as_mut()` — trait methods (`AsRef`, `AsMut`) used by container types to provide access to *internal* data, rather than a reference to the container itself

### 2.4 Code Walkthrough (Listing 5.3)

This example demonstrates the full lifecycle of ownership, borrowing, moving, and invalidation:

```rust
fn main() {
    // Step 1: Create a mutable Vec
    let mut top_grossing_films =
        vec!["Avatar", "Avengers: Endgame", "Titanic"];

    // Step 2: Take a mutable reference
    let top_grossing_films_mutable_reference =
        &mut top_grossing_films;

    // Step 3: Use the mutable reference to modify the data
    top_grossing_films_mutable_reference
        .push("Star Wars: The Force Awakens");

    // Step 4: Take an immutable reference
    // This INVALIDATES the mutable reference from Step 2
    let top_grossing_films_reference = &top_grossing_films;

    // Step 5: Print using the immutable reference
    println!(
        "Printed using immutable reference: {:#?}",
        top_grossing_films_reference
    );

    // Step 6: Move the Vec to a new variable
    // This TRANSFERS ownership — the original variable is now invalid
    let top_grossing_films_moved = top_grossing_films;

    // Step 7: Print using the moved variable (works fine)
    println!("Printed after moving: {:#?}", top_grossing_films_moved);

    // WOULD NOT COMPILE: original variable was moved
    // println!("Print using original value: {:#?}", top_grossing_films);

    // WOULD NOT COMPILE: mutable reference was invalidated at Step 4
    // println!(
    //     "Print using mutable reference: {:#?}",
    //     top_grossing_films_mutable_reference
    // );
}
```

Walking through the critical transitions:

**Step 2 → Step 3:** We borrow mutably with `&mut` and use that reference to `push()` a new item. This works because we hold the only mutable reference.

**Step 4:** We take an immutable reference with `&`. At this moment, the **mutable reference from Step 2 becomes invalid**. Rust's rule: you cannot have a mutable reference and an immutable reference to the same data at the same time. The mutable reference's "lifetime" ends because we create an immutable borrow.

**Step 6:** We do `let top_grossing_films_moved = top_grossing_films;`. This is a **move**. Ownership transfers. After this line, `top_grossing_films` is no longer valid — using it would be a compile error.

The commented-out lines at the bottom demonstrate what would fail: you can't use the original variable after a move, and you can't use the mutable reference after it was invalidated by the immutable borrow.

The output shows all four films (including the pushed "Star Wars: The Force Awakens") in both prints, confirming the data was properly modified and then moved.

---

## Section 3: Deep Copying (§5.3)

### 3.1 The Problem Deep Copying Solves

In many languages (Python, Ruby, JavaScript), assignment creates a **shallow copy** — a reference to the same data, not a new copy. This can cause unintended side effects: you modify what you think is your own copy, but you're actually changing the shared original.

Rust eliminates this entire category of bugs: **there are no shallow copies in Rust**. Instead, there is **borrowing** (explicit references) and **moving** (transfer of ownership). If you want an actual duplicate, you must explicitly **clone**.

### 3.2 The Clone Trait

In Rust, the term **cloning** (not "copying") describes creating a new data structure with all data duplicated from the original. This is done through the `clone()` method, which comes from the `Clone` trait.

Key facts about `Clone`:
- Can be automatically derived with `#[derive(Clone)]`
- When derived, operates **recursively** — cloning a `Vec` clones all its contents
- Most standard library types already implement `Clone`
- Deeply nested structures can be cloned without extra work, as long as all contained types implement `Clone`

### 3.3 Code Walkthrough (Listing 5.4)

```rust
fn main() {
    let mut most_populous_us_cities =
        vec!["New York City", "Los Angeles", "Chicago", "Houston"];

    // Clone: creates an entirely separate, independent copy
    let most_populous_us_cities_cloned = most_populous_us_cities.clone();

    // Modify the ORIGINAL — the clone is unaffected
    most_populous_us_cities.push("Phoenix");

    println!("most_populous_us_cities = {:#?}", most_populous_us_cities);
    println!(
        "most_populous_us_cities_cloned = {:#?}",
        most_populous_us_cities_cloned
    );
}
```

Output: The original contains 5 cities (including "Phoenix"), while the clone still has only 4. They are completely independent data structures. The `clone()` created a deep copy — new heap allocation, new `Vec`, with all string data duplicated.

---

## Section 4: Avoiding Copies (§5.4)

### 4.1 The Problem with Excessive Cloning

`Clone` makes it *too easy* to copy data. In string processing or large dataset operations, you can accidentally end up with many copies of the same data, wasting memory and CPU time.

Many core library functions return **new copies** rather than modifying in place. This is intentional — immutability makes it easier to reason about algorithms — but it comes at the cost of duplicating memory.

### 4.2 How to Identify Whether a Function Copies

The author provides an invaluable analysis table of core string functions. The pattern:

**You can identify whether a function copies by examining its signature.**

| Signature Pattern | Behavior | Example |
|---|---|---|
| `fn func(&self) -> &str` | **No copy** — returns a reference/slice | `trim()` returns a `&str` slice into the existing string |
| `fn func(&self) -> String` | **Copy likely** — immutable input, owned output | `to_lowercase()` creates a new `String` |
| `fn func(&mut self)` | **Modifies in place** — no copy | `make_ascii_lowercase()` changes bytes in place |
| `fn func(String) -> String` | **Copy likely** — takes owned, returns owned | Ownership passes in, new string created internally |
| `fn func(mut String) -> String` | **Pass-through likely** — modifies in place and returns | Takes mutable ownership, modifies, returns same allocation |

### 4.3 The Pass-Through Pattern

The author introduces an important pattern:

```rust
fn lowercased(s: String) -> String {
    s.to_lowercase()        // Creates a COPY — the original s is dropped
}

fn lowercased_ascii(mut s: String) -> String {
    s.make_ascii_lowercase();   // Modifies in place — no copy
    s                           // Returns the same allocation
}
```

The first function takes ownership of `s`, calls `to_lowercase()` which creates a *new* `String`, and returns it. The original `s` is dropped. **Two allocations existed briefly.**

The second function takes *mutable* ownership, modifies the string in place with `make_ascii_lowercase()`, and returns the same object. **Only one allocation ever existed.**

### 4.4 Summary Rules for Function Behavior

The author distills five rules (reproduced exactly):

1. Functions taking immutable references and returning references/slices are **unlikely to make copies** (e.g., `fn func(&self) -> &str`).
2. Functions taking a reference and returning an owned object **may be creating a copy** (e.g., `fn func(&self) -> String`).
3. Functions taking a mutable reference **may be modifying data in place** (e.g., `fn func(&mut self)`).
4. Functions taking an owned object and returning an owned object of the same type are **likely making a copy** (e.g., `fn func(String) -> String`).
5. Functions taking a mutable owned object and returning the same type **may not be making a copy** (e.g., `fn func(mut String) -> String`).

**General advice:** When unsure, examine documentation and source code. Rust's memory semantics make it relatively easy to reason about behavior just from signatures, but only if the functions follow these patterns. For serious performance concerns, examine the underlying algorithms directly.

---

## Section 5: To Box or Not to Box — Smart Pointers (§5.5)

### 5.1 What Is Box?

`Box` is Rust's simplest smart pointer. Its purpose: **allocate a single value on the heap**. That's essentially all it does — allocation and deallocation. It doesn't provide reference counting, sharing, or any other features. But simplicity is a feature.

The two main ways to allocate on the heap in Rust:
- `Vec` — for sequences of values
- `Box` — for single values

### 5.2 Box and Option — The Null Safety Pattern

A `Box` cannot be empty (with rare exceptions the author deliberately avoids discussing). So if the boxed data might not be present, wrap it in `Option`:

```rust
let maybe_data: Option<Box<MyType>> = None;    // No data yet
let maybe_data: Option<Box<MyType>> = Some(Box::new(my_value));  // Has data
```

**What is `Option`?** If you haven't encountered optionals before (from Ada, Haskell, Scala, Swift), think of `Option` as a safe way to handle null values. Rust doesn't have null pointers (outside `unsafe` code). Instead, it has `None`, which is functionally equivalent but without the safety problems (no null pointer exceptions, no uninitialized memory access).

The author notes that `Option` is a kind of **monad** — a functional design pattern where you wrap values that shouldn't have unrestricted access in a function. Rust provides syntax sugar to make working with `Option` pleasant (pattern matching, `if let`, the `?` operator, etc.).

**The safety guarantee:** When `Box` and `Option` are used together, it's **nearly impossible** to have runtime errors from invalid, uninitialized, or doubly-freed memory. The one caveat: heap allocations can fail (out of memory).

### 5.3 Handling Allocation Failures

Heap allocation can fail, most commonly when the system runs out of memory (OOM). How to handle this:

- **Default behavior:** `Box::new()` panics on allocation failure → program crashes. For most programs, this is the correct response.
- **Graceful handling:** `Box::try_new()` returns a `Result`, letting you handle the failure. Use this for mission-critical software (databases, transaction-processing systems).
- **Custom allocators** can also catch allocation failures (covered later in the chapter).

The author's practical advice: Most of the time, crashing is the best response to OOM. Notable exceptions include web browsers (which have their own task managers and memory management, much like the OS itself) and databases.

**Tip from the author:** To truly understand `Option` and `Result`, try implementing them yourself using an enum. In Rust, creating your own optionals is trivial with enums and pattern matching.

### 5.4 The Linked List Example (Listing 5.5) — Box in Practice

The author uses a singly linked list to demonstrate `Box`, `Option`, and ownership interacting together. This is one of the best learning exercises for Rust newcomers. Let's walk through every piece:

#### The Data Structures

```rust
struct ListItem<T> {
    data: Box<T>,                       // Data is boxed (heap-allocated)
    next: Option<Box<ListItem<T>>>,     // Next pointer: optional boxed list item
}

struct SinglyLinkedList<T> {
    head: ListItem<T>,                  // Head is NOT boxed — it must always exist
}
```

Why `Box<T>` for `data`? Because the data lives on the heap, and `Box` gives us ownership semantics — when the `ListItem` is dropped, the `Box` (and its data) are automatically freed.

Why `Option<Box<ListItem<T>>>` for `next`? Because:
- `Option` handles the "end of list" case — `None` means no next item.
- `Box` provides heap allocation for the next node.
- Without `Box`, the struct would have infinite size (it contains itself recursively). `Box` breaks the recursion by storing a *pointer* (fixed size) rather than the value itself.

Why is `head` not boxed? Because the list must always have a head — it can never be empty. The head is stored directly in the struct (on the stack, or wherever the `SinglyLinkedList` lives).

#### The ListItem Methods

```rust
impl<T> ListItem<T> {
    fn new(data: T) -> Self {
        ListItem {
            data: Box::new(data),   // Move data into a Box (heap allocation)
            next: None,             // New items don't know their position yet
        }
    }
```

`new()` creates a new list item. Data is moved into a `Box` (heap-allocated). `next` starts as `None` because new elements are created independently before being linked.

```rust
    fn next(&self) -> Option<&Self> {
        if let Some(next) = &self.next {
            Some(next.as_ref())     // Equivalent to Some(&*next)
        } else {
            None
        }
    }
```

`next()` returns an optional reference to the next item. The `if let` syntax destructures the `Option` — if `self.next` is `Some(box_ref)`, we extract the inner reference. `next.as_ref()` converts from `&Box<ListItem<T>>` to `&ListItem<T>` — it gives us a reference to the *contents* of the Box, not the Box itself. This is equivalent to writing `&*next` (dereference the Box, then take a reference).

```rust
    fn mut_tail(&mut self) -> &mut Self {
        if self.next.is_some() {
            self.next.as_mut().unwrap().mut_tail()  // Recurse
        } else {
            self  // This is the tail
        }
    }
```

`mut_tail()` finds the last element by recursing through `next` pointers. The author notes a subtle point: **`if let` won't work here** because we can't borrow `self.next` (for the `if let` pattern match) and simultaneously return a mutable reference to the inner pointer. Instead, we use `is_some()` to check, then `as_mut().unwrap()` to get the mutable reference. This is a common pattern when the borrow checker prevents the more elegant approach.

The chain `self.next.as_mut().unwrap()`:
1. `as_mut()` — converts `&mut Option<Box<ListItem<T>>>` to `Option<&mut Box<ListItem<T>>>`
2. `unwrap()` — extracts the `&mut Box<ListItem<T>>` (safe because we checked `is_some()`)
3. The resulting `&mut Box<ListItem<T>>` auto-derefs to `&mut ListItem<T>`

```rust
    fn data(&self) -> &T {
        self.data.as_ref()  // &Box<T> → &T
    }
}
```

`data()` provides a convenient reference to the inner `T`. `as_ref()` on a `Box<T>` gives `&T`.

#### The SinglyLinkedList Methods

```rust
impl<T> SinglyLinkedList<T> {
    fn new(data: T) -> Self {
        SinglyLinkedList {
            head: ListItem::new(data),  // Must provide first element
        }
    }
```

The list requires a first element at construction. To permit an empty list, the head would need to be wrapped in `Option` — a design choice the author deliberately avoids to keep the example focused.

```rust
    fn append(&mut self, data: T) {
        let mut tail = self.head.mut_tail();     // Find the current tail
        tail.next = Some(Box::new(ListItem::new(data)));  // Link new item
    }
```

`append()` finds the tail (last element where `next` is `None`), then sets its `next` pointer to a newly boxed `ListItem`. The new element becomes the new tail.

```rust
    fn head(&self) -> &ListItem<T> {
        &self.head
    }
}
```

`head()` provides direct access to the head element.

#### Testing the Linked List

```rust
fn main() {
    let mut list = SinglyLinkedList::new("head");
    list.append("middle");
    list.append("tail");

    let mut item = list.head();
    loop {
        println!("item: {}", item.data());
        if let Some(next_item) = item.next() {
            item = next_item;
        } else {
            break;
        }
    }
}
```

Output: `head`, `middle`, `tail`. The loop traverses by getting the head, printing its data, then following `next` pointers until `None`.

### 5.5 The Safety Guarantee

The author emphasizes: this linked list is **safe**. It will never be empty, never contain null pointers, never have invalid references. This is only possible because of Rust's ownership rules — `Box` owns its data, `Option` handles the absence case, and the borrow checker prevents misuse.

### 5.6 Practical Note

In practice, you'll almost never implement your own linked list. Rust provides `std::collections::LinkedList`. And for most use cases, **just use a `Vec`**. The linked list exercise is primarily a learning tool for understanding ownership and smart pointers.

---

## Section 6: Reference Counting (§5.6)

### 6.1 The Limitation of Box

`Box` **cannot be shared**. You can't have two separate `Box`es pointing to the same data. `Box` owns its data exclusively and doesn't allow more than one borrow at a time. This is usually a feature (preventing aliasing bugs), but sometimes you genuinely need shared ownership — for example, across threads or when the same data appears in multiple data structures (a `Vec` and a `HashMap` simultaneously).

### 6.2 Rc and Arc — Reference-Counted Smart Pointers

When `Box` isn't enough, Rust provides **reference-counted smart pointers**:

- **`Rc`** (Reference Counted) — **single-threaded** shared ownership
- **`Arc`** (Atomically Reference Counted) — **multi-threaded** shared ownership

**How reference counting works:**
1. A static counter tracks how many copies of a pointer exist.
2. Every time a new copy (clone) is made, the counter increments.
3. When a copy is destroyed (dropped), the counter decrements.
4. When the counter reaches **zero**, the memory is freed — no more copies exist, so the data is no longer accessible.

**Tip from the author:** Implementing your own reference-counted smart pointer is a great exercise, though it's tricky in Rust — it requires raw (unsafe) pointers. If the linked list exercise is too easy, try this.

### 6.3 Single- vs. Multi-Threaded Objects in Rust (Sidebar)

The author provides an important sidebar about threading in Rust:

- In many languages, functions/objects are classified as "thread safe" vs "unsafe." Rust doesn't map directly to this — **everything is safe by default**.
- Instead, the distinction is about whether objects can be **moved across threads** (`Send` trait) or **shared across threads** (`Sync` trait).
- **`Rc`** explicitly marks `Send` and `Sync` as **not implemented** — it can only be used in a single thread.
- **`Arc`** implements both `Send` and `Sync` — safe for multi-threaded use.
- `Arc` uses **atomic counters** (platform-dependent, usually CPU-level operations). Atomic operations are more expensive than regular arithmetic, so **only use `Arc` when you need atomicity**.
- As long as you don't use `unsafe`, Rust code is always safe. Getting it to **compile**, however, can be challenging.

### 6.4 Interior Mutability — Cell and RefCell

To use `Rc`/`Arc` effectively, we need **interior mutability** — the ability to mutate data inside an immutably-borrowed container.

**Why is this needed?** The borrow checker sometimes doesn't provide enough flexibility with mutable references. There are cases where perfectly safe code won't compile because the compiler doesn't understand what you're trying to do.

**Is this an escape hatch?** Yes! But it doesn't break Rust's safety contracts. It still allows safe code — it just defers some borrow-checking from compile time to run time.

Rust provides two types for interior mutability:

- **`RefCell`** — allows borrowing **references** to inner data (most commonly used)
- **`Cell`** — moves values **in and out** of itself (usually not what you want)

**When to use:** You shouldn't need `RefCell` or `Cell` very often. If you find yourself using them to "get around" the borrow checker, **rethink your design**. They're mainly for containers and data structures that hold data needing mutable access.

**Threading limitation:** `Cell` and `RefCell` are **single-threaded only**. For multi-threaded equivalents:
- `Mutex` — mutual exclusion (like `RefCell` but thread-safe)
- `RwLock` — reader/writer lock (distinguishes between read and write access)

These are paired with `Arc` rather than `Rc`. Concurrency is explored in Chapter 10.

### 6.5 The Doubly Linked List — Rc and RefCell in Practice (Listing 5.6)

The author upgrades the singly linked list to a **doubly linked list**, which is **impossible with `Box`** because `Box` doesn't allow shared ownership. In a doubly linked list, each node is pointed to by *both* its predecessor and its successor — two owners for the same node.

#### The Data Structures

```rust
use std::cell::RefCell;
use std::rc::Rc;

struct ListItem<T> {
    prev: Option<ItemRef<T>>,           // NEW: pointer to previous item
    data: Box<T>,                       // Data stays in Box — not shared
    next: Option<ItemRef<T>>,
}

type ItemRef<T> = Rc<RefCell<ListItem<T>>>;   // Type alias for readability
```

The type alias `ItemRef<T>` is crucial. Without it, you'd write `Rc<RefCell<ListItem<T>>>` everywhere — unreadable.

**Why is `data` still in a `Box`?** Because we're not sharing ownership of the *data* — only of the *nodes*. Each piece of data belongs to exactly one node. But nodes themselves are pointed to from multiple directions (prev and next).

**Why `Rc<RefCell<ListItem<T>>>`?**
- `Rc` — enables shared ownership (multiple pointers to the same node)
- `RefCell` — enables interior mutability (we need to modify node pointers even when we only have a shared `Rc` reference)
- Together: we can have multiple references to a node AND modify its contents

#### The DoublyLinkedList Implementation

```rust
struct DoublyLinkedList<T> {
    head: ItemRef<T>,    // Head is an Rc<RefCell<ListItem<T>>>
}

impl<T> ListItem<T> {
    fn new(data: T) -> Self {
        ListItem {
            prev: None,
            data: Box::new(data),     // Data moved into Box
            next: None,
        }
    }

    fn data(&self) -> &T {
        self.data.as_ref()
    }
}
```

```rust
impl<T> DoublyLinkedList<T> {
    fn new(data: T) -> Self {
        DoublyLinkedList {
            head: Rc::new(RefCell::new(ListItem::new(data))),
        }
    }
```

Construction: `ListItem::new(data)` creates the item → `RefCell::new(...)` wraps it for interior mutability → `Rc::new(...)` wraps it for shared ownership.

```rust
    fn append(&mut self, data: T) {
        // Step 1: Find the current tail
        let tail = Self::find_tail(self.head.clone());

        // Step 2: Create the new node
        let new_item = Rc::new(RefCell::new(ListItem::new(data)));

        // Step 3: Set the new node's prev to point to the old tail
        new_item.borrow_mut().prev = Some(tail.clone());

        // Step 4: Set the old tail's next to point to the new node
        tail.borrow_mut().next = Some(new_item);
    }
```

This is the key method. Let's trace each step:

**Step 1:** `self.head.clone()` clones the `Rc` — this increments the reference count by 1, giving us a new `Rc` pointing to the same data. It does **not** clone the underlying `ListItem`. We pass this to `find_tail()` to locate the last node.

**Step 2:** Create a new node wrapped in `Rc<RefCell<...>>`.

**Step 3:** `new_item.borrow_mut()` calls `RefCell::borrow_mut()`, which returns a mutable reference to the inner `ListItem`. We set its `prev` to a clone of the `tail` Rc (incrementing the reference count so both the new node's `prev` pointer and the `tail` variable point to the same node).

**Step 4:** `tail.borrow_mut()` similarly gets a mutable reference to the tail's inner data. We set `next` to point to the new node.

```rust
    fn find_tail(item: ItemRef<T>) -> ItemRef<T> {
        if let Some(next) = &item.borrow().next {
            Self::find_tail(next.clone())    // Recurse with cloned Rc
        } else {
            item.clone()                     // This is the tail
        }
    }
```

`find_tail()` recursively follows `next` pointers. `item.borrow()` gets an immutable reference to the inner `ListItem`. If `next` exists, we clone the Rc (incrementing the count) and recurse. If not, we're at the tail — return a clone.

```rust
    fn head(&self) -> ItemRef<T> {
        self.head.clone()
    }

    fn tail(&self) -> ItemRef<T> {
        Self::find_tail(self.head())
    }
}
```

Both return cloned `Rc`s, which is the standard pattern — you clone `Rc` to get a new shared reference.

### 6.6 Summary of Rc/Arc + Interior Mutability

- `Rc` and `Arc` provide reference-counted pointers for shared ownership
- To access inner data mutably through a shared pointer, use:
  - **Single-threaded:** `RefCell` or `Cell` (paired with `Rc`)
  - **Multi-threaded:** `Mutex` or `RwLock` (paired with `Arc`)

---

## Section 7: Clone on Write (§5.7)

### 7.1 Copy on Write vs Clone on Write

**Copy on write (COW)** is a design pattern where data is never mutated in place. Instead, any mutation triggers a copy to a new location, the copy is mutated, and a reference to the new copy is returned. The original remains unchanged.

Languages/libraries that use this pattern:
- **Scala** — classifies data structures as mutable or immutable; immutable ones implement COW
- **Immutable.js** (JavaScript) — all mutations return new copies

Benefits: Makes it much easier to reason about data flow. No hidden mutations.

In Rust, this pattern is called **clone on write** because it depends on the `Clone` trait.

### 7.2 Clone vs Copy Traits

These two traits are often confused:

- **`Copy`** — a **bitwise copy** (literally copying bytes to a new memory location). Happens **implicitly** via assignment (`let x = y;`). Only for types where bitwise duplication is safe (primitives, simple structs of `Copy` types).
- **`Clone`** — an **explicit copy**. You call `clone()` manually. Can have custom logic. Typically auto-derived with `#[derive(Clone)]`.

### 7.3 Three Smart Pointers for Clone on Write

Rust provides three approaches:

1. **`Cow`** — an enum-based smart pointer with convenient semantics
2. **`Rc::make_mut()`** — clone-on-write for reference-counted single-threaded pointers
3. **`Arc::make_mut()`** — clone-on-write for reference-counted multi-threaded pointers

### 7.4 The Cow Type (Listing 5.7)

```rust
pub enum Cow<'a, B>
where
    B: 'a + ToOwned + ?Sized,
{
    Borrowed(&'a B),
    Owned(<B as ToOwned>::Owned),
}
```

`Cow` is an enum with two variants:
- `Borrowed(&'a B)` — holds a reference (no allocation)
- `Owned(...)` — holds owned data (like `Box`, but not necessarily heap-allocated)

Key insight: `Cow` starts as a borrow (cheap, no allocation). If you need to mutate, calling `to_mut()` triggers a clone — the borrowed data is cloned into an owned variant, and you get a mutable reference to the clone. If it's already owned, `to_mut()` just returns a mutable reference (no clone needed).

**Important nuance:** Unlike `Box`, `Cow` data is not necessarily heap-allocated. If you want heap allocation with `Cow`, use a `Box` inside `Cow`, or use `Rc`/`Arc` instead. Cow is not a language-level feature — you must explicitly use it.

### 7.5 Cow Singly Linked List (Listings 5.8 and 5.9)

The author rewrites the singly linked list to be **immutable** using `Cow`:

**ListItem** (Listing 5.8): Almost identical to the original, but with `#[derive(Clone)]` added and a `where T: Clone` bound. The `Clone` derivation is essential because `Cow` depends on the Clone trait's behavior.

**SinglyLinkedList** (Listing 5.9): The critical changes:

```rust
#[derive(Clone)]
struct SinglyLinkedList<'a, T>
where
    T: Clone,
{
    head: Cow<'a, ListItem<T>>,    // Head is stored in a Cow
}
```

The head pointer is stored within a `Cow`, requiring a lifetime specifier `'a` so the compiler knows the struct and its borrowed data have the same lifetime.

The key method change — `append()`:

```rust
fn append(&self, data: T) -> Self {          // No longer &mut self!
    let mut new_list = self.clone();          // Clone the entire list
    let mut tail = new_list.head.to_mut().mut_tail();  // to_mut() triggers COW
    tail.next = Some(Box::new(ListItem::new(data)));
    new_list                                 // Return the new list
}
```

The signature changed dramatically:
- **Before:** `fn append(&mut self, data: T)` — mutates in place
- **After:** `fn append(&self, data: T) -> Self` — no mutation, returns a new list

`self.clone()` creates a complete copy of the list. `new_list.head.to_mut()` triggers the clone-on-write — if the head was borrowed, it's now cloned into an owned variant. We get a mutable reference to the new head, find its tail, and append. The new list is returned, leaving the original untouched.

---

## Section 8: Custom Allocators (§5.8)

### 8.1 When You Need Custom Allocators

The author lists specific scenarios:

1. **Embedded systems** — highly memory-constrained or lacking an OS
2. **Performance-critical applications** — custom heap managers like `jemalloc` or `TCMalloc`
3. **Security/safety-critical applications** — protecting memory pages with `mprotect()` and `mlock()` system calls
4. **Cross-language boundaries** — special allocators for handing off data between Rust and garbage-collected languages to avoid memory leaks
5. **Custom heap management** — tracking memory usage from within your application

### 8.2 The Two Allocator APIs

Rust provides two levels of allocator customization:

- **`GlobalAlloc`** — overrides the allocator for the **entire Rust program** (available in stable Rust)
- **`Allocator`** — overrides the allocator for **individual data structures** (nightly-only as of writing)

By default, Rust uses the system's `malloc()` and `free()`.

**Note:** The `Allocator` API requires nightly Rust. To use it, run `cargo +nightly ...` or set the toolchain with `rustup override set nightly`.

### 8.3 The Allocator Trait (Listing 5.10)

```rust
pub unsafe trait Allocator {
    // Required methods:
    fn allocate(&self, layout: Layout) -> Result<NonNull<[u8]>, AllocError>;
    unsafe fn deallocate(&self, ptr: NonNull<u8>, layout: Layout);

    // Optional methods (with default implementations):
    fn allocate_zeroed(&self, layout: Layout) -> Result<NonNull<[u8]>, AllocError> { ... }
    unsafe fn grow(&self, ptr: NonNull<u8>, old: Layout, new: Layout) -> Result<NonNull<[u8]>, AllocError> { ... }
    unsafe fn grow_zeroed(&self, ptr: NonNull<u8>, old: Layout, new: Layout) -> Result<NonNull<[u8]>, AllocError> { ... }
    unsafe fn shrink(&self, ptr: NonNull<u8>, old: Layout, new: Layout) -> Result<NonNull<[u8]>, AllocError> { ... }
    fn by_ref(&self) -> &Self { ... }
}
```

Only two methods are required: `allocate()` and `deallocate()` — analogous to `malloc()` and `free()`.

C equivalents for the optional methods:
- `allocate_zeroed()` → `calloc()` (allocate + zero-fill)
- `grow()` / `shrink()` → `realloc()` (resize allocation)
- `grow_zeroed()` → `realloc()` + zero-fill new portion

**Why `unsafe`?** Allocating and deallocating memory nearly always involves unsafe operations. `deallocate()` is marked `unsafe` because calling it with invalid data (bad pointer, incorrect layout) produces undefined behavior.

**Default implementations:** The grow/shrink defaults allocate new memory, copy all data, then deallocate old memory. `allocate_zeroed()` calls `allocate()` and writes zeros.

### 8.4 Pass-Through Allocator (Listing 5.11)

The simplest possible custom allocator — just delegates to Rust's global allocator:

```rust
#![feature(allocator_api)]   // Required for nightly feature
use std::alloc::{AllocError, Allocator, Global, Layout};
use std::ptr::NonNull;

pub struct PassThruAllocator;

unsafe impl Allocator for PassThruAllocator {
    fn allocate(&self, layout: Layout) -> Result<NonNull<[u8]>, AllocError> {
        Global.allocate(layout)       // Delegate to global allocator
    }

    unsafe fn deallocate(&self, ptr: NonNull<u8>, layout: Layout) {
        Global.deallocate(ptr, layout)  // Delegate to global allocator
    }
}
```

Testing it:

```rust
fn main() {
    let mut custom_alloc_vec: Vec<i32, _> =
        Vec::with_capacity_in(10, BasicAllocator);    // Use custom allocator
    for i in 0..10 {
        custom_alloc_vec.push(i as i32 + 1);
    }
    println!("custom_alloc_vec={:?}", custom_alloc_vec);
}
```

`Vec::with_capacity_in()` creates a vector with a specified capacity using a custom allocator. Output: `[1, 2, 3, 4, 5, 6, 7, 8, 9, 10]` — works identically to the default allocator.

### 8.5 Basic malloc/free Allocator (Listing 5.12)

A custom allocator that calls C's `malloc()` and `free()` directly:

```rust
use libc::{free, malloc};

pub struct BasicAllocator;

unsafe impl Allocator for BasicAllocator {
    fn allocate(&self, layout: Layout) -> Result<NonNull<[u8]>, AllocError> {
        unsafe {
            // Call C's malloc() with the requested size
            let ptr = malloc(layout.size() as libc::size_t);

            // Convert raw C pointer to a Rust slice
            let slice = std::slice::from_raw_parts_mut(
                ptr as *mut u8,
                layout.size(),
            );

            // Wrap in NonNull and return
            Ok(NonNull::new_unchecked(slice))
        }
    }

    unsafe fn deallocate(&self, ptr: NonNull<u8>, _layout: Layout) {
        // Convert Rust pointer to C pointer and free
        free(ptr.as_ptr() as *mut libc::c_void);
    }
}
```

Key details:

- `allocate()` doesn't have `unsafe` in its trait signature, but it still needs `unsafe` internally (because `malloc()` and raw pointer manipulation are unsafe operations). Hence the `unsafe {}` block.
- `deallocate()` already has `unsafe` in the trait signature (it's inherently unsafe to free memory).
- The `Layout` struct provides `size()` (minimum bytes to allocate) and `align()` (minimum byte alignment, in powers of two). For portability, both should be handled. The author notes this is simplified for the example.

### 8.6 Protected Memory Allocator — A Real-World Example (Listings 5.13–5.14)

The author presents a partial implementation from the **dryoc** cryptographic library, demonstrating a genuine production use case for custom allocators.

**The problem:** Security-sensitive applications (handling secret keys, passwords, etc.) need to protect memory from being read by other processes or dumped to disk. Modern OSes provide system calls for this:
- UNIX: `mprotect()` (control access permissions) and `mlock()` (prevent swapping to disk)
- Windows: `VirtualProtect()` and `VirtualLock()`

**The memory layout (Figure 5.2):**

```
Page-aligned protected memory:
┌─────────────────────┐
│ Fore page           │  ← no-access (prevents scanning from below)
│ (unused, locked)    │
├─────────────────────┤
│ Target memory       │  ← read/write (your actual data)
│ (active region)     │
├─────────────────────┤
│ Aft page            │  ← no-access (prevents scanning from above)
│ (unused, locked)    │
└─────────────────────┘
```

Two extra memory pages are allocated as "bumpers" — one before and one after the actual data. These are locked and marked no-access, protecting against certain types of memory attacks.

**The allocate() implementation (Listing 5.13):**

1. Calculate the total size: round up to page boundaries + 2 extra pages (fore and aft)
2. Allocate page-aligned memory:
   - UNIX: `posix_memalign()` — allocates memory aligned to page boundaries
   - Windows: `VirtualAlloc()` with `MEM_COMMIT | MEM_RESERVE`
3. Mark the **fore page** as **no-access** with `mprotect_noaccess()` — prevents scanning
4. Mark the **aft page** as **no-access** — prevents scanning from the other direction
5. Mark the **target region** as **read/write** — this is where actual data goes
6. Return a slice pointing to only the target region

**The deallocate() implementation (Listing 5.14):**

The reverse process:
1. Calculate the original pointer (offset back by one page)
2. Return the **fore page** to read/write (restore default state so it can be freed)
3. Return the **aft page** to read/write
4. Free the memory:
   - UNIX: `libc::free()`
   - Windows: `VirtualFree()` with `MEM_RELEASE`

**Conditional compilation with `cfg`:** The allocator uses `#[cfg(unix)]` and `#[cfg(windows)]` to compile platform-specific code. This is Rust's equivalent of C's `#ifdef`:

### 8.7 Conditional Compilation — cfg, cfg_attr, and the cfg! Macro (Sidebar)

Rust provides three tools for conditional compilation:

1. **`#[cfg(...)]` attribute** — conditionally includes the attached code item
2. **`#[cfg_attr(...)]` attribute** — sets new compiler attributes based on existing ones
3. **`cfg!(...)` macro** — returns `true` or `false` at compile time

Example:

```rust
#[cfg(target_family = "unix")]
fn get_platform() -> String { "UNIX".into() }

#[cfg(target_family = "windows")]
fn get_platform() -> String { "Windows".into() }

fn main() {
    println!("Running on {} family OS", get_platform());

    if cfg!(target_feature = "avx2") {
        println!("avx2 is enabled");
    }

    if cfg!(not(any(target_arch = "x86", target_arch = "x86_64"))) {
        println!("Running on a non-Intel CPU");
    }
}
```

Key details:
- `cfg` attribute applies to the entire following item (function, block, statement)
- Shorthand predicates: `#[cfg(unix)]` instead of `#[cfg(target_family = "unix")]`
- Predicates can be combined: `all()`, `any()`, `not()`. `all()` and `any()` accept lists; `not()` accepts one.
- Get all configuration values for your target: `rustc --print=cfg -C target-cpu=native`
- Full listing of options: http://mng.bz/OP7K

---

## Section 9: Smart Pointers Summarized (§5.9)

The author provides a comprehensive reference table:

| Type | Kind | Description | When to Use | Threading |
|------|------|-------------|-------------|-----------|
| **`Box`** | Pointer | Heap-allocated smart pointer | Single value on the heap (not in a Vec) | Single |
| **`Cow`** | Pointer | Clone-on-write, works with owned or borrowed data | Heap data with clone-on-write functionality | Single |
| **`Rc`** | Pointer | Reference-counted, heap-allocated, shared ownership | Shared ownership of heap data | Single |
| **`Arc`** | Pointer | Atomically reference-counted, shared ownership | Shared ownership across threads | Multi |
| **`Cell`** | Container | Interior mutability using move | Interior mutability via move semantics | Single |
| **`RefCell`** | Container | Interior mutability using references | Interior mutability via references | Single |
| **`Mutex`** | Container | Mutual exclusion + interior mutability | Synchronize data across threads | Multi |
| **`RwLock`** | Container | Reader/writer lock + interior mutability | Reader/writer locking across threads | Multi |

---

## Final Synthesis: How Everything Connects

This chapter builds a layered model of Rust's memory management, from simple to complex:

**Layer 1: The Foundation — Stack vs Heap.** Stack is fast, automatic, and size-must-be-known-at-compile-time. Heap is flexible, dynamic, but requires allocation/deallocation management. Rust's types encode this distinction: primitives live on the stack, `Vec`/`String`/`Box` live on the heap.

**Layer 2: Ownership — The Core Innovation.** Every value has one owner. Assignment is a move (transfer of ownership). References are borrows (temporary access). This eliminates entire classes of bugs (use-after-free, double-free, dangling pointers) at compile time with zero runtime cost.

**Layer 3: Deep Copying with Clone.** When you genuinely need a duplicate, use `clone()`. It's explicit, recursive, and composable through `#[derive(Clone)]`. The chapter teaches you to recognize when copies are being made by reading function signatures — a skill that directly translates to performance optimization.

**Layer 4: Box — Single-Value Heap Allocation.** The simplest smart pointer. Owns its data exclusively. Combined with `Option`, provides null-safety guarantees. The linked list example demonstrates how `Box` enables recursive types and heap allocation while maintaining complete memory safety.

**Layer 5: Rc/Arc — Shared Ownership.** When one owner isn't enough, reference counting allows multiple owners. `Rc` for single-threaded, `Arc` for multi-threaded. But shared ownership creates a tension with mutability, which leads to...

**Layer 6: Interior Mutability — Cell, RefCell, Mutex, RwLock.** These "escape hatches" defer borrow-checking from compile time to run time, allowing mutation through shared references. `Cell`/`RefCell` for single-threaded code, `Mutex`/`RwLock` for multi-threaded.

**Layer 7: Clone on Write — Immutable Data Patterns.** `Cow` enables functional programming patterns where data is never mutated in place. Mutations produce new copies, leaving originals untouched. This makes reasoning about data flow much simpler.

**Layer 8: Custom Allocators — Full Control.** For specialized needs (embedded, security, performance), Rust lets you replace the memory allocator entirely. The `Allocator` trait requires only `allocate()` and `deallocate()`. The dryoc example shows a real-world use: protected memory with guard pages for cryptographic key storage.

The unifying theme: **Rust gives you C-level control over memory with language-level safety guarantees.** Each layer adds capability while maintaining the invariant that safe Rust code cannot have memory errors. The progression from `Box` → `Rc`/`Arc` → `Cell`/`RefCell` → `Cow` → custom allocators mirrors increasing complexity, but each tool exists because the simpler tools genuinely aren't sufficient for certain use cases.
