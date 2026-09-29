# liftoff

BORUIX 的引导程序。README 面向使用者；本文件记录引导协议的分阶段规划与改动前需要知道的约定。

## 引导协议分两步走

Liftoff 与内核之间的交接契约分两个阶段实施，不要跳过第一阶段。

### 阶段一：复用 Limine 协议

内核现在用 `brxlimine-rs` 解析 Limine 协议结构。耦合面经核对为 26 行、7 个文件：

- `kernel/crates/kernel/src/main.rs` —— `BaseRevision`、`Framebuffer`、`KernelFile`
- `kernel/crates/kernel/src/acpi.rs` —— `Rsdp`
- `kernel/crates/kernel/src/smp.rs` —— `Smp`
- `kernel/crates/kernel/src/drivers.rs` —— 消费帧缓冲
- `kernel/crates/kernel/src/lib.rs`、`mod.rs`、`init.rs` —— 引用转发

阶段一 Liftoff 只实现这 5 个请求，内核零改动。

之所以先走这条路：本阶段同时引入 UEFI 引导、EXT2、ISO9660、ELF 加载四件新事物。若再叠加
自拟协议，任何异常都无法定位是哪一层的锅。复用 Limine 协议还保留了一个对照物——用
`brxLimine` 跑一遍作基线，可快速区分「引导程序错了」还是「内核错了」。

### 阶段二：自拟 BORUIX 协议

阶段一稳定运行后再启动。动机有三条：

1. Limine 协议有 40 余个请求，本项目只用 5 个
2. Limine 是 C ABI，强制 `#[repr(C)]` 加手写的 `Ptr<T>`／`NonNullPtr<T>` 包装，
   `brxlimine-rs` 中大半代码在此
3. `BaseRevision` 的版本协商为跨版本兼容而设；本项目引导程序与内核同仓发布，版本恒匹配

设计约束，不可协商：

- 不做多版本协商。只保留 `magic` 与 `version` 两个校验字段做精确匹配，版本不符即拒绝启动
- 不手写指针包装。协议结构用 Rust 原生类型（`Option<&T>`、`&[T]`）定义，仅在交接那一刻
  转为裸地址
- 必须写「谁负责填」清单。协议文档的核心价值是明确每个字段由引导程序填、由内核消费，还是
  双方约定不可触碰。Limine `PROTOCOL.md` 中最有价值的部分正是这份职责划分
- 保留 `brxLimine` 作为回归基线，直到阶段二稳定

设计时需明确的待定项：

- 内存映射的编码方式（类型枚举、保留位语义）
- 高半区直接映射（HHDM）地址约定 —— 内核 `mm` 强依赖
- 页表交接方式 —— 内核是否重建页表
- SMP 启动 AP 的握手协议
- 帧缓冲像素格式枚举

## 为什么不用 brxLimine

`brxLimine` 是 Limine 12.5.2 的 fork，约 10 万行源码。BORUIX 只用到其中的 5 个协议请求与
EXT2、ISO9660 加载。翻译或沿用它等于把 95% 用不到的功能一起背负，且永远追不上上游。

## ISO9660 的必需性

ISO9660 不是可选功能。`tools/tools_build/run.py` 的默认启动方式是从光盘启动
（`-cdrom` 加 `boot_order=d`），磁盘启动反而需要显式传 `--systemdisk`。内核的
`boot_source()` 据此区分两种来源：

- `media_type` 为 `1`（optical）判定为 liveCD，走 ISO9660
- `media_type` 为 `0`（generic）判定为磁盘模式，走 EXT2

因此两个文件系统都必须实现。改动 `src/config.rs` 中的 `MEDIA_TYPE_*` 常量时必须同步核对
内核侧的同名常量，这两处是一份契约。

## FAT32 不在自研范围

UEFI 固件只从 FAT 分区加载 `.efi`，这是规范强制的。但固件同时提供
`EFI_SIMPLE_FILE_SYSTEM_PROTOCOL`，读自身所在分区无需自写 FAT32 驱动。
自研文件系统只有 EXT2 与 ISO9660 两个。

## 命名

仓库名 `liftoff`，取火箭离地那一刻。不使用 `limine` 相关命名——本项目实现的是引导协议，
不是 Limine 的分支。不使用 `loader`，内核 `crates/` 下已有同名 crate。
## 观测走串口，不走固件控制台

输出有两条通道：固件控制台（`SimpleTextOutput`）与 COM1 串口（`src/serial.rs`）。串口是
观测主通道，固件控制台只是附带。理由：panic 处理器不能依赖固件服务——panic 可能在
`ExitBootServices` 之后或固件状态可疑时发生，此时唯一还活着的就是 16550 端口。串口初始化
（`serial::init`）可重复调用，panic 路径依赖这一点。

QEMU 下串口经 `-serial stdio` 直达宿主标准输出，验收脚本读宿主进程输出即可判定启动结果，
无需截图或人工看窗口。

## M1 的验证方式

