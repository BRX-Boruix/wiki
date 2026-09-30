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
## M4 的验证方式

M4 让 liftoff 从"验收壳"变成真 bootloader：把 M3 装载的内核送进 CPU，
内核以 Limine 语义子集接管机器。交接面 = BORUIX 内核已消费的 7 项协议：
BaseRevision、HHDM、Memmap、Framebuffer、Rsdp、KernelFile、KernelAddress
（SMP 等 16 项未消费请求全部裁剪）。内核零改动。

交接链（`src/boruix.rs` + `src/handover.rs` + `src/paging.rs`）：

- 响应扫描：内核镜像内 48B 窗口滑步匹配 `COMMON_MAGIC|id(2u64)|rev|response`
  请求标记（brxlimine-rs `Request` 布局），命中即在内核镜像内回填响应指针；
  BaseRevision 标记（24B，无 COMMON_MAGIC 前缀）单独扫，rev≤0 时写 0 表示支持
- 响应区：单块 EfiLoaderData；所有响应结构指针以 **HHDM 虚地址**填入
  （Limine 语义：内核经 HHDM 直接解引用；KernelAddressResponse.physical_base
  除外——保持物理）
- 内存映射：GetMemoryMap（64KiB 缓冲，重试上限 4）→ UEFI 类型转 Limine 7 类
  （Conventional/LoaderCode/LoaderData/BootServices* → Usable；ACPI reclaim/
  NVS 保留原语义；Runtime*/MMIO → Reserved）→ 内核区单列 KernelAndModules、
  帧缓冲区单列 Framebuffer → 按 base 排序 + 相邻同类型合并
- 帧缓冲：LocateProtocol(GOP) → mode.info → Framebuffer{address=HHDM 虚地址,
  1280x800x32, RGB/BGRX 掩码}；GOP 槽位序 QueryMode/SetMode/Blt/Mode
  （UEFI §11.9——mode 在 24B，首轮错排成 16B 读到垃圾指针，靠断言+运行期抓拍揪出）
- RSDP：M4 子集传 NULL（内核 acpi::init 有 Option 退化）；M5 补 configuration
  table 遍历
- 页表：4 级大页（2MiB）。恒等 [0,ram_top)+LAPIC 区（0xFF000000，EBS 回调
  期间固件 handler 仍写 LAPIC EOI——CR2=0xFEE00020 #PF 实锤）、HHDM
  [HHDM, +max(ram_top, fb_end))（帧缓冲 BAR 在 RAM 顶端之外）、内核高区
  [kvbase, +ksize) 逐 2MiB 窗口（物理基址必须 2MiB 对齐——AllocateAnyPages
  只保证 4KiB，多分配 512 页取对齐子区）
- 跳转：cli + cld → CR3 → 近跳 entry。长模式与 CS 沿用 UEFI；内核 kmain
  自切 __kstack_top 栈、自建 GDT。far ret（retfq）在 LLVM Intel 语法下编码
  不可靠（实测跳后静默），弃用
- EBS：GetMemoryMap 取 key → ExitBootServices(key)，失败（映射变更）重取
  重试 ≤4；EBS 后零 BootServices 调用

M4 修复链（每条都有 QMP 抓拍/串口实锤，记入评审素材）：

1. GOP mode 槽位 16→24（info 指针读到 0xcd894d5541c68945 垃圾）
2. 内核物理基址 2MiB 对齐（AllocateAnyPages 4KiB 对齐 → 大页 #PF）
3. 响应指针物理→HHDM 虚地址（Limine 语义）
4. **PIE 重定位**：内核是 DYN，.rela.dyn 3138 条 R_X86_64_RELATIVE，GOT/字面量
   槽在文件里为 0——不处理则 kmain 读 __kstack_top 得 0，rsp=0，第一条 push
   #PF（CR2=0xfffffffffffffff8，QMP info registers 抓拍 RSP=0 实锤）。
   elf.rs 第三遍：PT_DYNAMIC→DT_RELA/DT_RELASZ/DT_RELAENT→逐条 RELATIVE
   slot(image_base+r_offset-vbase)=addend（高半恒等装载 load_bias=0）
5. HHDM 映射上界覆盖帧缓冲 BAR（内核 [terminal] 写 fb #PF，内核异常处理器
   自报 CR2=0xffff800080000000=HHDM+ram_top）
6. asm! clobber 声明（rdx/rbx/rcx）——noreturn 下遗漏导致寄存器踩踏随机死

验收（`tools/boottest.ps1 -Variant boot`）：

- 断言串口出现 `Kernel M0 is running.`——只有固件真实装载 liftoff、liftoff
  真实装载内核、交接真实成功，内核 kmain 才能打出该行
- 内核侧完整链路：Hello, BORUIX! → Kernel M0 is running. → [kmain] serial &
  driver hub → [mm] HHDM offset 0xffff800000000000 → [pmm] 112 entries →
  driver_hub 四阶段 → pci 枚举
- M2a/M2b/M2c/M3/M4 全 12 变体回归全绿

工程注记：EBS 后 COM1 输出在 OVMF/QEMU 下不可靠（偶发丢字节），调试期用
QEMU `-debugcon`（io 0x402）做第二通道；QMP（`-qmp tcp:...` + human-monitor-
command `info registers`/`xp`）是页表/寄存器级实锤来源，`-no-shutdown` 保留
三重故障现场。所有 DIAG 代码验证后即删，正式路径 EBS 后零内存写、零服务调用。
## M5 的验证方式

M5 补齐 RSDP 传递并修复 M4 memmap 交接的两个结构错误。内核侧全链变为：
RSDP rev=2 → XSDT 遍历 → FADT/HPET/S5 → `acpi initialized`，比 M4 更深
（栈守护武装、mmio 4K map、hpet enable 全部落地）。

RSDP（`src/efi.rs` + `src/main.rs`）：

- 配置表遍历：SystemTable.configuration_table（EFI_CONFIGURATION_TABLE
  24B 项 = GUID16 + 指针8，size 断言钉死 24——首版误断言 16 被 E0080 当场
  拦截）按 ACPI_20_GUID {8868E871-E4F1-11D3-BC22-0080C73C8881}（UEFI §4.6.2）
  命中后取 RSDP 物理指针
