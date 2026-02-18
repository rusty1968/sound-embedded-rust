# The Boundary: Safe vs Unsafe

If the Abstract Machine is the "law" of Rust, the **Safe/Unsafe Boundary** is the "border control". It is the single most important architectural concept in Rust, especially for kernel development.

In a pure C kernel, safety is a global property. A buffer overflow in the networking stack can corrupt the scheduler's run queue. Every line of code must be trusted.

In a Rust kernel, we strive to make safety a **modular** property. We want to be able to say: "If there is a memory corruption bug, it *must* originate from within one of these 5 modules that use `unsafe`, because the other 50 modules contain only safe code."

This containment strategy relies entirely on correctly implementing the Safe/Unsafe pointer boundary.

## The Core Philosophy

The relationship between Safe and Unsafe Rust can be summarized in two rules:

1.  **Unsafe Code** has the authority to violate the compiler's temporary checks (like dereferencing a raw pointer) but takes on the responsibility of maintaining the Abstract Machine's invariants.
2.  **Safe Code** relies on those invariants being true. If Safe code can cause Undefined Behavior (UB), the bug is **always** in the `unsafe` block that permitted it (or the `unsafe` code that set up the invalid state).

### The Invariant Pattern

The most common pattern in kernel development is transforming a **Precondition** into an **Invariant**.

*   **Precondition**: A requirement the caller must satisfy before calling a function (e.g., "this pointer must not be null").
*   **Invariant**: A truth that is always held by a data structure (e.g., "this struct's internal pointer is never null").

The goal of a safe abstraction is to capture unsafe preconditions and wrap them in a type that enforces invariants.

#### Example: The Non-Null Pointer

Consider a raw pointer `*mut u8`. Dereferencing it is `unsafe` because it has a precondition: the pointer must be valid.

```rust
// Unsafe: Caller must guarantee 'p' is valid
unsafe fn write_byte(p: *mut u8, val: u8) {
    *p = val;
}
```

To make this safe, we can create a type `OwnedBuf` that manages the pointer.

```rust
pub struct OwnedBuf {
    ptr: *mut u8,
    len: usize,
}

impl OwnedBuf {
    // Unsafe constructor: We trust the caller here.
    pub unsafe fn new(ptr: *mut u8, len: usize) -> Self {
        Self { ptr, len }
    }

    // Safe method: We enforce the invariant here.
    pub fn write_at(&mut self, offset: usize, val: u8) {
        if offset < self.len {
            unsafe {
                // SAFETY: We checked bounds, and our type invariant guarantees
                // 'ptr' is valid for 'len' bytes.
                self.ptr.add(offset).write(val);
            }
        } else {
            panic!("index out of bounds");
        }
    }
}
```

Code using `OwnedBuf` cannot cause UB. It might panic if it accesses out of bounds, but it cannot write to random memory. The unsafety is **encapsulated**.

## Trust Anchors

In embedded development, our invariants often come not from other software, but from hardware.

When we define a peripheral driver, we are asserting:
1.  This memory address (e.g., `0x4000_1000`) actually exists.
2.  Reading this address won't hang the CPU.
3.  The bits we read have the meaning the datasheet says they do.

We call these **Trust Anchors**. They are the axioms of our system.

### Defining a Safe MMIO Wrapper

Let's look at how we might wrap a hardware register.

```rust
// The raw, dangerous reality
const UART0_DR: *mut u32 = 0x4001_0000 as *mut u32;

// The Safe Abstraction
pub struct Uart {
    base: *mut u32,
}

impl Uart {
    /// # Safety
    /// Caller must ensure that `base` is the valid address of a UART peripheral
    /// and that no other reference to this memory exists.
    pub unsafe fn new(base: *mut u32) -> Self {
        Self { base }
    }

    /// Sends a byte. This is safe to call!
    pub fn send(&mut self, byte: u8) {
        unsafe {
            // SAFETY: We rely on the invariant established in `new`.
            // We assume the hardware is powered on and ready.
            self.base.write_volatile(byte as u32);
        }
    }
}
```

Notice the `# Safety` doc comment on `new`. This is part of the contract. The `unsafe` keyword on `new` acts as a fence. Once you are inside the `Uart` struct (having successfully called `new`), the rest of the execution is assumed safe.

