# The Rust Abstract Machine

When writing embedded systems code, specifically kernels or drivers that interact directly with hardware, it is tempting to think about the code in terms of the assembly instructions it generates and how the CPU executes them. We imagine registers, caches, physical memory addresses, and the program counter steps through our logic one cycle at a time.

This "hardware mental model" is useful for debugging and performance analysis, but it is **insufficient and dangerous** for reasoning about correctness in Rust (or C/C++).

Instead, Rust code executes on an **Abstract Machine (AM)**.

> **Note on "The" Abstract Machine**:
> Rust does not yet have a single, finalized formal specification for its Abstract Machine (unlike C or C++). Even the memory model is a work in progress. When we speak of the "Abstract Machine" in this book, we refer to the **consensus model** used by the Operational Semantics (OpSem) team, enforced by tools like **Miri** (via Stacked Borrows/Tree Borrows), and relied upon for LLVM codegen optimizations. It is a rigorous mental model, even if the ink on the official spec is not yet dry.

The compiler does not translate your source code directly into machine code; it translates your source code into operations on this Abstract Machine. Understanding this layer of abstraction is the key to understanding Undefined Behavior (UB), `unsafe` code, and why the compiler is allowed to optimize your code in ways that might seem surprising.

## Philosophy: Why an Abstract Machine?

Why do we need this intermediate layer? Why doesn't the compiler just map `let x = 5` to a `MOV` instruction?

The primary reason is **optimization**.

If the language semantics were defined strictly in terms of the underlying hardware, the compiler would be handcuffed. It would have to emit instructions that exactly mimic the source code's structure. For example, if you write two independent reads from memory, a strict hardware model might require them to happen in that exact order.

However, real CPUs are complex. Memory latencies vary, pipelines stall, and superscalar architectures can execute multiple instructions at once. To achieve high performance, compilers need the freedom to:
*   Reorder instructions.
*   Eliminate redundant reads or writes.
*   Assume certain conditions hold true (like loop invariants).

The Abstract Machine serves as a contract between the programmer and the active compiler. The contract states: "*As long as your program has defined behavior on the Abstract Machine, the compiler guarantees the generated machine code will have the same observable effects.*"

Crucially, if your program violates the rules of the Abstract Machine, the contract is void. The compiler is no longer bound to produce meaningful code.

## The State of the Machine

The Rust Abstract Machine tracks more state than a physical CPU does. While a CPU sees bytes in RAM and values in registers, the AM sees *typed values*, *initialization*, and *provenance*.

### 1. Memory (Bytes + Initialization + Validity)
In the Abstract Machine, a byte of memory is not just a value from 0-255. A byte of memory has two key properties:
1.  **Definitions**: Is it initialized?
2.  **Validity**: Does it represent a valid value for its type?

**Uninitialized Memory**: Reading uninitialized memory on a physical CPU usually just gives you garbage data (whatever was left in that cache line). Reading uninitialized memory on the Abstract Machine is immediate Undefined Behavior. `MaybeUninit<T>` exists precisely to handle this state without triggering UB.

**Invalid Representations**: Even if memory is initialized, it might be "bit-invalid" for a specific type.
*   A `bool` must be `0x00` or `0x01`.
*   A `&T` reference must never be NULL.
*   An `enum` must match one of its defined discriminants.

If you transmute `3u8` into a `bool`, you have created an invalid value. The Abstract Machine considers this UB immediately, even if you never use the value.

### 2. Pointer Values (Address + Provenance)
To a CPU, a pointer is just an integer—a 32-bit or 64-bit number representing a spot in memory. You can do arithmetic on it, cast it to an integer, and cast it back.

To the Abstract Machine, a pointer is **Address + Provenance**.

**Provenance** is the history of where that pointer came from. It tracks which *allocation* the pointer is allowed to access.
*   If you allocate variable `x` and variable `y`.
*   You take a pointer to `x` (`&x`).
*   You cast it to an integer, add the offset to reach `y`, and cast it back to a pointer.
*   Can you read `y` through this pointer?