- 立即拷贝：源页类型 AcpiReclaim 会被内核回收，两阶段读取（20B 基础 →
  rev≥2 读 offset20 的 length）后拷入 EfiLoaderData；RsdpResponse.address
  填 HHDM 虚地址（Limine 语义）

memmap 交接修复（`src/handover.rs`，两处都是 M4 遗留、被 M5 揭开）：

1. **指针数组语义**：brxlimine-rs MemmapResponse.entries 是
   `ArrayPtr<MemmapEntry>` = "entry_count 个指针的数组"（lib.rs 626-627），
   内核 `mmap: &[NonNullPtr<MemmapEntry>]` 逐槽解引用。M4 误填结构体连续
   数组 → 内核把 8B 槽当指针解引用出全垃圾（102 条 0x0-0x0 段、
   max_phys 4GB 假地址）。新布局：LoaderData 页首 = 指针数组 [ptr; n]，
   页尾 = 结构体数组，指针 = 槽位物理 + HHDM（8n+24n ≤ 4096 → n ≤ 128）
2. **usable 重叠**：convert_memmap 剥离切割未 clamp 到 desc 区间——
   k_lo 在 desc 之外时 `k_lo - base` 产出越界长度，91 条 usable 相互重叠，
   内核 pmm 反复处理同一 448MB 段。三段切割全部 clamp 后 usable 降到 13 条，
   max_phys 0x7ef4000（126.9MB = 真实 RAM 顶）

loader 自占区（BootloaderReclaimable）：M4 把 EfiLoaderCode/Data 转 usable
——响应区/文件拷贝/页表会被内核 pmm 回收踩踏。改标
BootloaderReclaimable(5)（内核 pmm 只回收 Usable，且语义与 Limine 一致：
内核快照页表根后可复用）。帧缓冲剥离同步改用新增 Handover.fb_phys（
fb_struct.address 是 HHDM 虚地址，误作物理曾产出 base=0xffff800080000000
的 Framebuffer 条目，pmm max_phys 飙 17TB）。

验收（`tools/boottest.ps1 -Variant boot`，内置 5 锚，不依赖调用参数）：

- `Hello, BORUIX!` / `Kernel M0 is running.`（跳转 + banner）
- `[mm] HHDM offset: 0xffff800000000000`（协议 HHDM）
- `[acpi] RSDP rev=2`（M5 配置表 → 拷贝 → HHDM 全链）
- `LazyBuddy init done`（memmap 指针数组语义 + 无重叠 usable 的 pmm 全程）
- 全 12 变体回归全绿
## M6 的验证方式

M6 让 liftoff 具备真正的 SMP 能力：MADT 枚举 → AP trampoline（实模式→长模式）→
INIT-SIPI → AP 停泊轮询 `goto_address` → 内核原子写入即跳 `ap_entry`。
验收锚：`[smp] AP online, lapic_id=1`（`-smp 2` 下真实双核）。

交接面（`src/smp.rs`）：

- MADT：RSDP→XSDT→签名 "APIC"→type 0 条目（flags bit0 enabled）枚举 LAPIC；
- SmpInfo/SmpResponse：cpu_count 含 BSP（索引 0 = BSP，内核按 lapic_id 跳过）；
  `cpus` 是指针数组（brxlimine-rs ArrayPtr 语义，与 memmap 同款教训）；
- AP 资源：trampoline 页固定 0x70000（SIPI 向量 = phys>>12，必须 <1MB；
  0x1F0000 被 OVMF 的 AP 缓冲覆写——内存 dump 实锤）、64KiB 引导栈、
  诊断/标记区；全部 EfiLoaderData → 重转 memmap 后为 BootloaderReclaimable，
  内核不回收（`[0x70000,0x72000)` 在 usable 之外，pmm 日志实证）。

trampoline 三段式（对齐 brxLimine `common/sys/smp_trampoline.asm_x86`）：

1. **16 位 @0**：`cli/cld` → `lgdt [cs:0x1A0]`（**16 位模式必须 disp16**，
   `disp32` 编码无效——首版 GDTR base/limit 全 0 的实锤）→ PAE → `CR0=PE|ET`
   → 远跳 0x08 段；
2. **32 位 @0x40**：数据段 0x10 → `EFER.LME` → `CR3 ← pml4` → `CR0.PG`（**PE 已在
   段1 置位**；实模式下单独置 PG 非法）→ `push 0x18/push off32/retf` 进 64 位段
   （兼容模式下 `jmp far` 到 L=1 段非法，#GP 实锤）；
3. **64 位 @0x80**：写存活标记 → 轮询 `SmpInfo.goto_address`（HHDM 虚地址）→
   非零则 `rsp=64KiB 栈`、`rdi=&SmpInfo`、`jmp rax`（brxlimine-rs lib.rs 537-543）。

自备 GDT（4 项：null/code32/data/code64）不依赖 OVMF 布局——但 **GDT 占
0x180..0x1A0，gdtdesc 必须放在 0x1A0 之后**：首版把 gdtdesc 放在 0x198
直接覆盖 code64 描述符，选择子 0x18 加载 → #GP e=0x18（异常日志实锤）。

IPI 时序（`final_ebs_and_jump`，**必须在切 CR3 之后**）：

- EBS → `switch_cr3`（我们的页表：恒等 + LAPIC + HHDM + 内核高区）→ `start_aps`
  → `jump_kernel`。trampoline 的 HHDM 轮询依赖我们的页表；且 OVMF 的 EBS 流程
  会把先前收编的 AP 重新挂起（`-d cpu_reset` 实证 11 次 CPU#1 reset），
  因此 AP 启动只能在 EBS 之后；
- LAPIC：SVR(0xF0) bit8 软件使能 + 伪向量 0xFF、TPR(0x80)=0（SDM 10.4.7）；
- INIT（delivery 5 + level assert + edge，目标 = 该 AP LAPIC id）→ 10ms 级延迟 →
  SIPI ×2（vector = tramp_phys>>12，SDM Vol3 8.4.4）；
- 存活判定：AP 进长模式首先写页内标记（+0x300 = 0x5A5A1234），BSP 轮询确认；
  未上线仅串口告警不阻塞（内核 `wait_all_online` 超时会打印归因）。

调试方法论（本轮全部依赖 QMP 内存/寄存器取证，因 EBS 后串口会静默丢行——
诊断行写入固定物理页再 QMP `xp` dump；`-d int` 抓 AP 异常现场：
IP/CR2/错误码逐条定位）。

