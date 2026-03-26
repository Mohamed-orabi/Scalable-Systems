# Understanding Pages — The Complete Beginner's Guide

## From Bytes to Real Database Operations

---

# Chapter 1: Forget Everything — Start With a Piece of Paper

Imagine a blank piece of paper. It's exactly 20 centimeters tall.
That's it. That's a page.

In a database, a page is a FIXED number of bytes — always 4096 bytes
(4KB). Not 4095, not 4097. Always exactly 4096.

```
A page is just this:

  byte 0    ┌──────────────────────┐
            │                      │
            │                      │
            │   4096 bytes         │
            │   of space           │
            │                      │
            │   (initially all     │
            │    zeros)            │
            │                      │
            │                      │
  byte 4095 └──────────────────────┘

That's it. A box of 4096 bytes. Nothing magical.
```

**Question**: "Why 4096?"
**Answer**: Because that's how the CPU and disk naturally work. The CPU's
memory management unit moves data in 4096-byte chunks. The disk reads and
writes in 4096-byte chunks. If we match our page size to theirs, everything
aligns perfectly and nothing is wasted.

But forget hardware for now. Just think: **a page is a box that holds
exactly 4096 bytes.**


---

# Chapter 2: A Page Is Just an Array

In Rust or C#, a page is literally an array:

```rust
let page: [u8; 4096] = [0; 4096];   // 4096 bytes, all zeros
```

That's a page. An array of 4096 bytes. You can read any byte:

```rust
let value = page[0];      // read byte 0
let value = page[100];    // read byte 100
let value = page[4095];   // read byte 4095 (the last one)
```

You can write any byte:

```rust
page[0] = 72;             // write the letter 'H' (ASCII 72) at byte 0
page[1] = 101;            // write 'e' at byte 1
page[2] = 108;            // write 'l' at byte 2
page[3] = 108;            // write 'l' at byte 3
page[4] = 111;            // write 'o' at byte 4
```

Now the page contains "Hello" in bytes 0-4, and zeros in bytes 5-4095.

```
  byte 0:  72  ('H')
  byte 1:  101 ('e')
  byte 2:  108 ('l')
  byte 3:  108 ('l')
  byte 4:  111 ('o')
  byte 5:  0   (empty)
  byte 6:  0   (empty)
  ...
  byte 4095: 0 (empty)
```

**That's ALL a page is.** An array you can read from and write to.
Everything else is just ORGANIZING what we put inside this array.


---

# Chapter 3: A Database File Is Just Pages Stacked Together

Your database file is a sequence of pages, one after another:

```
Database file on disk:

  Page 0:  bytes 0     - 4095     (the first 4096 bytes of the file)
  Page 1:  bytes 4096  - 8191     (the next 4096 bytes)
  Page 2:  bytes 8192  - 12287    (the next 4096 bytes)
  Page 3:  bytes 12288 - 16383    (the next 4096 bytes)
  ...
  Page N:  bytes N*4096 - (N+1)*4096-1
```

**Analogy**: A notebook where every page is exactly the same size.
Page 0 is the first page. Page 1 is the second page. Page 47 is the
48th page. You find any page by its number.

**The math is dead simple:**

```
To find page number P in the file:
  start_byte = P × 4096
  end_byte   = P × 4096 + 4095

Examples:
  Page 0:  starts at byte 0 × 4096 = 0
  Page 1:  starts at byte 1 × 4096 = 4096
  Page 5:  starts at byte 5 × 4096 = 20480
  Page 100: starts at byte 100 × 4096 = 409600
```

To read page 5 from disk:

```rust
let mut page = [0u8; 4096];
file.read_at(&mut page, 5 * 4096);  // read 4096 bytes starting at byte 20480
```

To write page 5 to disk:

```rust
file.write_at(&page, 5 * 4096);     // write 4096 bytes starting at byte 20480
```

**That's the entire I/O model.** Read a page = read 4096 bytes at a
calculated offset. Write a page = write 4096 bytes at a calculated offset.


---

# Chapter 4: Raw Pages Are Useless — We Need Structure

Right now our page is just 4096 raw bytes. If you write "Hello" at byte 0
and "World" at byte 100, how do you later know where "World" starts?
You don't — unless you add STRUCTURE.

