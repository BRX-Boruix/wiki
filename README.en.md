# wiki

BORUIX's **documentation site**: tutorials and manuals for users, and per-repository implementation notes for maintainers.

[简体中文](README.md)

## Contents

| Path | Contents | Status |
| --- | --- | --- |
| [`tutorial/`](tutorial/) | Getting-started tutorials | Available |
| [`contributor/`](contributor/) | Implementation notes per repository | Available |
| [`Boruix.md`](Boruix.md) | Project history | Unfinished |
| `install/` | Installation guide | Planned |
| `usage/` | User manual: commands, the shell, configuration | Planned |
| `faq/` | Frequently asked questions | Planned |

## Tutorials

### [Writing and installing drivers](tutorial/write-and-install-drivers.md)

How to add support for a new piece of hardware.

It starts from a counter-intuitive fact: in BORUIX, **a driver is not a kernel module but an ordinary user-space program**. Installing one means placing a file in the system's driver directory and declaring which device it binds to, with **no rebuild of the system itself**.

| Section | Contents |
| --- | --- |
| 0 | What a driver actually is in BORUIX |
| 1 | Quickly installing an existing driver (no compiling) |
| 2 | Writing a driver yourself |
| 3 | Building your driver into the system image |
| 4 | The complete flow at a glance (write → build → install → verify) |
| 5 | Common questions |

> If you already have a compiled driver, section 1 alone is enough; to write one from scratch, start at section 2.

## Implementation notes per repository

[`contributor/`](contributor/) collects the **implementation-level notes** for each repository — design tradeoffs, constraints, and traps that were hit. They matter to anyone reading or modifying that code, but do not belong in the repository's own README.

| File | Repository |
| --- | --- |
| [`contributor/libsys.md`](contributor/libsys.md) | The user-space system call library |
| [`contributor/libline.md`](contributor/libline.md) | The line editing library |
| [`contributor/login.md`](contributor/login.md) | The login program |

## Project history

[`Boruix.md`](Boruix.md) records how the project came to be — the generations of versions, why each ended, and what it left behind.

> That document is **unfinished**; its last section has a heading only.

## About this repository

The documentation here comes in two kinds: tutorials and manuals for **users**, and per-repository implementation notes for **maintainers** ([`contributor/`](contributor/)).

## License

MIT License, copyright Yang Borui. See [LICENSE](LICENSE).