验收（`tools/boottest.ps1 -Variant boot`，QEMU 追加 `-smp 2`）：

- 8 锚：内核 banner（跳转）、HHDM、RSDP rev=2、LazyBuddy done（memmap）、
  `[smp] BSP lapic_id=0`、`fired AP lapic_id=1`（响应结构 + 指针数组）、
  `[smp] AP online, lapic_id=1`（AP 启动全链）；
- 全 12 变体回归全绿。
## M8 的验证方式

M8 实现 Limine **模块请求**（`ModuleRequest`）：引导器把启动载荷作为模块
交给内核。关键设计是**按需加载**——只有内核映像声明了该请求标记时才读文件、
才分配内存（`boruix::has_request`）。BORUIX 内核不声明，因此它"介质即系统 /
单源"（ADR-017/028）架构完全不受影响：不读、不分配、行为零变化。

协议面（`src/modules.rs` + `src/boruix.rs`）：

- 请求标记 id `[0x3e7e279702be32af, 0xca1c4f3bd1280cee]`（brxlimine-rs 689）；
- `ModuleResponse { revision, module_count, modules }` + module_count 个
  `File` **指针**（ArrayPtr 语义，与 memmap/SMP 同款教训）；
- `File.base` / `path` / `cmdline` 都是 HHDM 虚地址（内核直接解引用）；
- 内存布局：LoaderData 结构页（File 数组 8×112B @0x000、指针数组 @0x400、
  字符串区 @0x500，编译期 `assert!` 钉死各区间不重叠）+ 每个模块内容独占
  页块；重转后的 memmap 里是 BootloaderReclaimable——EBS 后仍有效且不被
  BORUIX 的 pmm 回收；
- 模块清单是 `config::MODULES`（`limine.cfg` 的编译期等价物）：(ISO 路径,
  命令行)。上游 Limine 的模块清单来自配置文件，我们的编译期常量与之同构。

**独立消费者验收**（新增 `mod` 变体）：`tools/modtest` 是一个极小的 no_std
测试内核（静态 ET_EXEC、高半 vbase），只声明 `BaseRevision` + `ModuleRequest`
两个标记，打印收到的模块清单后停机。它与 BORUIX 无关，是"liftoff 真的把模块
交出去了"的第三方证据（同 `elf_oracle.py` 的真实链路思路）：

```
[modtest] baserev=0
[modtest] count=2
[modtest] m0 path=/MODULES/ALPHA.BIN len=4096 media=1 sum16=0xf800 cmd=role=alpha
[modtest] m1 path=/MODULES/BETA.BIN len=8192 media=1 sum16=0xf000 cmd=role=beta
```

期望值由 `tools/modtest_oracle.py` 从**同一份 fixture 字节**算出（生成模块
文件 + 生成断言，单一来源）；`modtest_oracle.py` 中模块表的 cmdline 与
`config.rs::MODULES` 必须一致，不一致会以断言失败自曝。

红态证据（实现前，`git stash -- src` 后实跑）：

```
PASS: contains [modtest] alive
FAIL: missing [modtest] baserev=0      ← 实际 baserev=6
FAIL: missing [modtest] count=2        ← 实际 "no module response"
```

**顺带修正的协议不合规**：`BaseRevision` 的语义是"引导器支持请求版本时把
该字段原位写 0"（brxlimine-rs `BaseRevision::is_supported()` 只认 `revision == 0`）。
M4–M7 一直传 `max_revision = 0`，于是声明 6 的内核（BORUIX/modtest）字段一直是 6
——等于引导器声称"不支持"。现在传 `config::LIMINE_BASE_REVISION`，声明版本 ≤ 我们
协议版本的一律写 0（红态里 `baserev=6`、绿态 `baserev=0` 是这条修复的直接证据）。

工具链改动：`tools/mkiso.py` 新增可重复的 `--extra ISO/PATH=HOST_FILE`
（单层目录，确定性布局；无 `--extra` 时输出与旧版逐字节相同）；
`tools/modtest/{main.rs,linker.ld,build.ps1}` 为消费者内核与其构建。
## M9 的验证方式

M9 让 liftoff 能引导**真实安装盘**：MBR 分区 → 分区内 EXT2 → 读内核 ELF →
完整交接（EBS/协议/页表/跳转），内核据此进入**安装模式**并把同一分区挂为根。

验收对象是项目自产的系统盘（`python tools/main.py build --systemdisk`）：

```
MBR: sig 0xAA55, disk_id 0x424f5255 ("BORU"), part1 status=0x80 type=0x83
     start_lba=2048 sectors=129024      ← 1MiB 偏移的 63MiB 分区
EXT2: 1KiB 块, 标签 BORUIX_SYS, /boot/kernel = 24,629,664 B ELF, /programs/*
```

liftoff 侧实现（`src/main.rs` M2c/M9 段 + `src/ext2.rs`）：

- **挂载顺序**：先试"整盘即文件系统"（M2c fixture 无 MBR 的口径，既有契约行
  逐字不变），失败再解析 MBR —— 签名 0xAA55 → 磁盘签名（offset 0x1B8）→
  选分区（活动分区 0x80 优先，否则首个非空项）→ `mount_at(part_lba × 512)`；
- **EXT2 读器两处扩展**：`Volume.base`（分区字节偏移，`mount`/`mount_at` 两个
  入口）与**二级间接块**（直块 12 + 单级 256 + 二级 256²，配 `MapCache` 缓存
  间接表——24.6MB/1KiB 块是 24576 次映射，无缓存会多出数万次 BlockIo）；
- **大文件读取路径**：内核直接读进 EfiLoaderData 常驻页（不再是 12KiB 栈缓冲），
  该区即 `File.base` 最终位置，省掉二次拷贝；
- **BootSource 参数化**：`File.media_type` / `partition_index` / `mbr_disk_id` 由
  启动来源决定（ISO → optical/0/0；磁盘 → generic/1-based 分区/MBR 签名）；
- **内核路径**：`/boot/kernel`（规范安装布局，与 `tools/limine.conf` 的
  `kernel_path: boot():/boot/kernel` 一致）优先，其次 `BOOT/KERNIMG.BIN`
  （M2c fixture），两者皆无则报 `M2C: open failed status=0x…`。

验收（`tools/boottest.ps1 -Variant ext-boot`，11 锚）：