This is like the difference between a blank piece of paper and a FORM.
A blank paper could have anything scribbled anywhere. A form has labeled
fields, boxes, and a known layout.

**We add structure by dividing the page into regions with known purposes.**


## 4.1: The Simplest Possible Page Structure

Let's start extremely simple. We'll store fixed-size records (every record
is exactly 100 bytes). No variable sizes, no complexity.

```
Page layout (fixed-size records, 100 bytes each):

  Bytes 0-3:    record_count (how many records are stored)  ← HEADER
  Bytes 4-7:    (reserved)                                  ← HEADER
  Bytes 8-107:  Record 0 (100 bytes)
  Bytes 108-207: Record 1 (100 bytes)
  Bytes 208-307: Record 2 (100 bytes)
  ...
  Bytes 3908-4007: Record 39 (100 bytes)
  Bytes 4008-4095: (unused — can't fit another 100-byte record)
```

The **header** (first 8 bytes) tells us how many records are stored.
The rest is divided into fixed-size slots.

```
How many records fit?
  Available space = 4096 - 8 (header) = 4088 bytes
  Records that fit = 4088 / 100 = 40 records (with 88 bytes wasted)
```

### Reading record N:

```rust
fn read_record(page: &[u8; 4096], record_number: u32) -> &[u8] {
    let offset = 8 + (record_number as usize * 100);  // skip header, then N records
    &page[offset..offset + 100]
}

// Read record 0: starts at byte 8
// Read record 3: starts at byte 8 + 3*100 = 308
// Read record 39: starts at byte 8 + 39*100 = 3908
```

### Writing (inserting) a new record:

```rust
fn insert_record(page: &mut [u8; 4096], data: &[u8; 100]) -> Option<u32> {
    // Step 1: Read the current record count from the header
    let count = u32::from_le_bytes(page[0..4].try_into().unwrap());
    
    // Step 2: Is there room?
    if count >= 40 {
        return None;  // Page is full!
    }
    
    // Step 3: Calculate where to put the new record
    let offset = 8 + (count as usize * 100);
    
    // Step 4: Copy the record data into the page
    page[offset..offset + 100].copy_from_slice(data);
    
    // Step 5: Increment the record count in the header
    let new_count = count + 1;
    page[0..4].copy_from_slice(&new_count.to_le_bytes());
    
    // Step 6: Return the record number we used
    Some(count)
}
```

### Deleting a record:

For fixed-size records, the simplest approach is to mark it as deleted:

```rust
fn delete_record(page: &mut [u8; 4096], record_number: u32) {
    let offset = 8 + (record_number as usize * 100);
    
    // Set the first byte to 0xFF to mark "deleted"
    // (or use a separate bitmap — we'll see that later)
    page[offset] = 0xFF;
}
```

**This is the most basic page structure.** Fixed-size records, simple math,
no complexity. But it has a problem: what if records are different sizes?


---

# Chapter 5: The Slotted Page — Handling Variable Sizes

Real database records have different sizes:

```
"Mohamed|Cairo|5000.00"           → 21 bytes
"Ahmed|Alexandria|12500.50"       → 27 bytes
"Sara|Giza|8750.25"               → 18 bytes
"Omar Mohamed|Luxor|320000.00"    → 30 bytes
```

We can't use fixed-size slots because:
- If we use 30-byte slots, short records waste space.
- If we use 18-byte slots, long records don't fit.

**The solution: the slotted page.** It separates the "directory" (where
records are) from the "data" (the actual records).

```
SLOTTED PAGE LAYOUT:

  ┌──────────────────────────────────────────────┐ byte 0
  │ HEADER:                                       │
  │   record_count = 3                            │ 2 bytes
  │   free_space_pointer = 4027                   │ 2 bytes
  │   (maybe more header fields)                  │
  ├──────────────────────────────────────────────┤ byte 8
  │ SLOT ARRAY (grows → downward):                │
  │   Slot 0: offset=4075, length=21              │ 4 bytes per slot
  │   Slot 1: offset=4048, length=27              │
  │   Slot 2: offset=4030, length=18              │
  ├──────────────────────────────────────────────┤ byte 20
  │                                              │
  │              F R E E   S P A C E              │
  │                                              │
  │   (this area shrinks as you add records)     │
  │                                              │
  ├──────────────────────────────────────────────┤ byte 4030
  │ Record 2: "Sara|Giza|8750.25"     (18 bytes) │ ← grows ↑ upward
  ├──────────────────────────────────────────────┤ byte 4048
  │ Record 1: "Ahmed|Alexandria|12500" (27 bytes) │
  ├──────────────────────────────────────────────┤ byte 4075
  │ Record 0: "Mohamed|Cairo|5000"     (21 bytes) │
  └──────────────────────────────────────────────┘ byte 4096
```

