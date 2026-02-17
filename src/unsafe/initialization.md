# MaybeUninit & Initialization

In the embedded kernel context, we often face a dilemma: we need to allocate memory (like a buffer for a DMA transfer) but we cannot afford the CPU cycles to initialize it with zeroes or dummy data, especially if hardware is about to overwrite it immediately. Safe Rust, however, is uncompromising: **every variable must be initialized before use**.

## The Problem: Instant UB

In C, `int x;` allocates space on the stack without writing to it. The value of `x` is indeterminate (garbage). In Rust, the compiler tracks initialization. If you try to use a variable before assigning it, the compiler rejects the code.

But what if we use `unsafe` to cheat?

```rust
// ⚠️ THIS IS UNDEFINED BEHAVIOR
let x: i32 = unsafe { core::mem::uninitialized() };
```

This is **instant Undefined Behavior (UB)**, even if you never read `x`. Why? Because Rust makes validity assumptions about types. 
- A `bool` must be `0x00` or `0x01`.
- A `&T` (reference) must be non-null and aligned.
- A `Box<T>` must own a valid heap allocation.

If the garbage bits on the stack happen to be `0x05` and you interpret them as a `bool`, the compiler's optimizations (which assume `bool` is only 0 or 1) will explode. For integers like `i32`, specific bit patterns aren't usually invalid, but the compiler is allowed to assume the value is "stable" (doesn't change without a write). Uninitialized memory might change due to hardware or OS background activity, breaking this assumption.

## The Solution: `MaybeUninit<T>`

To handle uninitialized memory safely, Rust provides `core::mem::MaybeUninit<T>`.

This type tells the compiler: "I am reserving space for a `T`, but it might contain garbage right now. Do not assume it is a valid `T`."

- It has the same size and alignment as `T`.
- It does not drop `T` when it goes out of scope (because `T` might not be initialized yet).
- It is safe to create (unless creating it requires initialization itself), but `unsafe` to read the inner value.

## Kernel Use Cases

### 1. Stack Buffers for Hardware (DMA)
We need a 1KB buffer for a UART read. Zeroing 1KB takes time. We want to pass the raw pointer to the UART driver and let the hardware fill it.

### 2. Lazy Initialization in `no_std`
Before we have synchronization primitives like `OnceLock` (or if we are implementing them), we often need a global `static` that starts uninitialized and is set up during the kernel boot process.

## The Lifecycle of `MaybeUninit`

### 1. Creation

```rust
use core::mem::MaybeUninit;

// Allocate space on the stack, but do NOT write to it.
// The content is garbage.
let mut buffer: MaybeUninit<[u8; 1024]> = MaybeUninit::uninit();

// Alternatively, if you want it zeroed (safer, but costs cycles):
let mut zeroed: MaybeUninit<[u8; 1024]> = MaybeUninit::zeroed();
```

### 2. Writing (The Safe Way)

You cannot simply assign to it like `buffer = ...` because `MaybeUninit` wraps the type.

**Pointer access is key.** You must get a raw pointer to the memory to write to it without triggering the compiler's validity checks on the inner type.

```rust
let ptr = buffer.as_mut_ptr();

unsafe {
    // Write data to the pointer. 
    // This is where we manually initialize the memory.
    ptr.write([0; 1024]); // or byte-by-byte
}
```

### 3. Transition to Initialized

Once you have guaranteed that the memory contains valid data for type `T`, you can strip the `MaybeUninit` wrapper.

```rust
unsafe {
    // Consumes the MaybeUninit and returns the T.
    // UB if the memory wasn't actually initialized!
    let data: [u8; 1024] = buffer.assume_init();
}
```

Often, we don't want to move the data (copying 1KB), but just view it as a reference:

```rust
unsafe {
    // Returns &mut [u8; 1024]
    let slice: &mut [u8; 1024] = buffer.assume_init_mut();
}
```

## Pitfalls & Deprecations

### The `mem::uninitialized` Trap
You might see old Rust code using `core::mem::uninitialized()`. **Do not use it.** It was deprecated because it was impossible to use correctly with complex types. It would return a "valid" `T` that was actually garbage. If `T` was `Box<u32>`, usage usually resulted in an immediate segfault when the drop handler tried to free a random memory address. `MaybeUninit` replaced it completely.

### The `&mut T` Reference Trap
A common mistake is trying to get a mutable reference to the uninitialized data to write to it:

```rust
let mut x = MaybeUninit::<u32>::uninit();
// ⚠️ UB: Creating a reference `&mut u32` asserts that the data 
// pointed to is ALREADY a valid u32.
let reference = unsafe { &mut *x.as_mut_ptr() }; 
*reference = 5; 
```

Technically, creating a reference to uninitialized data is undefined behavior (references must point to valid data). While this *often* works for integers, it is incorrect. **Always use raw pointers (`.as_mut_ptr()`) to write to uninitialized memory.**

## Example: Zero-Copy DMA Buffer

In high-performance drivers (like Ethernet or USB), we allocate large buffers in RAM that the hardware fills directly via Direct Memory Access (DMA). Zeroing these buffers beforehand is a waste of cycles because the hardware will immediately overwrite them.

Here is a correct pattern for creating a buffer, having "hardware" (simulated here) fill it, and then reading it safely.

```rust
use core::mem::MaybeUninit;

/// Simulates a hardware DMA controller writing to memory.
/// In a real kernel, this would be a register write to start the transfer.
unsafe fn trigger_dma_transfer(ptr: *mut u8, len: usize) {
    // Hardware writes to the memory range [ptr, ptr + len]
    for i in 0..len {
        *ptr.add(i) = 0xAA; // Hardware fills with data
    }
}

pub fn handle_packet() {
    // 1. Allocate buffer for maximum packet size (e.g., 1500 bytes for Ethernet).
    // Using MaybeUninit prevents the compiler from generating a memset(0).
    // This is critical for performance in the hot path of a driver.
    let mut packet_buffer: MaybeUninit<[u8; 1500]> = MaybeUninit::uninit();
    
    // 2. Get the pointer
    // .as_mut_ptr() returns a raw pointer (*mut [u8; 1500]), 
    // casting to *mut u8 for byte-wise access.
    let ptr = packet_buffer.as_mut_ptr() as *mut u8;
    
    // 3. Start DMA Transfer
    // We hand ownership of the memory region to the hardware.
    unsafe {
        trigger_dma_transfer(ptr, 1500);
    }
    
    // 4. Freeze and Assume Init
    // Once the DMA interrupt fires (signaling completion), we know the 
    // memory is valid. We can now transition to safe Rust.
    // .assume_init_ref() is preferred for large structs to avoid stack copies.
    let packet: &[u8; 1500] = unsafe {
        packet_buffer.assume_init_ref()
    };
    
    // We can now safely read the data.
    assert_eq!(packet[0], 0xAA);
    assert_eq!(packet[1499], 0xAA);
}
```
