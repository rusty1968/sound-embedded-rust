# Sound Embedded Rust

> **Building a Kernel for the Abstract Machine**

This repository contains the source for the book **"Sound Embedded Rust"**.

It is a rigorous guide to building sound, panic-free embedded operating system kernels in Rust. The book focuses on reasoning about the Rust Abstract Machine, enforcing strict boundaries between safe and unsafe code, and establishing **trust anchors** with hardware by correctly modeling memory-mapped interfaces.

## Building the Book

This project uses [mdBook](https://github.com/rust-lang/mdBook).

### Prerequisites

1. Install Rust and Cargo: [rustup.rs](https://rustup.rs/)
2. Install mdBook:
   ```bash
   cargo install mdbook
   ```

### Building

To build the book:

```bash
mdbook build
```

The output will be in the `book/` directory.

### Serving Locally

To preview the book as you write:

```bash
mdbook serve --open
```

This will start a local web server and open your browser. It automatically reloads when you save a file.

## Structure

- `src/SUMMARY.md`: The table of contents.
- `src/*.md`: Chapter content.
- `book.toml`: Configuration.

## License

This project is licensed under the Apache License, Version 2.0. See [LICENSE](LICENSE) for details.