```
[m2c] mbr disk_id=0x424f5255 partition=1 start_lba=2048   ← MBR/分区选择
M2C: mount ok                                             ← 分区内 EXT2 挂载
[m9] kernel path=/boot/kernel size=24629664               ← 二级间接读 24.6MB
M3: segs=3 entry=…                                        ← ELF 装载
Hello, BORUIX! / Kernel M0 is running.                    ← 交接后跳转成功
[mm] HHDM offset: 0xffff800000000000 / [acpi] RSDP rev=2
[boot] install mode detected: boot disk mbr_disk_id=0x424f5255 partition_index=1
partition lba=2048 (EXT2)                                 ← 内核按我方 BootSource 挂根
LazyBuddy init done
```

**红态与排障（每步都有实测证据）**：

1. 首次实现后 `M2C: open failed status=0x8000000000000002`（EFI_INVALID_PARAMETER）——
   我最初把失败状态写死成 EFI_NOT_FOUND，掩盖了真实错误；改为透传真错后才定位；
2. 在 `UefiBlock::read_at` 加一次性诊断，打出失败读参数：
   `off=0x3c09d7d880 block=503639020 last_block=131071` —— 偏移是垃圾；
3. 根因：`read_inode` 的 **GDT 偏移与 inode 表偏移没有加分区 `base`**
   （我只补了超级块与 `read_block`），于是从盘首读 GDT 得到垃圾 inode 表号，
   再据此算出天文数字的 inode 偏移。补 `self.base` 后一次通过；
4. 顺带给 `read_at` 加越界保护（`block > last_block` 时返回内部码 0x13，不再
   让 `span_blocks` 归零触发 `allocate_pool(0)`）。

工程注记：真实内核 24.6MB（带符号表）在 TCG 下从 EXT2 读完 + 装载 + 重定位需要
数分钟，`ext-boot` 变体超时设为 600s（其余变体 400s）；内核在该盘上还会继续走到
`[kmain] booting user init (PID 1) …`（盘内 `/programs` 有程序），但用户态启动
时长不稳定，故**不把 init 上线列为断言**，避免用例抖动。
## M10 的验证方式

M10 把模块交付（M8）扩展到 **EXT2 安装模式**：同一个内核声明同一个
`ModuleRequest`，liftoff 从**启动介质**（ISO 或 EXT2）装载模块。两条路径共用
一份装配逻辑与同一份 `config::MODULES` 清单。

实现（`src/modules.rs` + `src/main.rs` + `tools/mkext2.py`）：

- `modules.rs` 重构为**介质无关**：`Assembler` 负责结构页（File 数组 / 指针
  数组 / 字符串区）与每模块内容页；介质侧只提供两件事——"打开取大小"与
  "读进缓冲"（`load_from_iso` / `load_from_ext2` 各约 20 行）；
- `m2c`（EXT2 启动路径）在 ELF 装载成功后按 `boruix::has_request` 判定并调用
  `load_from_ext2`，与 ISO 路径（`m2b`）完全同构；
- `tools/mkext2.py` 从"单文件 ≤1KiB 固定布局"改写为**通用构建器**：任意单层
  目录树（`--extra ISO/PATH=HOST_FILE`，可重复）、多块文件（12 直块 + 一级
  间接）、按路径排序的确定性 inode/块分配；`--flat` 语义保留（ext-nopath）。

验收（`tools/boottest.ps1 -Variant ext-mod`）：测试内核 `tools/modtest` 经
EXT2 启动（内核文件本身 17,360 B → 走间接块），从同一 EXT2 卷读到两个模块：

```
[modtest] baserev=0
[modtest] count=2
[modtest] m0 path=/MODULES/ALPHA.BIN len=4096 media=1 sum16=0xf800 cmd=role=alpha
[modtest] m1 path=/MODULES/BETA.BIN len=8192 media=1 sum16=0xf000 cmd=role=beta
```

期望值仍由 `tools/modtest_oracle.py` 从同一份 fixture 字节算出（ISO `mod` 变体
与 EXT2 `ext-mod` 变体共用同一组期望——介质不同、交付语义相同）。

红态（`git stash -- src` 后实跑）：

```
PASS: contains [modtest] alive
PASS: contains [modtest] baserev=0      ← M8 的协议修正（已提交，未随本次 stash）
FAIL: missing [modtest] count=2        ← 实际 count=0（EXT2 侧尚未装载模块）
```

## M11 的验证方式

M11 让 AP 横向扩展：一台机器上**多个**辅助处理器同时上线。

修复的缺陷（M6 遗留，代码即证据）：

- `smp.rs` 里每个 AP 都申请 `ALLOCATE_ADDRESS` 的**同一个**页 `0x70000`，
  第二个 AP 起分配必然失败并 `break`——即 M6 实际只支持 1 个 AP；
- AP 数量上限写死为 4，且 `TRAMP_PAGES`/`AP_LAPIC_IDS` 静态数组定长 4。

现在：每 AP 独立 trampoline 页 `0x70000 + i×0x1000`（SIPI 向量 = 页号，
必须 < 256 → 页 < 1MB；同时避开 OVMF 的 AP 重定位缓冲 0x1F0000），上限提到
`MAX_CPUS`（8），静态数组随之定长。

验收（`tools/boottest.ps1 -Variant smp4`，QEMU `-smp 4`，13 锚）：

```
[smp] BSP lapic_id=0, total cpus=4
[smp] fired AP lapic_id=1 / =2 / =3        ← liftoff 发出 INIT-SIPI 并确认 AP 存活
[smp] AP online, lapic_id=1 / =2 / =3      ← 内核接管三个 AP（percpu/MSR/计时器）
[kmain] SMP done, 4 cpus online (target 4)
```

红态（`git stash -- src` 后实跑 `-smp 4`）：`total cpus=2`、只有 `fired AP lapic_id=1`
与 `AP online, lapic_id=1`，其余 6 条断言全红——正是"每 AP 抢同一页"的直接后果。

**边界（有意不做）**：x2APIC。MADT 的 type 9 条目与 MSR 形式 ICR（0x830）属
"引导器 + 内核"联合项——内核 arch 层目前只有 MMIO LAPIC（`[lapic] mapped to
0xffff8000fee00000`），liftoff 单方面切到 x2APIC 会让内核的 LAPIC 访问失效。
因此本里程碑只做 xAPIC 路径的横向扩展，x2APIC 留待内核侧同步支持。
## M12 的验证方式