**Two things grow toward each other:**
- The SLOT ARRAY grows DOWNWARD from the header (byte 8, 12, 16, 20...)
- The RECORD DATA grows UPWARD from the bottom (byte 4075, 4048, 4030...)
- The FREE SPACE is the gap in the middle
- When they MEET, the page is full

### Why this design?

```
Problem: Records are different sizes. How do I find record 2?

With fixed slots: record 2 is at byte 8 + 2*100 = 208. Easy math.
                  But wastes space.

With slotted page: record 2's location is stored in slot 2.
                   Slot 2 says: "offset=4030, length=18"
                   So record 2 starts at byte 4030 and is 18 bytes long.

The slot array is a DIRECTORY — it tells you where each record lives.
```


---

# Chapter 6: Slotted Page — Step by Step Operations

## 6.1: Inserting a Record

Let's insert "Omar|Luxor|320000.00" (22 bytes) into our page:

```
BEFORE INSERT:
  record_count = 3
  free_space_pointer = 4030   (points to start of last record data)
  Slot array ends at byte 20  (3 slots × 4 bytes + 8 header)
  Free space = 4030 - 20 = 4010 bytes available

Step 1: Calculate the new record's position
  new_record_length = 22
  new_record_offset = free_space_pointer - 22 = 4030 - 22 = 4008
  
Step 2: Check if there's enough room
  space_needed = 22 (record) + 4 (new slot entry) = 26 bytes
  space_available = 4030 - 20 = 4010 bytes
  4010 ≥ 26? Yes! Proceed.

Step 3: Write the record data at byte 4008
  page[4008..4030] = "Omar|Luxor|320000.00"

Step 4: Write the new slot entry
  slot_offset = 8 + 3 * 4 = 20  (slot 3, each slot is 4 bytes)
  page[20..22] = 4008 (record offset, as 2-byte little-endian)
  page[22..24] = 22   (record length, as 2-byte little-endian)

Step 5: Update the header
  record_count = 4
  free_space_pointer = 4008


AFTER INSERT:
  ┌──────────────────────────────────────────────┐ byte 0
  │ HEADER: count=4, free_ptr=4008               │
  ├──────────────────────────────────────────────┤ byte 8
  │ Slot 0: offset=4075, length=21               │
  │ Slot 1: offset=4048, length=27               │
  │ Slot 2: offset=4030, length=18               │
  │ Slot 3: offset=4008, length=22  ← NEW        │
  ├──────────────────────────────────────────────┤ byte 24
  │              F R E E   S P A C E              │
  │              (now 4008 - 24 = 3984 bytes)     │
  ├──────────────────────────────────────────────┤ byte 4008
  │ Record 3: "Omar|Luxor|320000.00"  ← NEW      │
  ├──────────────────────────────────────────────┤ byte 4030
  │ Record 2: "Sara|Giza|8750.25"                │
  ├──────────────────────────────────────────────┤ byte 4048
  │ Record 1: "Ahmed|Alexandria|12500.50"         │
  ├──────────────────────────────────────────────┤ byte 4075
  │ Record 0: "Mohamed|Cairo|5000.00"             │
  └──────────────────────────────────────────────┘ byte 4096
```


## 6.2: Reading a Record

To read record 2:

```
Step 1: Find slot 2 in the slot array
  slot_offset = 8 + 2 * 4 = 16
  record_offset = page[16..18] as u16 = 4030
  record_length = page[18..20] as u16 = 18

Step 2: Read the bytes at that location
  data = page[4030..4048]  →  "Sara|Giza|8750.25"

Done! Two reads: one for the slot, one for the data.
```

