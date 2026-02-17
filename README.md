# My Markdown Book Repo

This repository contains the source for "My Markdown Book".

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
