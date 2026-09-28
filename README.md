# wiki

**简体中文** | [English](#english)

BORUIX 的**用户文档**——写给使用这个系统的人，而不是开发它的人。

这里讲的是：怎么装、怎么用、遇到问题怎么办。

---

## 这里放什么

| 目录 | 内容 |
| --- | --- |
| `tutorial/` | 入门教程——从一个具体目标出发，一步步做完 |
| `install/` | 安装指南 |
| `usage/` | 使用手册（命令、shell、配置） |
| `faq/` | 常见问题 |

> 目前 `tutorial/` 已有内容，其余目录为规划。

## 为什么不和开发者文档放在一起

这个项目有两类文档，面向的读者完全不同：

| 文档 | 读者 | 关心的问题 |
| --- | --- | --- |
| **本仓库** | 使用系统的人 | 怎么把它跑起来、怎么用 |
| 开发者文档 | 参与开发的人 | 为什么这样设计、内部怎么实现 |

混在一起的结果是两边都不好用：普通用户被设计决策的讨论淹没，开发者又要在面向新手的教程里
翻找架构说明。分开之后，各自可以有合适的深度和语气。

**所以这里的文档有意不假设读者懂内核。** 讲一个功能时，从"你要完成什么"出发，而不是从"这个
模块怎么实现"出发。

## 教程

### 编写与安装驱动程序

[`tutorial/write-and-install-drivers.md`](tutorial/write-and-install-drivers.md)

讲怎么给系统加一个新硬件的支持。这份教程有个很好的切入点：**它先破除一个常见的预设**。

在大多数系统里，"装驱动"意味着改内核、重编译。这个项目的做法完全不同——驱动**不是内核模块**，
而是一个**普通的用户态程序**。所以装驱动只是把文件放进系统的驱动目录并声明它绑定哪个设备，
**不需要重新编译系统本身**。

教程分两条路，读者可以按需选择：

| 情况 | 做法 |
| --- | --- |
| 已经有编译好的驱动 | 直接装进系统，不碰编译器 |
| 要自己写一个 | 从驱动程序的结构讲起 |

## 项目历史

[`Boruix.md`](Boruix.md) 记录这个项目的来龙去脉——几代版本的更替、每一代为什么结束、留下了
什么。

> 该文稿**尚未完成**，最后一部分只有标题。

## 许可

MIT License，版权归 Yang Borui 所有。详见 [LICENSE](LICENSE)。

---

# English

[简体中文](#wiki) | **English**

BORUIX's **user documentation** — written for people who use this system, not for people who build it.

It covers how to install it, how to use it, and what to do when something goes wrong.

---

## What goes here

| Directory | Contents |
| --- | --- |
| `tutorial/` | Getting-started tutorials — start from a concrete goal and work through it |
| `install/` | Installation guide |
| `usage/` | User manual (commands, the shell, configuration) |
| `faq/` | Frequently asked questions |

> `tutorial/` has content today; the rest are planned.

## Why it is separate from the developer documentation

The project has two kinds of documentation aimed at entirely different readers:

| Documentation | Reader | Question they care about |
| --- | --- | --- |
| **This repository** | People using the system | How do I get it running, how do I use it |
| Developer documentation | People building it | Why is it designed this way, how does it work inside |

Mixing them serves neither well: ordinary users drown in design debates, and developers end up
hunting for architecture notes inside beginner tutorials. Kept apart, each can strike the right depth
and tone.

**So the documentation here deliberately assumes no knowledge of the kernel.** A feature is explained
starting from "what are you trying to accomplish", not from "how is this module implemented".

## Tutorials

### Writing and installing drivers

[`tutorial/write-and-install-drivers.md`](tutorial/write-and-install-drivers.md)

How to add support for a new piece of hardware. The tutorial has a good starting move: **it first
disposes of a common assumption**.

On most systems "installing a driver" means modifying the kernel and rebuilding. This project works
quite differently — a driver is **not a kernel module** but **an ordinary user-space program**. So
installing one means placing a file in the system's driver directory and declaring which device it
binds to, with **no rebuild of the system itself**.

The tutorial offers two routes, so readers can pick:

| Situation | Approach |
| --- | --- |
| You already have a compiled driver | Install it, never touching a compiler |
| You want to write one | Start from the structure of a driver program |

## Project history

[`Boruix.md`](Boruix.md) records how the project came to be — the generations of versions, why each
one ended, and what it left behind.

> That document is **unfinished**; its last section has a heading only.

## License

MIT License, copyright Yang Borui. See [LICENSE](LICENSE).
