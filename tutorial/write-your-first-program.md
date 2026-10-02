# 写第一个程序（面向普通用户）

> 适用版本：当前 BORUIX（`tools` 构建系统 + `b3p` 第三方程序通道已落地）。
> 本文面向**普通用户**：不要求理解内核，只讲「怎么写一个能在这系统上跑的程序、
> 怎么把它放进去、怎么运行」。
>
> **先说清现状**：今天写 BORUIX 程序，仍然需要先把系统的各仓库拿到本地（见 §1）。
> 这是已知的不便；消除它正是工具集 `sdk` 的立项理由（见
> [`contributor/sdk.md`](../contributor/sdk.md)），但尚未落地。本文只讲**今天
> 就能跑通**的路径。

## 0. 一个 BORUIX 程序是什么

- 它是一个**独立的 ELF 可执行文件**：不是内核模块，也不编进系统镜像；
- 它住在**数据盘**上，路径形如 `/volumes/BORUIX_DATA/3p/你的程序.elf`，由 shell 直接执行；
- 它只依赖 `libsys`（用户态系统调用库），没有任何内核特权；
- 它是 `no_std` 程序：没有标准库，入口是 `user_main` 而不是 `main`。

> **一句话**：写程序 = 写一个只依赖 libsys 的 no_std ELF；装程序 = 把它放到数据盘的 `3p/` 下。

---

## 1. 准备

1. **拿到系统的各仓库**（这一步将来会被 `sdk` 消除）。BORUIX 按仓库划分（ADR-002），
   各仓库见 `github.com/BRX-Boruix`；把它们**并列**放在同一个工作目录下，形如：

       work/
       ├── kernel/
       ├── libsys/
       ├── libc/
       ├── shell/
       └── tools/

   本文后续命令都在这个工作目录下执行。
2. **Rust nightly 工具链**：编译目标是内建的 `x86_64-unknown-none`。
3. **Python 3**：构建脚本在 `tools/`。
4. **QEMU**（`qemu-system-x86_64`）：用来运行系统。

## 2. 建程序目录

在**工作目录根**下建一个目录，名字就是程序名（它同时决定产物 ELF 的名字）：

    mkdir -p myprog/src

目录里需要四个文件，缺一不可：

| 文件 | 作用 |
|---|---|
| `Cargo.toml` | 声明 libsys 依赖；`panic = "abort"`（裸机无 unwinding 运行时） |
| `build.rs` | 把链接脚本与 `-no-pie` 传给链接器 |
| `linker.ld` | 段布局：从 `0x400000` 起，入口 `_start` |
| `src/main.rs` | 你的代码 |

### 2.1 Cargo.toml

```toml
[package]
name = "myprog"
version = "0.1.0"
edition = "2024"
build = "build.rs"

[dependencies]
libsys = { path = "../libsys" }

# 裸机用户态程序：无 unwinding 运行时，panic 直接 abort。
[profile.dev]
panic = "abort"
[profile.release]
panic = "abort"
opt-level = "z"
lto = true
```

### 2.2 build.rs

```rust
fn main() {
    let target_os = std::env::var("CARGO_CFG_TARGET_OS").unwrap_or_default();
    if target_os == "none" {
        let dir = std::env::var("CARGO_MANIFEST_DIR").expect("CARGO_MANIFEST_DIR");
        println!("cargo:rustc-link-arg=-T{}/linker.ld", dir);
        println!("cargo:rustc-link-arg=-no-pie");
    }
}
```

**为什么必须有这两行**：

- `-T…/linker.ld`：不指定链接脚本，段会被链接到地址 0，与内核用户半区冲突，加载不了；
- `-no-pie`：`rust-lld` 默认产出 PIE（`ET_DYN`），而内核加载器只接受 `ET_EXEC`
  （静态、固定布局）。

### 2.3 linker.ld

链接脚本是**通用模板**，与你的程序内容无关。直接复制仓库里现成的一份：

    cp cowsay/linker.ld myprog/linker.ld

（它把 `.text`/`.rodata`/`.data`/`.bss`/`.got` 各自页对齐放置，产出多个
`PT_LOAD` 段以满足 W^X；并丢弃 `.eh_frame`、`.dynamic` 等裸机静态程序不需要的
动态残留。）

