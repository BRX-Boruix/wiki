# wiki

BORUIX's documentation site, containing tutorials and manuals for users and per-repository implementation notes for maintainers.

[简体中文](README.md)

## Contents

- [`tutorial/`](tutorial/) — getting-started tutorials
- [`contributor/`](contributor/) — implementation notes per repository
- [`Boruix.md`](Boruix.md) — project history

## Tutorials

### [Writing and installing drivers](tutorial/write-and-install-drivers.md)

How to add support for a new piece of hardware.

A driver is not a kernel module but an ordinary user-space program. Installing one means placing a file in the system's driver directory and declaring which device it binds to, with no rebuild of the system itself.

The tutorial has two parts:

- Section 1: installing a compiled driver, without touching a compiler
- Sections 2 to 5: writing a driver yourself, from its structure to packaging it into the system image

## Implementation notes per repository

[`contributor/`](contributor/) collects the implementation-level notes for each repository: design tradeoffs, constraints, and the conventions to know when modifying that code.

- [`contributor/readme.md`](contributor/readme.md) — the README standard (Chinese only)
- [`contributor/libsys.md`](contributor/libsys.md) — the user-space system call library
- [`contributor/libline.md`](contributor/libline.md) — the line editing library
- [`contributor/login.md`](contributor/login.md) — the login program

## Project history

[`Boruix.md`](Boruix.md) records how the project came to be, including the generations of versions and their outcomes.

## License

MIT License, copyright Yang Borui. See [LICENSE](LICENSE).
