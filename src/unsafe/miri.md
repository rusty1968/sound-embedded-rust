# Miri: The Abstract Machine Interpreter

In previous chapters, we defined the **Rust Abstract Machine (AM)** as the theoretical computer that your code targets. We learned that **Undefined Behavior (UB)** is simply the state where this machine gets "stuck".

But a theoretical machine is hard to debug. You can't attach GDB to a theory.

**Miri** is the solution. It is an interpreter that executes your Rust code directly on an implementation of the Abstract Machine.

## Why Miri Matters

If your code triggers UB on real hardware, it might crash, or it might silently corrupt memory, or it might run perfectly fine today and break tomorrow.

If your code triggers UB in Miri, **Miri halts and tells you exactly why.**

### Detecting Subtle UB

Consider this function which tries to create two mutable references to the same data:

```rust
fn duplicate_mut(x: &mut i32) -> (&mut i32, &mut i32) {
    let ptr = x as *mut i32;
    // SAFETY: This is UB! The Abstract Machine forbids 
    // two active &mut to the same data.
    unsafe { (&mut *ptr, &mut *ptr) }
}
```

On real hardware, this compiles and runs. The CPU doesn't care.
In Miri, this crashes with: `Undefined Behavior: trying to reborrow for Unique ... but the parent tag is not in the borrow stack`.

## Stacked Borrows & Tree Borrows

Miri implements a memory model called **Stacked Borrows** (and the newer experimental **Tree Borrows**).

Think of every memory location as having a stack of permissions.
1.  When you create `&mut T`, you push a "Unique" permission onto the stack.
2.  When you use that reference, it must be at the top of the stack.
3.  If you use an older pointer (lower in the stack), it pops everything above it. The popped permissions are now invalid.

This model explains why you can't sneakily use a raw pointer alias while a `&mut` reference exists. The `&mut` asserts exclusivity; the raw pointer access invalidates that exclusivity.

## Strict Provenance

Pointers are complicated. In C, `u32` (address) and `*u32` (pointer) are often interchangeable. In the Rust AM, they are fundamentally different.

**Strict Provenance** is an experimental model that formalizes this. It states: **"You cannot create a valid pointer just from an integer."**

```rust
let x = 42;
let ptr = &x as *const i32;
let addr = ptr as usize; // cast to integer

// In Strict Provenance, 'addr' has lost its provenance (link to 'x').
// Casting it back to a pointer creates a wild pointer with no permission to access 'x'.
let bad_ptr = addr as *const i32; 

unsafe { *bad_ptr }; // UB in strict provenance!
```

**The Kernel Fix**: Use `ptr::with_addr` (or `map_addr`) to carry provenance.

```rust
// 'new_ptr' has the address of 'addr' but the provenance of 'ptr'.
let new_ptr = ptr.with_addr(addr); 
```

This is critical for linked lists or page tables where we mask bits in pointers.

## Verifying Kernel Code with Miri

Miri cannot run your entire kernel because it doesn't support inline assembly or MMIO accesses (it doesn't simulate an ARM CPU). However, substantial parts of a kernel are **pure logic**:
-   Linked Lists / Queues
-   Allocators (buddy systems, slabs)
-   Page Table walkers (logic only)
-   Synchronization primitives (Spinlocks, Mutexes)

**Strategy**:
Extract these data structures into separate crates or modules. Write unit tests for them. Run those tests with `cargo +nightly miri test`.

If Miri accepts your linked list implementation, you can be highly confident it is sound, regardless of the architecture.