On hardware: Yes, it's just an address.
On the Abstract Machine: **No**. The pointer's provenance is still "tied" to `x`. Accessing `y` through a pointer with `x`'s provenance is UB. This rule allows the compiler to assume that pointers to different allocations do not alias, enabling aggressive optimization (similar to `restrict` in C).

*(Note: If you work with low-level kernel structures where pointers must travel through pure `usize` (e.g., in a page table entry), you must use "Exposed Provenance" APIs like [`core::ptr::from_exposed_addr`](https://doc.rust-lang.org/core/ptr/fn.from_exposed_addr.html).)*

```rust
use core::ptr::{from_exposed_addr, addr_of};

let x: i32 = 0;
// Converts pointer to address and "exposes" it to the world
let addr = addr_of!(x) as usize; 

// Reconstruct a pointer with exposed provenance if you really must
let p = from_exposed_addr::<i32>(addr);
// Warning: This is an advanced escape hatch. 
// Do not use this unless you understand the performance implications.
```

### 3. The Borrow Stack (Stacked Borrows / Tree Borrows)
This is the most strictly "Rust" part of the Abstract Machine. To enforce the rules of references (`&mut T` must be exclusive, `&T` must be shared-read-only), tools like Miri use models like **Stacked Borrows** (the current default) or **Tree Borrows** (an experimental alternative proposal). These are not yet fully standardized in the language spec, but they are the de facto models used to reason about aliasing.

Every time you create a reference, the AM pushes a new "permission item" onto the stack for that memory location.
*   **Accessing memory** requires the tag associated with your pointer to be live in the stack.
*   **Popping items**: If you write to memory using an older pointer (lower in the stack), it "pops" everything above it off the stack. Those popped pointers are now invalidated. Using them later is UB.

This model explains why you cannot use a raw pointer to access data while a specialized `&mut` reference to that data exists, even if the raw pointer has the correct address.

## Operations

The execution of a Rust program is a sequence of operations on this state.

### Reads and Writes
Every read or write is checked against the current state:
1.  **Alignment check**: Is the pointer aligned?
2.  **Initialization check**: (For reads) Is the memory initialized?
3.  **Provenance check**: Does this pointer have the authority to access this *allocation* (e.g., is it within the bounds of the original object it derived from)?
4.  **Permission check**: Does this pointer have the current *borrowing authority* to access this memory (i.e., is its tag live in the Borrow Stack for this specific operation)?

### Retagging
"Retagging" is an operation that happens implicitly when you create specific references or cast pointers. When you do `let y = &mut *x;`, you aren't just copying an address. The Abstract Machine is **retagging**: it generates a new unique ID (tag) for `y`, pushes it onto the permission stack for that memory, and establishes `y` as the current active owner.

## Defining Undefined Behavior

We can now define Undefined Behavior (UB) rigorously.

**Undefined Behavior is the unexpected state where the Abstract Machine gets "stuck".**

If the program attempts an operation that the Abstract Machine assumes cannot happen (like popping an empty stack, or reading uninitialized memory), the machine has no defined transition to a next state. It crashes.

Because the compiler assumes the AM never gets stuck, it assumes UB never happens. If your code contains a path that leads to UB, the compiler is allowed to assume that path is never taken. It can delete the code entirely, or reorder instructions assuming the constraints (like non-aliasing) are never violated.

When the CPU executes the resulting "optimized" code, it creates the chaotic, disjointed reality we associate with UB bugs.

## Summary

*   **You code against the Abstract Machine, not the CPU.**
*   **Pointers are not just integers**; they have provenance (history/permission).
*   **Memory is not just bytes**; it tracks initialization state.
*   **UB is a stuck state** in the AM, which effectively breaks the contract with the compiler.

In the next chapter, we will look at **Miri**, an interpreter for Rust's Abstract Machine that can detect when your program causes the machine to get stuck, effectively catching UB that mere testing would miss.

## Lab: Breaking the Abstract Machine

To solidify these concepts, let's examine a piece of code that *appears* correct if you only consider the underlying hardware, but violates the Abstract Machine's rules.

### Challenge

Consider the following function. It takes a mutable reference `r` and a raw pointer `p` to the same integer. It writes through the raw pointer and then reads through the reference.

```rust
fn naive_alias(r: &mut i32, p: *mut i32) -> i32 {
    unsafe {
        *p = 42; // Write through raw pointer
        *r       // Read through exclusive reference
    }
}

fn main() {
    let mut x = 0;
    let r = &mut x;
    let p = r as *mut i32;
    // Calling naive_alias(r, p) is UB
}
```

The compiler sees `r` is `&mut i32`. The definition of `&mut` guarantees it is **exclusive**. Therefore, the compiler assumes *nothing else* can write to the memory `r` points to. It optimizes the function to return the *original* value of `*r` (cached in a register), effectively ignoring the write to `*p`.

### Task

Using the concepts of **Stacked Borrows** and **Provenance**, explain *why* the Abstract Machine gets "stuck" (or why the behavior is undefined).

1.  **State**: When `naive_alias` is called, what permissions does `r` have? What about `p`?
2.  **Conflict**: What happens to the borrow stack when `*p = 42` is executed? Does it invalidate `r`?
3.  **Optimizations**: Why does the compiler feel safe ignoring the write to `*p`?

### Solution

<details>
<summary>Click to reveal</summary>

**Explanation**:

This is a violation of the **Unique/Exclusive** contract of `&mut T`.

1.  **Aliasing Rules**: Rust's Abstract Machine asserts that `&mut T` is unique and exclusive.
2.  **Stacked Borrows (The Popping Rule)**:
    *   When `r` is created/passed, it is pushed onto the borrow stack: `[..., Unique(p), Unique(r)]`. `r` is now the active borrower.
    *   When we write to `*p = 42`, we are using the permission `Unique(p)` which is *below* `r` in the stack.
    *   **The Check**: To use `Unique(p)`, the AM must pop everything above it. It pops `Unique(r)`.
    *   The stack is now: `[..., Unique(p)]`. `r` is invalid.
3.  **Stuck State**: When the code later tries to use `r` (in `*r`), the Abstract Machine checks the stack for `Unique(r)`. It is gone. The machine has no valid state transition—it is "stuck", resulting in UB.

**The Fix**:
Do not mix `&mut` references and raw pointers to the same data in the same scope if you intend to mutate via the raw pointer. If you need shared mutability, use `UnsafeCell<T>`.

```rust
use std::cell::UnsafeCell;

fn correct_alias(c: &UnsafeCell<i32>) -> i32 {
    unsafe {
        let p = c.get();
        *p = 42;    // Write through raw pointer obtained from UnsafeCell
        *p          // Read back
    }
}
```

`UnsafeCell` effectively tells the compiler (and the Abstract Machine): "This data is being shared and mutated. Do not assume exclusive access, and do not apply `noalias` optimizations."

</details>

## References & Further Reading

For those who want to explore the mathematical rigor behind the Abstract Machine. Note that the **OpSem Team** decisions are often the most up-to-date source of truth, superseding older UCG discussions.

1.  **[OpSem Team Decisions (FCPs)](https://github.com/rust-lang/opsem-team/issues?q=label%3Afinal-comment-period+is%3Aclosed)**: The binding decisions made by the Operational Semantics team. This is the "Supreme Court" for Undefined Behavior.
2.  **[Unsafe Code Guidelines (UCG)](https://rust-lang.github.io/unsafe-code-guidelines/)**: The broader, work-in-progress specification. Valuable, but check OpSem for recent rulings.
3.  **[RFC 3559: Strict Provenance](https://rust-lang.github.io/rfcs/3559-rust-has-provenance.html)**: The formal proposal to codify the pointer provenance model.
4.  **[MiniRust](https://github.com/RalfJung/minirust)**: The executable specification of the Rust Abstract Machine. If Miri is the linter, MiniRust is the law book.
5.  **[Stacked Borrows](https://plv.mpi-sws.org/rustbelt/stacked-borrows/)**: The academic paper defining the borrowing model Miri uses.