M12 是**引导器 + 内核联合**里程碑：x2APIC。x2APIC 把 LAPIC 寄存器访问从
MMIO（0xFEE00000）改为 MSR（0x800 + 偏移/16），ICR 变成单次 64 位写
（MSR 0x830）。**x2APIC 使能后 MMIO 窗口不再代表 LAPIC**，因此两侧必须
同时支持并达成一致——这正是 Limine `SmpResponse.flags` bit0
（"X2APIC has been enabled"，brxlimine-rs lib.rs 552）的用途。

### 分工

| 侧 | 工作 |
| --- | --- |
| liftoff | CPUID.1:ECX[21] 探测；`IA32_APIC_BASE` bit10 使能 x2APIC；MADT **type 9**（x2APIC）条目枚举（无 type 9 时回退 type 0）；SVR/TPR/ID/ICR 全部按模式分派（MSR vs MMIO）；`SmpResponse.flags` bit0 上报 |
| 内核 | `lapic_read/write` 按模式分派到 MSR；`send_fixed_ipi` 在 x2APIC 下单次 64 位 MSR 写；`current_lapic_id()` 取全 32 位 id；`is_mapped()` 在 x2APIC 下不再依赖 MMIO 映射；`init()` 依 flags 选模式（x2APIC 不做 MMIO 映射）；**每个 AP 在入口最先把本核切到 x2APIC** |

### 验收

新增 `x2apic` 变体：`-cpu qemu64,+x2apic`（最小 CPU 模型 + 恰好打开 x2APIC）
与 `-smp 4`，15 锚全过：

```
[m6] x2apic supported=true enabled=true bsp_lapic=0   ← liftoff 探测并使能
[lapic] x2APIC mode: MSR access (id=0)                ← 内核按 flags 选同一模式
[smp] fired AP lapic_id=1 / =2 / =3                   ← MSR ICR 单次 64 位写启动 AP
[smp] AP online, lapic_id=1 / =2 / =3                 ← 三个 AP 全部被内核接管
[kmain] SMP done, 4 cpus online (target 4)
```

**xAPIC 回退路径同样被覆盖**：默认 `qemu64` CPU 无 x2APIC（实测
`[m6] x2apic supported=false enabled=false`），故既有 `boot`/`smp4` 变体跑的
就是 MMIO 路径（内核打印 `[lapic] mapped to 0xffff8000fee00000`），
`smp4` 变体新增该回退断言。两条路径因此都有真实链路覆盖。

### 红态与两个真实缺陷（都有实测证据）

1. **AP 未上线**（liftoff 已开 x2APIC、内核未在 AP 侧切模式）：AP 经 INIT 起来
   仍是 xAPIC，而内核的访问模式是全局的——`ap_entry` 第一句就调
   `current_lapic_id()`，在 xAPIC 的核上读 x2APIC MSR 触发 #GP，机器级联
   停机（日志停在 `[smp] fired AP lapic_id=1`）。修法：新增
   `lapic::ensure_x2apic_on_this_cpu()`，`ap_entry` 最前面调用。
2. **修好后又变成 SpinMutex 同核重入 panic**（`cpu: 2`，spin.rs:109）。根因是
   我在切换函数里加了一条日志：该函数运行在 AP 的 per-CPU 状态就绪**之前**，
   而 klib 串口锁用 `cpu_slot_id()`（返回槽位 + 1）做同核重入检测——槽位未就绪
   时它回退为常量 1，于是「BSP 持锁」被误判为「本核重入」并当场 panic。
   修法：切换保持静默（函数文档写明**不得打印日志**），并顺手修正 panic
   消息把槽位印成 `slot+1` 的误导（`me.wrapping_sub(1)`）。

工程注记：`-cpu max` 会在同一处触发另一个内核 panic（btree/alloc 栈帧），
与 x2APIC 无关（那是 `-cpu max` 暴露的其它特性路径），故本变体刻意选用
最小 CPU 模型 + 单一特性，把变量隔离到 x2APIC 一个维度。
## M13 的验证方式

M13 让 `ext-boot` 变体**自建系统盘**，不再依赖工作区里现成的 `systemdisk.img`
（那是由 `tools/main.py build --systemdisk` 产出的 64MiB 安装盘）。

### 新工具

- `tools/mksysdisk.py`：确定性 MBR + EXT2 系统盘构建器。**逐字段镜像规范盘的
  布局**，因此既有的锚（磁盘签名 / 分区号 / 起始 LBA）一字不改仍然成立：

```
MBR : sig 0xAA55, disk_id 0x424F5255("BORU"), part1 status=0x80 type=0x83
      start_lba=2048 sectors=129024        （CHS 全 0：内核 MBR 解析有意忽略 CHS）
EXT2: 1KiB 块，blocks_count=64512，/boot/kernel = 内核 ELF
```

- `tools/extverify.py`：**独立于 liftoff** 的夹具校验器——自己解析超级块/GDT/
  inode，映射直块 / 一级 / 二级间接，把文件读回来与宿主源文件**逐字节比对**。
  夹具构建器的缺陷在启动任何东西之前就能暴露。

### 顺带修掉的构建器缺陷（红态有实测证据）

`mkext2.py` 原先只支持直块 + 一级间接（上限 268 KiB）。直接用它装 8.9 MiB 内核
时它**不报错**，却把 8,480 个间接表项（33,920 B）写进了 1 KiB 的间接块——
越界覆盖了后续数据块。用新的校验器实测：

```
FAIL /BOOT/KERNIMG.BIN size=8957688 host=8957688
     first diff at byte 274432 (logical block 268): image=0x00 host=0x4b
```

差异位置恰好是 12 直块 + 256 一级间接的边界（268 × 1024 = 274432），之后全是零
（inode 的 i_block[13] 为 0，读作稀疏洞）。补上**二级间接**（l2 表 + 每 256 块
一个 l1 表）与 `min_blocks`（按分区容量定文件系统大小）后：

```
OK   /BOOT/KERNIMG.BIN size=8957688 sum16=0xc225
```

### 验收

`ext-boot` 现在自己产出 `target/systemdisk.img` 再引导。**当前 10/11 锚通过**：唯一
未过的是内核侧的安装模式根行 `partition lba=2048 (EXT2)`——自建盘尚未被内核接受为
可读写根（根因与修法见本节末「未完成项」）：