```rust
fn read_record(page: &[u8; 4096], slot_number: u16) -> Option<&[u8]> {
    let count = u16::from_le_bytes(page[0..2].try_into().unwrap());
    
    if slot_number >= count {
        return None;  // Slot doesn't exist
    }
    
    let slot_offset = 8 + slot_number as usize * 4;
    let record_offset = u16::from_le_bytes(
        page[slot_offset..slot_offset + 2].try_into().unwrap()
    ) as usize;
    let record_length = u16::from_le_bytes(
        page[slot_offset + 2..slot_offset + 4].try_into().unwrap()
    ) as usize;
    
    if record_length == 0 {
        return None;  // Deleted record
    }
    
    Some(&page[record_offset..record_offset + record_length])
}
```


## 6.3: Deleting a Record

To delete record 1 ("Ahmed|Alexandria|12500.50"):

```
Step 1: Find slot 1
  slot_offset = 8 + 1 * 4 = 12

Step 2: Set the length to 0 (mark as deleted)
  page[14..16] = 0   (length = 0 means "deleted")

That's it! The physical bytes are still there, but the slot says
"this record doesn't exist anymore (length = 0)."


AFTER DELETE:
  ┌──────────────────────────────────────────────┐ byte 0
  │ HEADER: count=4, free_ptr=4008               │
  ├──────────────────────────────────────────────┤ byte 8
  │ Slot 0: offset=4075, length=21               │
  │ Slot 1: offset=4048, length=0   ← DELETED    │
  │ Slot 2: offset=4030, length=18               │
  │ Slot 3: offset=4008, length=22               │
  ├──────────────────────────────────────────────┤ byte 24
  │              F R E E   S P A C E              │
  ├──────────────────────────────────────────────┤ byte 4008
  │ Record 3: "Omar|Luxor|320000.00"             │
  ├──────────────────────────────────────────────┤ byte 4030
  │ Record 2: "Sara|Giza|8750.25"                │
  ├──────────────────────────────────────────────┤ byte 4048
  │ Record 1: ██████DEAD SPACE██████  (27 bytes) │ ← still occupies space!
  ├──────────────────────────────────────────────┤ byte 4075
  │ Record 0: "Mohamed|Cairo|5000.00"             │
  └──────────────────────────────────────────────┘ byte 4096

Problem: The 27 bytes from record 1 are wasted!
Solution: COMPACTION (see Chapter 7)
```


---

# Chapter 7: Page Compaction — Reclaiming Dead Space

After deleting records, you get "holes" in the data area. Compaction
slides all live records together, eliminating the holes:

```
BEFORE COMPACTION:
  Slot 0: offset=4075, length=21   (Mohamed — LIVE)
  Slot 1: offset=4048, length=0    (Ahmed — DELETED)
  Slot 2: offset=4030, length=18   (Sara — LIVE)
  Slot 3: offset=4008, length=22   (Omar — LIVE)

  Data area: [Omar 22B][Sara 18B][DEAD 27B][Mohamed 21B]
  Free space: 4008 - 24 = 3984 bytes
  Dead space: 27 bytes (from deleted record 1)
  Usable free space: 3984 + 27 = 4011 bytes (if we compact)


COMPACTION PROCESS:

  Step 1: Collect all LIVE records and their sizes
    Record 0: "Mohamed|Cairo|5000.00"    21 bytes
    Record 2: "Sara|Giza|8750.25"        18 bytes
    Record 3: "Omar|Luxor|320000.00"     22 bytes

  Step 2: Rewrite them contiguously from the bottom of the page
    byte 4075: "Mohamed|Cairo|5000.00"      (21 bytes)
    byte 4057: "Sara|Giza|8750.25"          (18 bytes)
    byte 4035: "Omar|Luxor|320000.00"       (22 bytes)

  Step 3: Update the slot array with new offsets
    Slot 0: offset=4075, length=21   (unchanged — it was already at the bottom)
    Slot 1: offset=0,    length=0    (still deleted)
    Slot 2: offset=4057, length=18   (MOVED from 4030 to 4057)
    Slot 3: offset=4035, length=22   (MOVED from 4008 to 4035)

  Step 4: Update free_space_pointer
    free_space_pointer = 4035  (was 4008)


AFTER COMPACTION:
  ┌──────────────────────────────────────────────┐ byte 0
  │ HEADER: count=4, free_ptr=4035               │
  ├──────────────────────────────────────────────┤ byte 8
  │ Slot 0: offset=4075, length=21               │
  │ Slot 1: offset=0,    length=0   (deleted)    │
  │ Slot 2: offset=4057, length=18               │
  │ Slot 3: offset=4035, length=22               │
  ├──────────────────────────────────────────────┤ byte 24
  │                                              │
  │       F R E E   S P A C E                     │
  │       (now 4035 - 24 = 4011 bytes!)           │
  │       (was 3984 — we recovered 27 bytes)      │
  │                                              │
  ├──────────────────────────────────────────────┤ byte 4035
  │ Record 3: "Omar|Luxor|320000.00"             │
  ├──────────────────────────────────────────────┤ byte 4057
  │ Record 2: "Sara|Giza|8750.25"                │
  ├──────────────────────────────────────────────┤ byte 4075
  │ Record 0: "Mohamed|Cairo|5000.00"             │
  └──────────────────────────────────────────────┘ byte 4096

★ The dead space is gone! Records are packed tightly.
★ Slot numbers did NOT change — slot 0 is still slot 0, slot 2 is still slot 2.
★ Only the offsets inside the slots changed.
★ External references (like B+ tree entries that say "page 5, slot 2")
  still work perfectly — slot 2 just points to a different byte offset now.
```

