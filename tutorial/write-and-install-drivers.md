# 编写与安装驱动程序（面向普通用户）

> 适用版本：当前 BORUIX（/modules 运行时驱动安装 + UIO 用户态驱动机制已落地）。
> 本文面向**普通用户**：不要求理解内核，只讲「怎么写一个驱动、怎么把它装进系统并让它跑起来」。

## 0. 驱动在 BORUIX 里到底是什么

在 BORUIX，一个驱动 **不是内核模块**，而是一个**普通用户态 ELF 程序**：

- 它是一个 no_std 可执行文件，开机后由装载方把它跑起来；
- 它通过 libsys 的 UIO 系统调用去「认领」一个真实设备、把该设备的 MMIO 窗口映射进自己的地址空间，然后直接读写硬件寄存器；
- 设备到底驱动哪个，靠**传给它的参数（设备名）**决定，不写死在 ELF 里。

所以「装驱动」不等于「改内核 / 重新编译系统镜像」。新硬件支持可以完全在用户态完成。

> **一句话**：写驱动 = 写一个会 register/claim/读写 MMIO 的 ELF；装驱动 = 把它放进 /modules 并声明绑哪些设备。

---

## 1. 快速安装一个现成驱动（不需要编译）

如果你已经有一个编译好的驱动 ELF（或直接用系统自带的示例 /programs/userdrv.elf），装起来只要两步，全在 shell 里完成。

### 第 1 步：把驱动装进系统

在 shell 里执行：

    driver install <源ELF路径> <驱动名> [绑定的设备]

例如：

    driver install /programs/userdrv.elf mynet pci-ethernet-00-03-0

它做这几件事（对你透明）：

- 把源 ELF 复制为 /modules/<驱动名>/driver.elf；
- 生成 manifest.json 声明文件（记录名称、二进制、绑定设备）；
- 把这些文件标记为仅系统可写/可执行（安全门禁）。

**绑定设备怎么写？**

- **具体设备名**：pci-ethernet-00-03-0（只认领这一台）；
- **类别通配**：pci-ethernet-*（认领所有前缀匹配的设备，例如给同一型号网卡批量装驱动）。

设备名要来自系统里**真实注册**的设备，执行 cat /devices/list 可查到当前设备。

### 第 2 步：让驱动跑起来（二选一）

驱动装好之后由装载方拉起，有两种方式：

**方式 A —— 让守护进程 driverd 自动装载（推荐）**

常驻守护进程 driverd 周期性扫描 /modules。只要驱动声明绑定的设备当前真实存在且还没被其它驱动认领，driverd 就会自动拉起它，让驱动**持续驻留**服务该设备（长驻模式）；驱动进程崩溃或被结束，driverd 会自动重新拉起。

装完即生效，不用手动操作。这是自动装载（P2-1）+ 崩溃自动重载（P2-2）的能力。

**方式 B —— 手动瞬时装载（自检用）**

    driver load <驱动名>

立刻把驱动 spawn 起来，做一次「认领设备 → 映射 MMIO → 读一次寄存器 → 释放退出」，用来确认驱动装对了、能读到设备。

### 第 3 步：确认装好了

    driver list              # 列出 /modules 下已装驱动及其声明
    driver status <设备名>    # 查某设备认领状态（看 uio_claimed 是否为 true）

若 driver status 显示 uio_claimed:true，说明设备已被你的驱动成功接管。

---

## 2. 自己动手写一个驱动

系统里没有现成驱动时，你需要自己写。BORUIX 给了可直接照抄的模板：**userdrv**（源码在工程 userdrv/ 目录）。下面以它为例。

### 2.1 写代码