```
[m2c] mbr disk_id=0x424f5255 partition=1 start_lba=2048
M2C: mount ok
[m9] kernel path=/boot/kernel            ← 规范安装路径（非 M2c 夹具路径）
M3: segs=3 entry=…
Hello, BORUIX! / Kernel M0 is running.
[mm] HHDM offset: 0xffff800000000000 / [acpi] RSDP rev=2
[boot] install mode detected: boot disk mbr_disk_id=0x424f5255 partition_index=1
partition lba=2048 (EXT2)
LazyBuddy init done
```

工程注记：自建盘里没有 `/programs`（规范盘有，故那台还能起 PID 1），所以内核在
挂好安装模式根之后会停在「找不到 init」的路径上——本变体的验收点仍是引导器侧的
安装模式全链，用户态启动不在断言范围（M9 起即如此，避免用例抖动）。

### 多组元数据（把 ext-boot 收成 11/11 的修复）

自建盘能被 **liftoff** 正常挂载并读出 8.9 MiB 内核（`M2C: mount ok` +
`[m9] kernel path=/boot/kernel` + ELF 装载均通过），但最初**内核**在 `[boot] install mode
detected` 之后、打印 `[boot] install mode root` 之前停住。补上多组元数据后 11/11 通过。

已定位的根因（有日志与字段证据）：**夹具缺少分配器一致性元数据**。安装模式把启动分区
**读写**挂为根，内核随即 `build_skeleton` 在其中创建骨架目录——那需要分配 inode 与块，
而当前夹具：

- 块位图 / inode 位图**全零**（写成「全部空闲」，与真实 mkfs 镜像不一致）；
- `blocks_per_group=8192` 而 `blocks_count=64512` → 规范上应为 **8 个块组**，但只写了
  1 个组描述符，其余组的位图 / inode 表不存在；
- 组描述符与超级块的空闲计数为 0。

规范系统盘（`mke2fs` 产出）具备完整多组元数据，这正是 M9 能过而自建盘不能的原因。
**采用的修法**：1 KiB 块下按 8 个块组铺开——每组一个 32 B 组描述符、块位图、inode 位图与
inode 表（2 块），位图按实际占用写位，组描述符与超级块的空闲计数同源计算；inode 寻址改为
按组（`(ino-1)/ipg` → 组描述符 → 表块 + 组内下标）。`blocks_per_group = 8192` 是 EXT2
经典上限（8 × 块大小，保证位图装进一个块）。

（另一条路是改用 4 KiB 块：63 MiB / 4 KiB = 16128 块 ≤ 32768 单组上限 → 单组即可，但
liftoff 的 EXT2 读器目前只支持 1 KiB 块，需一并扩展；本次未走这条路。）

### 顺带修掉的验收脚本缺陷（假绿）

`boottest.ps1` 有两个会让「测试没跑」伪装成「测试通过」的问题，本次均已修复：

- **空期望列表也报 PASS**：变体若没定义任何断言，`$failed` 保持 false → 直接
  `BOOTTEST PASS`。本次正是在编辑脚本时误删了 ext-boot 的整段断言，从而得到一次假绿；
  现在空期望列表**硬失败**（`FAIL: variant ... produced no expectations`）。
- **最终判定依赖 QEMU 被杀后的日志**：`-serial file:` 的缓冲在强杀时可能未落盘（实测
  只剩 87 字节 / 13 KiB 截断），事后重读会拿到不完整日志。现在轮询命中即**快照**日志，
  判定使用快照。
**修复后的实测**（`ext-boot`，11/11）：

```
[m2c] mbr disk_id=0x424f5255 partition=1 start_lba=2048
M2C: mount ok
[m9] kernel path=/boot/kernel
[boot] install mode detected: boot disk mbr_disk_id=0x424f5255 partition_index=1
partition lba=2048 (EXT2)          ← 内核接受自建盘为读写安装根
```

离线侧同时用 `tools/extverify.py` 逐字节核对 `/boot/kernel`（8,957,688 B，走二级间接），
`OK /boot/kernel size=8957688 sum16=0xc225`。
## M14：liftoff 接入 tools（`--liftoff`）与 PID 1 实证

### 定位：两层验收，各管一段

| 层 | 位置 | 管什么 |
| --- | --- | --- |
| **引导器单元级夹具** | liftoff 仓库 `tools/boottest.ps1`（17 变体） | 引导器自身行为：M2a 文件读、M2b ISO9660、M2c EXT2、M3 ELF 契约、模块、AP、x2APIC 等；由 `mkiso/mkext2/mksysdisk/elf_oracle/modtest_oracle` 确定性夹具驱动 |
| **端到端验收** | **tools 仓库 `checks/liftoff/l1_boot_check.py`** | 项目规范流程产出的介质 + 真实用户态：OVMF → ESP 里的 liftoff → 介质上的 `/boot/kernel` → 内核 → **PID 1** |

以后新增的 liftoff 验收一律进 tools（唯一真值），liftoff 仓库只保留引导器单元级夹具。

### `--liftoff`：把「BIOS + brxLimine」换成「UEFI + liftoff」

- `tools_build/liftoff.py`：`build_efi()` / `stage_esp()` / `ensure_ready()` /
  `ovmf_firmware()` / `uefi_args()` / `qemu_exe()`；
- `main.py`：`build` / `run` / `br` 各加 `--liftoff`，与既有开关**正交**——
  `br --systemdisk --redisk --serial --release --liftoff` 等组合均可解析；
- `build.py`：`_make_iso` / `_make_system_disk` 增加 `liftoff_only`，**跳过全部
  BIOS 专属步骤**（brxLimine fork 产物拷贝、El Torito `-b` 引导项、isohybrid MBR、
  EXT2 上的 BIOS 引导码安装）。理由：liftoff 走 UEFI，读的是文件系统里的文件，
  与 BIOS 引导码无关 → 该链路不再依赖 i686-elf 交叉工具链；
- `run.py`：`--liftoff` 追加 OVMF pflash + ESP，`-boot order=c`；该模式不传
  `-cpu max`（该 CPU 模型会在内核侧触发与 liftoff 无关的 panic，M12 实测）。

### 实证：liftoff 引导下 PID 1 真的起来了

`python main.py run --systemdisk --serial --liftoff` → 规范系统盘（M9 那张）：

