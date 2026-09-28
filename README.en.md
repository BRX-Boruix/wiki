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
constraints, and the conventions to know when modifying that code. One file per repository; the README
standard is [`contributor/readme.md`](contributor/readme.md).

### Daemons and system services

- [`audiod`](contributor/audiod.md) — audio mixing
- [`consoled`](contributor/consoled.md) — keyboard events to console bytes
- [`driverd`](contributor/driverd.md) — automatic driver loading
- [`init`](contributor/init.md) — the init process
- [`userd`](contributor/userd.md) — account and home-directory sync
- [`volumed`](contributor/volumed.md) — volume mounting and removal

### Drivers

- [`intel-hda`](contributor/intel-hda.md) — the audio hardware driver
- [`userdrv`](contributor/userdrv.md) — the user-space driver template

### The command interpreter

- [`shell`](contributor/shell.md) — the user shell

### Libraries

- [`libc`](contributor/libc.md) — the C standard library
- [`libline`](contributor/libline.md) — the line editing library
- [`libsys`](contributor/libsys.md) — the user-space system call library
- [`csrc`](contributor/csrc.md) — the freestanding C runtime

### Tools and planned repositories

- [`tools`](contributor/tools.md) — system build and acceptance
- [`openvt`](contributor/openvt.md) — new terminals at run time
- [`coreutils`](contributor/coreutils.md) — base commands (planned)
- [`sdk`](contributor/sdk.md) — third-party development toolchain (planned)

### Applications and acceptance test programs

- [`acee2e`](contributor/acee2e.md) — the syscall chain of access-control rules
- [`audiofile`](contributor/audiofile.md) — the WAV player
- [`audioe2e`](contributor/audioe2e.md) — the audio blocking wake round trip
- [`blkdemo`](contributor/blkdemo.md) — control diagnostics for the legacy byte path
- [`consoled-e2e`](contributor/consoled-e2e.md) — console ring blocking wake
- [`evdemo`](contributor/evdemo.md) — the diagnostic form of the event stream
- [`evsrcdemo`](contributor/evsrcdemo.md) — the event source component
- [`focusdemo`](contributor/focusdemo.md) — adversarial acceptance of the focus gate
- [`fpcheck`](contributor/fpcheck.md) — floating-point path observation
- [`pwde2e`](contributor/pwde2e.md) — the account look-up chain
- [`selftest`](contributor/selftest.md) — the on-demand self-test host
- [`spinburn`](contributor/spinburn.md) — the kill-stress target process
- [`synce2e`](contributor/synce2e.md) — the sync word blocking round trip
- [`threaddemo`](contributor/threaddemo.md) — threads and thread-local storage
- [`tokendemo`](contributor/tokendemo.md) — console token truth
- [`trave2e`](contributor/trave2e.md) — directory traversal permission
- [`yielder`](contributor/yielder.md) — busy-yield stress
- [`login`](contributor/login.md) — login authentication
- [`cowsay`](contributor/cowsay.md) — the third-party program sample

## Project history

[`Boruix.md`](Boruix.md) records how the project came to be, including the generations of versions and
their outcomes.

## License

The contents of this wiki are under the MIT License; see [LICENSE](LICENSE).