QEMU 加 `-drive if=pflash,file=edk2-x86_64-code.fd` 走 OVMF 固件，引导介质用
`-drive file=fat:rw:<esp目录>`（QEMU 虚拟 FAT），目录内放
`EFI/BOOT/BOOTX64.EFI`。判定依据是串口出现 `signature ok` 与
`console + serial alive` 两行。目录布局必须含 `EFI/BOOT/` 两级，直接放根目录固件
不会自动引导，会落进 UEFI Shell。

## M2a 的验证方式

M2a 证明固件文件协议链可用：`LoadedImage`（自身镜像）→ `DeviceHandle` →
`SimpleFileSystem` → `OpenVolume` → `Open` → `Read` 循环至 `EFI_END_OF_FILE`。
被读对象是引导介质根目录的 `m2a.txt`，串口输出长度与 16 位字节和，宿主端预计算比对。

约定与陷阱，后续里程碑直接受用：

- **句柄纪律**：卷句柄与文件句柄都经 `efi::FileGuard` 持有，`Drop` 即 `Close`，
  错误路径与正常路径共用一个释放点。引入写句柄时必须重审"关闭状态可忽略"的前提。
- **`EFI_END_OF_FILE` 是正常终止**：它带错误位但语义是 EOF，`efi::is_error` 为真
  不代表失败。Read 的单次调用可能部分填充，必须循环。
- **BootServices 只声明到 `ExitBootServices`（偏移 232..240，共 240 字节）**：其后
  的 `InstallMultipleProtocolInterfaces` 两个成员是 C 变参函数，Rust 的 efiapi ABI
  无法声明；触及对应能力时改走裸地址加手工调用约定。部分声明经 `allow(dead_code)`
  放行，理由写在 efi.rs 模块头——整段声明是布局正确性的前提，不是懒惰。
- **M1 布局缺陷已修**：`SimpleTextOutput` 曾漏 `EnableCursor` 字段，`mode` 偏移错
  8 字节。因 M1 未解引用 `mode` 而未暴露；现在全部结构的大小有编译期断言钉死。
- **串口验收契约行以 `M2A:` 为前缀**，与 `[liftoff]` 前缀的普通日志区分；
  验收脚本只断言契约行。

验收数据（`tools/boottest.ps1` 三个变体，QEMU 加 OVMF 真实引导，非模拟）：

- normal：`m2a.txt` 存在（36 字节），断言 `M2A: len=36` 与 `M2A: sum=0x0834`
- missing：文件不存在，断言 `M2A: open failed status=0x800000000000000e`（`EFI_NOT_FOUND`），
  且不出现 `M2A: len=`——文件缺失是合法分支，引导不失败
- empty：零字节文件，断言 `M2A: len=0` 与 `M2A: sum=0x0000`

脚本位于 `tools/boottest.ps1`，先构建、再摆 ESP、再引导、再断言串口日志；QEMU 路径取自
项目根 `.env` 的 `QEMU_DIR`。M2b 起 ISO9660 与 EXT2 的读取将复用同一脚本加新变体。
## M2b 的验证方式

M2b 实现了 ISO9660 只读驱动（`src/iso9660.rs`，ECMA-119 基本卷）。链路：
`LocateHandleBuffer(BlockIo)` → 逐句柄 `HandleProtocol` → 光驱判定（只读且块大小
2048，与 brxLimine 同款判定式）→ `CD001` 探测挂载 → 读 `KERNEL/KERNIMG.BIN`，
串口报长度与 16 位字节和。

架构约定：

- **解析器与固件解耦**：块读取经 `iso9660::BlockRead` trait 注入，`UefiBlock` 适配器
  负责 BlockIo 的块对齐拼接。M2c 的 EXT2 复用同一 trait，不重写这套边界。
- **暂存缓冲单点定义**：`PoolBuf` 走固件 `AllocatePool`/`FreePool`（类型
  EFI_BOOT_SERVICES_DATA），Drop 即释放。不许再引入第二套分配器。
- **名字归一化**：匹配时去 `;版本号` 后缀、去版本分隔符前的尾点、ASCII 大小写不敏感。
  未来 xorriso 生成的真实 liveCD 用大写 8.3 名，同一匹配路径覆盖。
- **不支持 multi-extent**：带 0x80 标志的记录直接报错（内核 ELF 单 extent 上限 4GB，
  超出拒绝比读出错误内容诚实）。
- **目录条目不跨扇区**：`length==0` 的填充区跳到下一扇区边界继续（ECMA-119 6.8.1.1）。

验收（`tools/boottest.ps1`，fixture 由 `tools/mkiso.py` 确定性生成，独立解析脚本
`_verify_iso.py` 先验证 fixture 合法）：

- iso：合法 ISO 含 `/KERNEL/KERNIMG.BIN`（36 字节），断言 `M2B: mount ok`、
  `M2B: len=36`、`M2B: sum=0x089c`
- iso-nosig：非 ISO 随机块，断言 `M2B: mount failed status=0x11`（探测失败码），
  且不出现 mount ok
- iso-nopath：合法 ISO 但根目录无 KERNEL 目录，断言 mount ok 后
  `M2B: open failed status=0x800000000000000e`