**When to compact:**
- When you need to insert but free_space < needed BUT free_space + dead_space ≥ needed
- NOT on every delete (too expensive)
- NOT before checking if the total space is sufficient (if even after compaction it won't fit, don't bother)


---

# Chapter 8: How the Database Decides WHICH Page to Insert Into

This is one of the most confusing parts: you have a file with 1000 pages.
A new record arrives. Which page do you put it in?

## 8.1: The Heap File — Just Find Any Page With Space

The simplest approach is called a **heap file** — records go in any page
that has room. No sorting, no ordering.

```
Strategy 1: LINEAR SCAN (simplest, slowest)

  For each page from 0 to N:
    Read the page
    Check free_space
    If free_space ≥ record_size + 4 (slot overhead):
      Insert here!
      Break.
  
  If no page has room:
    Allocate a new page (grow the file)
    Insert into the new page.

Problem: Scanning 1000 pages to find one with space is SLOW.
```

```
Strategy 2: FREE SPACE MAP (what databases actually use)

  Keep a separate "directory" that tracks how much free space
  each page has. This directory is itself stored in pages!

  Free Space Map (Page 0 or a dedicated page):
  ┌──────────────────────────────────────────┐
  │ Page 1:  3800 bytes free                  │
  │ Page 2:  0 bytes free (FULL)              │
  │ Page 3:  2200 bytes free                  │
  │ Page 4:  4080 bytes free (nearly empty)   │
  │ Page 5:  150 bytes free                   │
  │ ...                                       │
  └──────────────────────────────────────────┘

  To insert a 500-byte record:
    Scan the free space map (small — one byte per page is enough)
    Find first page with ≥ 504 bytes (500 + 4 for slot)
    → Page 1 has 3800 bytes. Use page 1!

  The free space map entry for each page can be approximate:
    0 = full (0 bytes free)
    1 = nearly full (1-500 bytes free)
    2 = half full (500-2000 bytes free)
    3 = mostly empty (2000-3500 bytes free)
    4 = empty (3500+ bytes free)

  One byte per page → 1000 pages need only 1000 bytes → fits in ONE page!
```

```
Strategy 3: LINKED LIST OF FREE PAGES (used by PostgreSQL)

  PostgreSQL maintains a "Free Space Map" (FSM) that uses a binary
  tree structure to quickly find a page with enough free space.

  But the principle is the same: don't scan every page.
  Keep metadata about which pages have space.
```

## 8.2: B+ Tree Pages — Not a Choice, A Calculation

For B+ tree index pages, you DON'T choose which page to insert into.
The tree structure determines it:

```
To insert key=42:

  Step 1: Start at the root page
  Step 2: Follow the tree:
    Root says: keys < 50 → go to child page 7
  Step 3: Page 7 is a leaf. Insert key=42 here.

  If page 7 is full → SPLIT it into two pages.
  You don't get to "choose" a different page.
  The tree structure dictates where every key goes.
```


---

# Chapter 9: Page Types — Different Pages for Different Jobs

