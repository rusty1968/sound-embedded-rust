# Introduction

Building an operating system kernel is one of the most challenging and rewarding tasks in computer science. Building one in **Rust** adds a unique layer of complexity and opportunity: we must explicitly define the boundary between the chaos of hardware and the order of safe software.

## Philosophy

This book follows a "bottom-up, then top-down" approach.

### Part 1: Verified Dangerous
We begin by acknowledging that hardware is inherently `unsafe`. Registers are just memory addresses that trigger side effects. Interrupts break linear control flow. The stack is just a pointer.

To tame this, we must become experts in:
*   **Pointers & Memory**: Understanding exactly what `*const u32` means and how it differs from `&u32`.
*   **Volatile Access**: Why the compiler wants to optimize away your hardware drivers, and how to stop it.
*   **Inline Assembly**: When Rust isn't enough, we drop down to the metal.
*   **Interior Mutability**: How to share data safely when the borrow checker says "no".

### Part 2: The Core
Armed with unsafe superpowers, we switch gears to **API Design**. A kernel's job is to provide safe abstractions over unsafe hardware.

We will design:
*   **Context Switching**: The magic trick of pausing one computation and resuming another.
*   **Scheduling**: Deciding who runs when.
*   **Synchronization**: Preventing race conditions on single-core and multi-core systems.
*   **Drivers**: Using Rust's type system (Traits, Generics) to write drivers that are impossible to misuse.

## Toolchain

We will be using:
*   **Rust Nightly**: For features like `naked_functions` and `asm!`.
*   **LLVM Tools**: `llvm-objdump`, `llvm-nm` for inspecting binaries.
*   **QEMU**: for rapid prototyping without hardware.

Let's begin.