### 2.4 src/main.rs

最小程序：

```rust
#![no_std]
#![no_main]

use libsys::{STDOUT, write};

#[unsafe(no_mangle)]
pub extern "C" fn user_main(_argc: isize, _argv: *const *const u8) -> i32 {
    let _ = write(STDOUT, b"hello, boruix\n");
    0
}
```

想读命令行参数时**不要自己解析**——入口参数的语义与 POSIX 不同（见 §5.1），
用 libsys 提供的 API：

```rust
#![no_std]
#![no_main]

use libsys::{STDOUT, cmdline, split_words, write};

#[unsafe(no_mangle)]
pub extern "C" fn user_main(argc: isize, argv: *const *const u8) -> i32 {
    // 整条命令行在 argv[0] 里，拆词是程序的职责。
    let line = match unsafe { cmdline(argc, argv) } {
        Some(l) => l,
        None => return 0,
    };
    for w in split_words(line) {
        let _ = write(STDOUT, w);
        let _ = write(STDOUT, b"\n");
    }
    0
}
```

## 3. 登记到构建清单

编辑 `tools/tools_build/b3p.py`，把程序名加进 `THIRD_PARTY_PROGRAMS`：

```python
THIRD_PARTY_PROGRAMS = (
    "cowsay",
    "myprog",     # <- 加这一行
)
```

这个元组是**单点定义**：程序名同时决定源码目录与产物路径。

## 4. 构建、放盘、运行

**第 1 步：编译并落到数据盘素材目录**

    python tools/main.py b3p --prog myprog

产物在 `tools/diskfiles/3p/myprog.elf`。不带 `--prog` 则编清单里的全部。

**第 2 步：重建数据盘并启动系统**

    python tools/main.py br --redisk

`--redisk` 会用当前构建产物重建数据盘再启动；加 `--serial` 可看到内核与程序的输出。

**第 3 步：登录**

种子账户：`alice` / `alicepw`。

**第 4 步：运行**

    /volumes/BORUIX_DATA/3p/myprog.elf alpha beta

程序会在标准输出打印它收到的每个词。

## 5. 常见错误

### 5.1 「参数不对」——入口参数不是 POSIX argv（最容易踩）

- `argc` **恒为 1**，`argv[0]` 指向**整条命令行字符串**，**不是程序名**；
- 经 shell 执行时，shell 已把命令名剥掉，程序收到的是**纯参数串**；
- **内核不拆词**：按空白拆词是**程序的职责**；
- 用 `libsys::cmdline` / `split_words` / `words_into`，不要自己写解析。

权威定义见仓库内 `docs/abi/syscall-abi.md` §4。

### 5.2 程序加载失败 / 段地址不对

- 少了 `-no-pie`：产物是 `ET_DYN`（PIE），内核只接受 `ET_EXEC`；
- 少了 `-T linker.ld`：段会链接到 vaddr 0。

自检产物：用 `objdump -f` 或 `llvm-readelf -h` 查看，`Type` 应为 `EXEC`，
入口应为 `0x400000`。

### 5.3 程序跑了但文件找不到

数据盘没挂上：确认启动时带了 `--disk` 或 `--redisk`，并确认路径是
`/volumes/BORUIX_DATA/3p/…`（卷标 `BORUIX_DATA` + 盘内 `3p/` 目录）。

## 6. 现状与将来

今天的不便，如实列出（每一条都是 `sdk` 要消除的）：

1. **必须先拿到系统的各仓库**——工具集的目标是把这条路独立出来；
2. **程序目录要手抄 `build.rs` / `linker.ld`**——将来由目标定义取代；
3. **libsys 以 path 依赖引用**——将来改 git 依赖，或随 sysroot 提供预编译库；
4. **C 语言还没有等价路径**——libc 尚未接进构建；
5. **编译目标用的是内建 `x86_64-unknown-none`**——将来是 `x86_64-unknown-boruix`。

规划与取舍见 [`contributor/sdk.md`](../contributor/sdk.md)。
