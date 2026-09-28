# wiki

BORUIX 的**文档站**：面向使用者的教程与手册，以及面向维护者的各仓库实现说明。

[English](README.en.md)

## 目录

| 路径 | 内容 | 状态 |
| --- | --- | --- |
| [`tutorial/`](tutorial/) | 入门教程 | 已有 |
| [`contributor/`](contributor/) | 各仓库的实现说明 | 已有 |
| [`Boruix.md`](Boruix.md) | 项目历史 | 未完成 |
| `install/` | 安装指南 | 待补 |
| `usage/` | 使用手册：命令、shell、配置 | 待补 |
| `faq/` | 常见问题 | 待补 |

## 教程

### [编写与安装驱动程序](tutorial/write-and-install-drivers.md)

怎么给系统加一个新硬件的支持。

起点是一个反直觉的事实：在 BORUIX，**驱动不是内核模块，而是一个普通的用户态程序**。装驱动只是把
文件放进系统的驱动目录、并声明它绑定哪个设备，**不需要重新编译系统本身**。

| 章节 | 内容 |
| --- | --- |
| 0 | 驱动在 BORUIX 里到底是什么 |
| 1 | 快速安装一个现成驱动（不需要编译） |
| 2 | 自己动手写一个驱动 |
| 3 | 把自己的驱动打进系统镜像 |
| 4 | 完整流程速览（写 → 编 → 装 → 验） |
| 5 | 常见疑问 |

> 已经是编译好的驱动就只看第 1 章；要从零写一个，再从第 2 章开始。

## 各仓库的实现说明

[`contributor/`](contributor/) 收录各个仓库的**实现层说明**——设计取舍、约束、踩过的坑。这些内容
对阅读或修改对应代码的人是必要的，但不适合放在仓库自己的 README 里。

| 文件 | 仓库 |
| --- | --- |
| [`contributor/libsys.md`](contributor/libsys.md) | 用户态系统调用库 |
| [`contributor/libline.md`](contributor/libline.md) | 行编辑库 |
| [`contributor/login.md`](contributor/login.md) | 登录认证程序 |

## 项目历史

[`Boruix.md`](Boruix.md) 记录这个项目的来龙去脉——几代版本的更替、每一代为什么结束、留下了什么。

> 该文稿**尚未完成**，最后一部分只有标题。

## 关于本仓库

这里的文档分两类：面向**使用者**的教程与手册，以及面向**维护者**的各仓库实现说明（[`contributor/`](contributor/)）。

## 许可

MIT License，版权归 Yang Borui 所有。详见 [LICENSE](LICENSE)。