M2a 的三个变体（SFS 链基线）与 M2b 共同回归，六个变体必须同时全绿。
## M2c 的验证方式

M2c 实现了 EXT2 只读驱动（`src/ext2.rs`）。链路：`BlockIo` 枚举 → 磁盘判定
（可写、512B 块、非分区）→ 超级块魔数（0xEF53 @1024）→ 组描述符 → inode 表 →
目录项下钻 `/BOOT/KERNIMG.BIN`，串口报长度与 16 位字节和。

布局事实源是内核 `kernel/crates/fs/src/ext2.rs` 的取证版（SB 字段偏移、GDT @
first_data_block+1、inode 表定位公式 `(ino-1)/ipg`、目录项 8 字节头）。两侧是
同一份磁盘格式的两个独立解析器，fixture 必须能同时被两者读懂。

有意收窄的边界（显式拒绝，不是遗漏）：

- 块大小只支持 1024（`log_block_size==0`），其余显式报错
- 数据寻址：直接块 12 + 一级间接；二级/三重间接报错（内核 ELF 不超过 12KiB 时
  不触达，M3 按需扩展）
- 目录数据上限一个块（1024B）：fixture 与安装镜像口径，超出报 CORRUPT_DIRENT
- 无 MBR：M2c 整盘挂载；分区解析与 media_type 语义归 M4 BootSource 逻辑

验收（`tools/boottest.ps1`，fixture 由 `tools/mkext2.py` 确定性生成，
`_verify_ext2.py` 独立验证）：

- ext：合法 EXT2 含 `/BOOT/KERNIMG.BIN`（38 字节），断言 `M2C: mount ok`、
  `M2C: len=38`、`M2C: sum=0x08b7`
- ext-nosig：非 EXT2 随机块，断言 `M2C: mount failed status=0x21`（BAD_MAGIC），
  且不出现 mount ok
- ext-nopath：合法 EXT2 但根目录无 BOOT 目录，断言 mount ok 后
  `M2C: open failed status=0x800000000000000e`

fixture 陷阱（验收抓出来的）：EXT2 目录块的尾部零填充不是合法目录项——
最后一项的 rec_len 必须覆盖到块尾，否则解析器把零字节当 rec_len=0 的损坏项拒绝。
这个语义与 ISO9660 的零长条目跳扇区规则不同，两个驱动不能共用目录遍历逻辑。

M2a/M2b/M2c 九个变体共同回归，必须同时全绿。
## M3 的验证方式

M3 实现了 ELF64 静态装载器（`src/elf.rs`）。链路：M2b 光盘链读到
`KERNEL/KERNIMG.BIN` 后交给 `elf::load`——真实内核 ELF（4.3MB PIE）走完整
装载路径，串口报段布局、入口、映像大小与校验和。

装载模型（brxLimine `common/lib/elf.c` 916..1039 同构，去 KASLR 与重定位）：

- 布局：PT_LOAD（memsz>0）参与，min_vaddr = 最小 p_vaddr，image_size =
  max(vaddr+memsz) - min_vaddr（vaddr 跨度，含段间空洞，因此可以大于 Σmemsz）
- 物理映像：单次 `AllocateAnyPages`（UEFI §7.2 保证单次调用返回连续页块），
  类型 EfiLoaderData（M4 交接语义）；整区先清零——空洞与 BSS 不继承陈旧内存
- 段落位：`image_base + (p_vaddr - min_vaddr)`，复制 filesz 字节，余下清零
- 校验（与 `tools/elf_oracle.py` 同源）：ELF64/LE/EM_X86_64、ET_EXEC|ET_DYN、
  phentsize==56、filesz<=memsz、offset+filesz 不越文件、4KB 页重叠拒绝
  （M3 无页表，页内权限无法区分，同页段一律拒绝——brxLimine 只拒绝不同权限，
  这里更严）

边界：ET_DYN 接受但**不做重定位**（内核 R_X86_64_RELATIVE 的重定位属 M4 协议层
或内核自举页表后自理）；段数上限 8（内核 3 段留余量）；不触碰页表——装载结果
目前只用于验收链路闭环，不实际跳转。

验收（`tools/boottest.ps1`）：

- elf-iso：真实内核 ELF 经 mkiso 打包，期望值由 `elf_oracle.py` 运行期计算
  （内核每次重建都变，fixture 与断言同源动态化），断言段布局 6 行契约
- elf-bad：4096 字节伪随机 blob 作 KERNIMG.BIN，断言 `M3: reject status=0x30`
  （BAD_MAGIC），且不出现任何段报告行

工程注记：`boottest.ps1` 的 `$ErrorActionPreference=Stop` 下，cargo 的正常
stderr（Compiling/Finished）经 `2>&1` 管道时会被当作终止错误——本机直接运行
不触发，换一种调用方式即触发（S05：脚本不得依赖调用方式）。已改经 cmd /c 执行
cargo；fixture 假设变量作用域时注意 `-like "iso*"` 不匹配 `elf-iso`（绿灯抓出）。

M2a/M2b/M2c/M3 十一个变体共同回归，必须同时全绿。