驱动是一个 Rust no_std 程序，核心逻辑如下（示意）：

    use libsys::*;
    // 入口：参数 argv 里带目标设备名
    pub extern "C" fn user_main(_argc, argv) -> i32 {
        // 1) 从 argv[0] 取出目标设备名
        // 2) 只读查设备状态（可选）
        driver_query(dev_name);        // -> {"device":..,"uio_claimed":..}
        // 3) 认领设备 + 映射 MMIO
        let uio_id  = driver_register(dev_name);  // 返回 uio_id
        let mmio_va = driver_claim(uio_id);       // 返回映射后的用户地址
        // 4) 读写硬件寄存器
        let reg = read_volatile(mmio_va as *const u32);
        write_volatile(mmio_va as *mut u32, 0x1);
        // 5a) 瞬时模式：释放并退出
        driver_unregister(uio_id);
        // 5b) 长驻模式：不释放，持续循环服务，直到进程被结束
        //     内核在进程退出/被杀时自动释放认领，driverd 据此自动重拉
    }

**必须知道的四个 UIO 调用**（来自 libsys）：

- driver_register(设备名)：认领设备，返回 uio_id；
- driver_claim(uio_id)：映射该设备 MMIO 窗口，返回用户态地址；
- driver_query(设备名)：只读查设备绑定/认领状态（JSON）；
- driver_unregister(uio_id)：释放认领。

> **注意**：driver_register / driver_claim 是 **System-only**（安全门禁）。由 driverd、driver load 等系统路径拉起没问题；若被普通 User 身份进程拉起，会返回 PermissionDenied 并如实失败——这是权限模型，不是 bug。

### 2.2 工程配置

直接复制 userdrv/ 工程结构即可：

- Cargo.toml：依赖 libsys，release 配 panic="abort"、opt-level="z"、lto=true；
- build.rs + linker.ld：把 ELF 定位到用户态地址（ENTRY(_start)、ET_EXEC），照抄不改；
- 编译：cargo build --target x86_64-unknown-none --release，得到驱动 ELF。

### 2.3 哪些设备能被驱动认领

只有内核 DriverHub 已注册、并发布了 MMIO 窗口的设备才能被认领（如网卡 pci-ethernet-00-03-0、显卡 pci-vga-display-00-02-0）。没有 MMIO 窗口的设备 driver_claim 会返回 NotSupported。

---

## 3. 把自己的驱动打进系统镜像

自己写的驱动要被「安装」，需先作为 liveCD 里的一个程序存在。参照 tools/tools_build/build.py 里 userdrv 的登记方式，把你的 ELF 加进 programs 列表、payloads 和 systemdisk 元组，然后重建镜像：

    python main.py build   # 在 tools 目录下

构建完成后，新 ISO 里就有 /programs/<你的驱动>.elf，之后就能 driver install 安装了。

> 构建是唯一需要接触工程/工具链的一步；日常「安装 / 装载 / 自动加载」都在运行中的 shell 完成。

---

## 4. 完整流程速览（写 → 编 → 装 → 验）

| 步骤 | 做什么 | 动作 |
|---|---|---|
| 1 写 | 照 userdrv 写认领 + MMIO 逻辑 | 编辑源码 |
| 2 编 | 编出用户态 ELF | cargo build --target x86_64-unknown-none --release |
| 3 打进镜像 | 登记进 build.py 并重建 | python main.py build |
| 4 装 | 拷到 /modules + 写 manifest | driver install /programs/<名字>.elf <名> <设备> |
| 5 跑 | 自动（driverd）或手动 | 自动无需动作；手动 driver load <名> |
| 6 验 | 看认领状态 | driver list、driver status <设备> |

---

## 5. 常见疑问

- **会不会把内核弄崩？** 驱动是独立用户态进程，且有 System-only 门禁。驱动崩溃/被杀时内核自动隔离其认领（0 panic），driverd 还能自动重拉。
- **装完要重启吗？** 不用。driverd 会自动发现新装驱动并拉起。
- **驱动挂了会自动恢复吗？** 会。driverd 检测到驱动进程退出后会自动重拉并重新认领（P2-2）。
- **为什么我装的驱动没生效？** 最常见是绑定设备名写错 / 设备不在 / 已被别的驱动认领。用 driver status <设备> 查 uio_claimed 是否为 true 即可定位。