Not all pages have the same internal layout. Your database has several
types of pages, each with its own structure:

```
PAGE TYPE 0: META PAGE (page 0 of the file)
  ┌──────────────────────────────────────┐
  │ magic number (identifies the file)    │ 4 bytes
  │ version number                        │ 4 bytes
  │ total page count                      │ 4 bytes
  │ free list head                        │ 4 bytes
  │ root B+ tree page number             │ 4 bytes
  │ (more metadata)                       │
  │ (unused space — zeros)                │
  └──────────────────────────────────────┘
  Only ONE meta page exists (page 0). It's the "table of contents."


PAGE TYPE 1: HEAP DATA PAGE (stores table records)
  ┌──────────────────────────────────────┐
  │ page type = 1                         │ 2 bytes
  │ record count                          │ 2 bytes
  │ free space pointer                    │ 2 bytes
  │ next page with free space             │ 4 bytes
  │ [slot array]                          │ grows down
  │ [free space]                          │
  │ [record data]                         │ grows up
  └──────────────────────────────────────┘
  Many of these. This is where your actual data rows live.


PAGE TYPE 2: B+ TREE INTERNAL NODE
  ┌──────────────────────────────────────┐
  │ page type = 2                         │ 2 bytes
  │ key count                             │ 2 bytes
  │ parent page number                    │ 4 bytes
  │ [key 0] [key 1] [key 2] ...          │ keys
  │ [child page 0] [child page 1] ...    │ child pointers
  │ (unused space)                        │
  └──────────────────────────────────────┘
  These are the "signpost" pages in the B+ tree.


PAGE TYPE 3: B+ TREE LEAF NODE
  ┌──────────────────────────────────────┐
  │ page type = 3                         │ 2 bytes
  │ key count                             │ 2 bytes
  │ parent page number                    │ 4 bytes
  │ next leaf page number                 │ 4 bytes  ← linked list!
  │ prev leaf page number                 │ 4 bytes
  │ [key 0, value 0]                      │ key-value pairs
  │ [key 1, value 1]                      │
  │ [key 2, value 2]                      │
  │ (unused space)                        │
  └──────────────────────────────────────┘
  These are the "data" pages of the B+ tree.
  Values are Record IDs (page_id, slot_number) pointing to heap pages.


PAGE TYPE 4: FREE PAGE (unused, on the free list)
  ┌──────────────────────────────────────┐
  │ next free page number                 │ 4 bytes
  │ (rest is garbage/zeros — doesn't matter)
  └──────────────────────────────────────┘
  These pages are not in use. They form a linked list
  so we can quickly find an available page when needed.
```

**How do you know a page's type?** Read the first 2 bytes. Every page
starts with a type code. When the page manager reads a page from disk,
the first thing it checks is the type code to know how to interpret
the rest of the bytes.


---

# Chapter 10: A Complete Example — From INSERT to Disk

Let's trace a complete database INSERT operation, showing exactly which
pages are involved and how bytes move:

```
SQL: INSERT INTO accounts VALUES (1005, 'Youssef', 'Aswan', 15000.00)

The record bytes: "1005|Youssef|Aswan|15000.00" = 28 bytes
```

### Step 1: Find a Heap Page With Space

```
Read the free space map (or scan pages).
Page 7 has 2000 bytes free. Use page 7.

Read page 7 from disk (or buffer pool):
  file.read_at(&mut page, 7 * 4096)  →  4096 bytes into memory
```

### Step 2: Insert the Record Into the Heap Page

```
Page 7 currently has 15 records.
free_space_pointer = 3200

New record goes at: 3200 - 28 = 3172
New slot goes at: 8 + 15 * 4 = 68

Write to the page (in memory):
  page[3172..3200] = "1005|Youssef|Aswan|15000.00"   (record data)
  page[68..70] = 3172   (slot offset)
  page[70..72] = 28     (slot length)
  page[0..2] = 16       (record count: 15 → 16)
  page[2..4] = 3172     (free_space_pointer updated)

The record now has an address: RID(page=7, slot=15)
```

### Step 3: Insert the Key Into the B+ Tree Index

