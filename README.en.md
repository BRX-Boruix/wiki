# wiki

BORUIX's documentation site, collecting usage notes for the public and per-repository implementation
notes for maintainers.

[简体中文](README.md)

## Contents

- [`tutorial/`](tutorial/) — tutorials
- [`contributor/`](contributor/) — implementation notes per repository
- [`Boruix.md`](Boruix.md) — project history

## Tutorials

### [Writing and installing drivers](tutorial/write-and-install-drivers.md)

How to add support for a new piece of hardware.

A driver is a user-space program, not a kernel module. Installing one means placing a file in the
system's driver directory and declaring which device it binds to, with no rebuild of the system.

The tutorial has two parts:

- Section 1: installing a compiled driver, without touching a compiler
- Sections 2 to 5: writing a driver yourself, from its structure to packaging it into the system image

## Implementation notes per repository

[`contributor/`](contributor/) collects implementation-level content per repository: design tradeoffs,
constraints, and the conventions to know when modifying that code. The README standard is
[`contributor/readme.md`](contributor/readme.md).

- [`contributor/readme.md`](contributor/readme.md) — the README standard
- [`contributor/libsys.md`](contributor/libsys.md) — the user-space system call library
- [`contributor/libline.md`](contributor/libline.md) — the line editing library
- [`contributor/login.md`](contributor/login.md) — the login program

## Project history

[`Boruix.md`](Boruix.md) records how the project came to be, including the generations of versions and
their outcomes.

## License

The contents of this wiki are under the MIT License; see [LICENSE](LICENSE).