```
[m2c] mbr disk_id=0x424f5255 partition=1 start_lba=2048
M2C: mount ok
[m9] kernel path=/boot/kernel size=24629664
[boot] install mode detected: boot disk mbr_disk_id=0x424f5255 partition_index=1
[boot] install mode root = 'ata0' partition lba=2048 (EXT2), /programs = pool directory
[kmain] booting user init (PID 1) ...
[kmain] init: loaded 54096 bytes of init.elf from /programs
[kmain] init: entry=0x400000 stack_top=0x7ffefffffff0
[kmain] init: spawned pid=1 from init.elf
```

验收脚本 `tools/checks/liftoff/l1_boot_check.py --medium systemdisk` 即以此 6 锚
判定，实测 **LIFTOFF E2E PASS**。注意「看到 `booting user init`」不算数——该行是
无条件打印；成功分界行是 `init: loaded` 与 `init: spawned pid=1`。

### 边界

- **ISO 链**（`--liftoff` 的 liveCD 介质）在本机**无法实测**：ISO 组装依赖
  `xorriso`，而当前环境没有（`shutil.which` 与 `C:\ffmpeg\bin\xorriso.exe` 皆无）。
  跳过 BIOS 的改动已完成，装了 xorriso 即可用 `--medium iso` 验收 liveCD 的 `/programs` + init；
- **systemdisk 链**（纯 Python 组装，`disk.create_system_disk_image`）已实测通过，
  且不再需要 brxLimine fork 产物；
- `tools_build/util.py::err()` 只打印不退出，且写 stderr；本次排查中出现过「exit=1 但
  日志无任何错误行」的现象（疑似 stderr 编码/缓冲），建议后续给它加 flush 与明确退出码。

### M14 后续：liftoff 引导下 OS 的「最后一公里」状态（未解决，接续点）

**已修**：内核 `cpu::enable_features()` 现在为每个 CPU 补齐 `CR4.OSFXSR/OSXMMEXCPT/PGE`
（kernel 56dd1af）——此前 liftoff 的 AP trampoline 只置 PAE（AP CR4=0x20），AP 的
FPU/SSE 探测触发 #UD 并无限重试（30+ 次「CPU EXCEPTION」风暴），PID 1 永远轮不上。
brxLimine 的 trampoline 恰好置好了这些位，所以缺陷只在 liftoff 链路暴露。

**当前卡点（有实测证据）**：CR4 修复后异常风暴消失，但 init `spawned pid=1` 之后
**系统画面冻结**（QEMU monitor screendump 连续 6 张、60 秒内逐字节一致）、串口
无后续输出、AP 全部 idle —— init 进程没有被真正执行。疑似卡在首次切入用户态
（sysret/iret 到 ring 3）或 init 的第一个 syscall。

**重要教训（采集缺陷）**：QEMU 的 -serial file/tcp/stdio 在本环境都会丢/截数据
（file 缓冲截断在 939B/11K/12K 等 chunk 边界；tcp 会中途 reset）。-serial stdio
（用户终端直看）是唯一可靠的通道。已用 monitor screendump 抓帧缓冲做无串口观测
（_shot_probe.py，60 秒画面零变化 = 冻结实锤）。

**下一步排查方向**（按优先级）：

1. 对比 BIOS/brxLimine 与 liftoff 两条链路在 spawned pid=1 之后的内核状态：
   monitor info registers（逐 CPU）看 RIP/CR3 —— 是否还在内核 idle、还是已经
   sysret 到 0x400000；
2. 检查 init 的用户页表映射来源：两条链路的 memmap 条目数不同（实测 26 vs 28）、
   HHDM 一致但 pmm 的 usable regions 划分可能不同，用户空间分配是否踩了
   引导器遗留页（liftoff 的 ESP 页/参数页在 memmap 里的类型）；
3. 给 scheduler::start/首次 sysret 加临时串口标记，二分定位卡在哪一步。

（本轮结论：liftoff 引导器侧完整可用；OS 侧卡在用户态首切，未解决。）
### M14 定位（对照实验）：根因是 AP 的 LAPIC 未按 BSP 使能

方法：**同一内核 + 同一盘内容**，只换引导器（`build --systemdisk` 走 BIOS/brxLimine；
`build --systemdisk --liftoff` 走 UEFI/liftoff），在 `scheduler::start()` 的第一条 x87
指令前打印 CPU 状态，逐字段对比。

| 观测 | BIOS (brxLimine) | UEFI (liftoff) |
| --- | --- | --- |
| 异常风暴 | 0 | 0 |
| `[smp] AP online` ×3 | ✓ | ✓ |
| `[kmain] init: spawned pid=1` | ✓ | ✓ |
| `[sched] AP slot N pre-x87` | **✓ 3 个 AP 都到** | **✗ 从未出现** |
| `fp probe cw=` | ✓ `0x037f` | ✗ |
| `username:`（登录提示） | **✓** | ✗ |

结论：liftoff 下 AP 停在 `ap_entry` 末尾的 halt 循环，**从未被中断唤醒进入
`scheduler::start()`**（该路径靠 IRQ 唤醒后检查 `AP_SCHED_ENABLED`）。即 **AP 的 LAPIC
没有工作**。

brxLimine 的 `smp_trampoline.asm_x86` 第 55-74 行正是为此同步 BSP 的 `IA32_APIC_BASE`：

```asm
mov ecx, 0x1b          ; IA32_APIC_BASE
rdmsr                  ; 读 AP 当前值，处理 x2APIC/xAPIC 差异
.write_apic_msr:
    mov eax, [bsp_apic_addr_msr_lo]   ; 取 BSP 的值
    mov edx, [bsp_apic_addr_msr_hi]
    bts eax, 11        ; 置 APIC 全局使能
    btr eax, 8         ; 清 BSP 位
    wrmsr              ; 写到 AP
```

**我们的 TRAMP64 从不碰 `IA32_APIC_BASE`** —— 这是与 Limine 语义（"AP 的 CPU 状态与 BSP
一致"）的第一处实质缺口，也是当前"init 拉不起来"的直接原因。

另一处结论（顺带纠正）：**AP 的 CR4 在两条链路下都不含 OSFXSR/OSXMMEXCPT**（BIOS 的
`pre-x87` 显示 `cr4=0x310220`，其中 0x200 是内核自己 `enable_fpu()` 补的；`[cpu]` 行打印
时还没补）。所以此前"为 liftoff 给内核补 CR4 SSE 位"的改动是**多余且方向错误**的，
已回退（kernel c443713 / 4ddb0e4）。