## Leaking Safety

A common mistake is creating "Safe" wrappers that actually leak internal unsafety.

**Bad Example:**

```rust
pub struct BadBuffer {
    ptr: *mut u8,
}

impl BadBuffer {
    // This looks safe, but returns a raw pointer!
    pub fn as_ptr(&self) -> *mut u8 {
        self.ptr
    }
}
```

If the user takes that raw pointer and writes to it while `BadBuffer` is dropped (and frees the memory), we have a Use-After-Free.

**Good Example:**

```rust
impl GoodBuffer {
    // Returns a reference, bounded by the lifetime of 'self'
    pub fn as_slice(&self) -> &[u8] {
        unsafe { core::slice::from_raw_parts(self.ptr, self.len) }
    }
}
```

By returning `&[u8]`, we tie the lifetime of the return value to `&self`. The Rust borrow checker now enforces that the buffer cannot be freed while the slice is being used. We have used the type system to enforce our invariant.

## Lab: Plugging the Leak

In this lab, you will identify a safety violation in a seemingly robust abstraction.

### The Scenario

We are designing a DMA (Direct Memory Access) controller wrapper. We want to give the DMA a buffer to write into.

```rust
pub struct DmaTransfer {
    buffer: Vec<u8>,
}

impl DmaTransfer {
    pub fn start(mut buffer: Vec<u8>) -> Self {
        let ptr = buffer.as_mut_ptr();
        let len = buffer.len();
        
        // Imagine we allow the hardware to write to 'ptr' here...
        unsafe { start_dma_hardware(ptr, len); }

        Self { buffer }
    }
}

impl Drop for DmaTransfer {
    fn drop(&mut self) {
        // Stop the hardware when the struct is dropped
        unsafe { stop_dma_hardware(); }
    }
}
```

### The Flaw

Consider this usage:

```rust
{
    let mut data = vec![0; 1024];
    let transfer = DmaTransfer::start(data);
    
    // ... do other work ...
    
    std::mem::forget(transfer); // Uh oh.
}
```

If the user calls `std::mem::forget(transfer)`, the `Drop` implementation is **never called**. The DMA hardware continues writing to the memory address.

However, the `Vec<u8>` inside `transfer` *is* conceptually gone (even if the memory technically leaks, the variable is out of scope). If `data` happened to be on the stack (if we weren't using `Vec` but a stack array), or if we reused that memory region, the DMA would corrupt unrelated memory.

Wait, `mem::forget` is **Safe**.

**The Rule**: `Drop` is not a safety invariant. You cannot rely on `Drop` running to ensure memory safety.

### The Fix

To make DMA safe, we typically need to use:
1.  **Ownership**: Pass the buffer into the Transfer and hold it until the hardware is done.
2.  **'static lifetimes**: Only allow buffers that live forever (or use a scoped thread API).
3.  **Leak-check**: Or accept that leaking the `Transfer` struct fundamentally leaks the memory (which is safe), but we must ensure the *hardware* doesn't write to freed memory.

In the case above, `mem::forget(transfer)` leaks the `Vec`'s memory too (calling `forget` on the struct forgets its fields). So the memory is never freed, and the DMA continues writing to allocated (but leaked) memory. This is actually... safe! (Leaking memory is safe in Rust).

**But wait**, what if we did this with a stack buffer?

```rust
struct StackDma<'a> {
    buf: &'a mut [u8],
}
// ...
```

If we `mem::forget` a `StackDma`, the struct is forgotten, but the *stack frame* containing the buffer will eventually pop when the function returns. The DMA will then be writing to a popped stack frame. **That is immediate, catastrophic UB.**

**Lesson**: You cannot rely on `Drop` to turn off hardware if the hardware has a pointer to stack memory. This is known as the **Leak Apocalypse** problem in Rust (pre-1.0).

## Summary

*   **The Boundary** is where we assert "Trust me, I checked the preconditions."
*   **Encapsulation** transforms loose preconditions into tight type invariants.
*   **Safe APIs** must not cause UB, even if the user deliberately tries to misuse them (unless they use `unsafe` themselves).
*   **Drop is not guaranteed**: Never rely on destructors for memory safety.
