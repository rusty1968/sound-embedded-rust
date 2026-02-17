# The Embedded Rust Kernel

> **Use the Unsafe to build the Safe.**

This book is a comprehensive guide to designing and implementing a small, real-time operating system (RTOS) kernel for microcontrollers using the Rust programming language.

It is structured as a collection of self-contained deep-dives ("books within a book") that can be read largely independently, though they build towards a complete kernel architecture.

## Roadmap

1.  **Verified Dangerous**: We start by mastering the dark arts of `unsafe` Rust. You cannot build a kernel without stepping outside the compiler's safety guarantees. This section strictly teaches the rigorous, low-level skills required to interface with hardware and manage memory manually—skills often glossed over in standard Rust tutorials.
2.  **The Core**: Once we have the tools, we design the kernel. We will implement high-performance abstractions for scheduling, synchronization, and driver management, designed specifically for the constraints of embedded systems.

## Prerequisites

*   Comfortable with basic Rust syntax (The Book).
*   Basic understanding of embedded systems (registers, interrupts).
*   A microcontroller development board (e.g., STM32, nRF52) or QEMU setup.