### 与 Limine 语义的完整差距清单（待移植进 liftoff trampoline）

| # | brxLimine 做、liftoff 没做 | 影响 |
| --- | --- | --- |
| 1 | **同步 `IA32_APIC_BASE`**（bit11 使能、bit8 清、x2APIC 位按请求） | **AP 收不到中断 → 当前根因** |
| 2 | 用 **BSP 的 GDT**（16/32 位阶段用 BSP GDTR；64 位加 HHDM 后重新 `lgdt`） | AP 段选择子语义 |
| 3 | **清 TSS busy 位并 `ltr`**（BSP 的 TSS） | IST/异常栈 |
| 4 | 同步 **MTRR**（`mtrr_restore` 回调） | 内存类型（WC/UC） |
| 5 | `lapic_setup` 回调（引导器配置 AP 的 LAPIC） | AP LAPIC 就绪 |
| 6 | **`iretq` 进内核**（CS=0x28/SS=0x30/RFLAGS=0x2）并**清零全部 GPR** | AP 起始寄存器确定 |
| 7 | 进入前 **TLB flush**（`mov cr3` 自读自重） | 页表一致性 |
| 8 | 早期 `lidt invalid_idt` | 失败即三故障而非跳垃圾 |

### 协议合规（本轮已修）

- **`SmpRequest.flags` 被无视**（vendor lib.rs:585：bit0 = "Enable X2APIC, if possible"；
  BORUIX 传 0 = 明确不要开）。liftoff 曾只要 CPUID 支持就开 x2APIC，迫使内核必须支持 MSR
  访问 —— 这直接导致了那次错误的内核改动。现已在 liftoff 侧修正（读字段、按请求决定）；
- 带字段的请求只有三个：`SmpRequest.flags`（已实现）、`StackSizeRequest.stack_size` 与
  `PagingModeRequest.flags`（不填响应 = 诚实的"不支持"）。其余 16 个无字段。
### gen2 重写（分支 `rewrite/gen2`）

2026-09-30 起 liftoff 进入清空重写：`master` 保留原实现（M1–M14），`rewrite/gen2` 从空树
重建，按 brxLimine 的语义逐项移植。

分层规划（按职责，避免与具体内核耦合）：

- 入口层：UEFI 入口、panic、halt
- 固件绑定层：EFI 类型与协议（按需展开）
- 架构层：cpu、gdt、lapic、smp、trampoline
- 内存层：paging、memmap、mtrr
- 协议层：Limine 请求的声明扫描与响应填写
- 装载层：ELF、EXT2、ISO9660

当前状态：只有入口层与 COM1 串口输出；`cargo build --release --target x86_64-unknown-uefi`
通过。

重写的方法约束（来自 master 上的教训）：

- 汇编先离线验证：生成的 trampoline 必须反汇编核对指令序列与补丁偏移，再上机
- AP 启动语义逐项对齐 brxLimine：GDT 选择子（0x18/0x20 = 32 位、0x28/0x30 = 64 位、
  0x38 = TSS）与 `ltr`；`IA32_APIC_BASE` 同步（bit11 置、bit8 清）；MTRR 恢复按 SDM
  MemTypeSet 规程（defType 是 MSR 0x2FF，fixed MTRR 共 11 个）；LAPIC handoff（SVR = 0x1ff、
  TPR = 0、屏蔽可屏蔽投递模式的 LVT）；`iretq` 进入内核并清零全部 GPR
- 请求字段必须读：带字段的请求只有 `SmpRequest.flags`、`StackSizeRequest.stack_size`、
  `PagingModeRequest.flags`
- Rust 代码在 AP 上必须先置 `CR4.OSFXSR`（Rust 默认使用 SSE2；brxLimine 用 `-mno-sse` 回避）

master 上已定位但未解决的最后一步：内核在 BSP 第一次进入 PID 1 用户态时三重故障（无异常
横幅），与 AP 无关（单核模式同样复现）。
### gen2 进展：ADR-049 第 1 步（arch 抽象）已完成

分支 `rewrite/gen2`，提交序列 `cfbc674` → `1e56ea5`。工作区成员：`boot`、`protocol/limine`、
`arch/arch`、`arch/x86_64`、`arch/current`、`mm`、`fs`、`driver`、`loader`、`utils`、`efi`，
外加 vendored 的 `flanterm_rust`（path 依赖 + `[workspace] exclude`）。

已落地的抽象组件（全部宿主可测）：

- `arch::addr` —— `PhysAddr`/`VirtAddr`/`PhysFrame`/`PAGE_SIZE`/`Alignment`（对齐量以类型级
  不变量约束，非法对齐量无法构造）；溢出与截断一律返回 `Option`
- `arch::paging` —— `PageFlags`（语义位）、`MapError`、`validate_range`（各实现共用的单点校验）、
  `PageTable` trait（`map_range` + 带 SAFETY 契约的 `activate`）
- `arch::hhdm` —— `DirectMap`（带区间不变量的直接映射，区间外一律 `None`）
- `arch::platform` —— `Platform` trait（`name`/`halt`/`write_byte`/`disable_interrupts`/
  `restore_interrupts`）与 `InterruptState`

实现与装配：`x86_64::platform` 实现 `Platform`（`hlt`、COM1 轮询输出、`pushfq`+`cli`/`sti`）；
`current` 按 feature `impl-x86_64`（默认，目标构建）或 `impl-mock`（宿主测试）选择实现，
两者互斥且都未启用时 `compile_error!`。宿主测试固定用
`cargo test --no-default-features --features impl-mock`。

当前测试账目：**23 个宿主单测**（addr 9、paging 5、hhdm 4、platform 2、x86_64 1、mock 2），
`cargo build --release --target x86_64-unknown-uefi` 零警告。

待办（进入 QEMU 验证之前必须完成）：

- ADR-051 的离线反汇编核对尚无**可复现脚本**：`hlt`/`cli`/`sti` 曾人工核对通过，但 `out`/`in`
  的核对与脚本化未完成；该脚本应落在 `tools/checks/`（ADR-052 第 3 层）
- `arch` 的分页与 HHDM 抽象尚无实现接入（`x86_64` 侧尚未实现 `PageTable`）


