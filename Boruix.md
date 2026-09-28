# Boruix 项目

> 本条目仍不完善。
> 本条目除介绍历史之外，其余内容将统一使用第二次重启的版本讲述。

Boruix 是一个操作系统项目，其首要选择语言是 Rust。其被有效记录的 Git 仓库第一个版本于 2025 年 10 月 16 日由杨博睿（HelloAIXIAOJI）提交。协议主要为 MIT。

Boruix 名字来自于杨博睿的姓名，标志灵感来源则为杨博睿在 2025 年 10 月 14 日生日时的图片。

## 历史

### 项目开始与取消开发：BoruixOS

2025 年 10 月 14 日，杨博睿开始着手开发 BoruixOS。系统主要在 VMware 的 Linux 虚拟机下进行开发，编程语言主要使用 C，并引入了 Rust、Zig 和 .ASM 编程语言进行开发。在开发期间，项目确立了 LazyBuddy 内存管理器。

但因开发过程中屡次崩溃、多语言编写困难，在 2025 年 11 月 1 日终止开发。

（https://github.com/BRX-Boruix/os-Gen1-old）

### 项目第一次重启：boruixkernel

2026 年 2 月 19 日，项目迎来了第一次重启（具体时间已不可考），但因代码库无法进行维护在 3 月 8 日再度搁置。第一次重启的 Boruix 主要使用 Rust 进行编写、在 VMware 的 Linux 虚拟机下进行开发，其称呼由 OS 转为 Kernel。

在开发期间，flanterm 被翻译为了 Rust 版本（也是之后 Boruix 所使用的 flanterm 版本。）、确立了统一的 driver hub、确认全 Rust 开发可行。boruixkernel 的重启为后续的设计奠定了基础。

其 Git 仓库因一些原因丢失（因从虚拟机复制文件到实体机的过程中没有复制 git 目录，且虚拟机也已被删除），项目开发时并未同步至 GitHub，但代码库得以保留。杨博睿于 2026 年 9 月 20 日正式公开了其代码库（https://github.com/BRX-Boruix/boruixkernel-Gen2-old）

### 项目第二次重启：BRX-Boruix/Kernel