```
We also need to add an index entry so we can find this record by key.
Key = 1005, Value = RID(7, 15)

Search the B+ tree for where 1005 should go:
  Read root page (page 1): keys=[500, 1500]
    1005 ≥ 500 and 1005 < 1500 → go to child page 4
  
  Read page 4 (leaf): keys=[502, 780, 990]
    1005 should go after 990.
    Page 4 has room → insert here.

Update page 4 (in memory):
  keys become: [502, 780, 990, 1005]
  values become: [RID(3,2), RID(5,8), RID(7,1), RID(7,15)]
  key_count: 3 → 4
```

### Step 4: Write the WAL Records (Before Flushing Pages!)

```
WAL record 1: "UPDATE page 7, offset 3172, old=zeros, new=record data"
WAL record 2: "UPDATE page 4, offset X, old=old_keys, new=new_keys"
WAL record 3: "COMMIT"

Write WAL to disk: wal_file.write_all(&wal_records)
fsync the WAL: wal_file.sync_all()

★ NOW the transaction is committed (WAL is on disk).
★ Pages 7 and 4 are modified in memory but NOT yet on disk.
```

### Step 5: Eventually Flush Dirty Pages to Disk

```
At checkpoint time (or when the buffer pool needs to evict these pages):

  Write page 7 to disk: file.write_at(&page7, 7 * 4096)
  Write page 4 to disk: file.write_at(&page4, 4 * 4096)
  fsync the data file: file.sync_all()

Now the data is on disk in both the WAL and the data file.
Old WAL records can be discarded after checkpoint.
```

### Summary of All Pages Involved in One INSERT:

```
Pages READ:
  Page 0 (meta) — to find the B+ tree root
  Page 1 (B+ tree root) — to start the index search
  Page 4 (B+ tree leaf) — the leaf where our key belongs
  Page 7 (heap data) — the page where we insert the record
  (Free space map page — to find page 7)

Pages MODIFIED:
  Page 4 (B+ tree leaf) — added key 1005
  Page 7 (heap data) — added the record

Pages WRITTEN TO WAL:
  WAL gets records describing changes to pages 4 and 7

Pages FLUSHED TO DISK (at checkpoint):
  Page 4 and page 7

Total: read 4-5 pages, modify 2 pages, write 2 WAL records.
For a database with 1 million records and a 3-level B+ tree,
that's 3-4 page reads + 2 page writes. Fast!
```


---

# Chapter 11: The Page Lifecycle

```
BIRTH:
  Page is allocated from the free list (or file grows)
  → All bytes are zero
  → Page type is set in the header

ACTIVE LIFE:
  Records are inserted → slots and data grow toward middle
  Records are read → slot lookup → data read
  Records are deleted → slot length set to 0
  Compaction → dead space reclaimed

  The page lives in the buffer pool (RAM) most of the time.
  It's written to disk periodically (checkpoint) or when evicted.

DEATH:
  All records deleted + compacted → page is empty
  Page is returned to the free list
  → First 4 bytes become "next free page" pointer
  → Page type becomes "FREE"
  → Page will be reused for the next allocation
```


---

# Chapter 12: Summary — The Mental Model

```
A PAGE is:
  → 4096 bytes (an array)
  → with a HEADER that describes the page type and contents
  → with a KNOWN LAYOUT so we can find data by position

A SLOTTED PAGE is:
  → a page with a SLOT ARRAY at the top (growing down)
  → and RECORD DATA at the bottom (growing up)
  → slots map record_number → byte_offset + length

READING is:
  → read the slot → get the offset → read the data at that offset

INSERTING is:
  → find a page with space (free space map)
  → write record data at the free_space_pointer (from the bottom up)
  → write a new slot entry (from the top down)
  → update the header

DELETING is:
  → set the slot's length to 0 (logical delete)
  → optionally compact later (physical reclaim)

PAGE SELECTION (which page to insert into):
  → Heap pages: any page with enough free space (use free space map)
  → B+ tree pages: determined by the tree structure (follow the keys)

EVERYTHING IS JUST BYTE MANIPULATION:
  → Reading a u16 at offset 12: u16::from_le_bytes(page[12..14])
  → Writing a u16 at offset 12: page[12..14].copy_from_slice(&value.to_le_bytes())
  → Copying a record: page[offset..offset+len].copy_from_slice(data)
  → There is no magic. It's arrays of bytes with conventions about
    which bytes mean what.
```
