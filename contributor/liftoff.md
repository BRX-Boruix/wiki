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
