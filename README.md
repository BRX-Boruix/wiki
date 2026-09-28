# wiki

BORUIX 的文档站，收录面向公众的使用说明，以及面向维护者的各仓库实现说明。

[English](README.en.md)

## 目录

- [`tutorial/`](tutorial/) —— 教程
- [`contributor/`](contributor/) —— 各仓库的实现说明
- [`Boruix.md`](Boruix.md) —— 项目历史

## 教程

### [编写与安装驱动程序](tutorial/write-and-install-drivers.md)

怎么给系统加一个新硬件的支持。

驱动是用户态程序，不是内核模块。安装驱动只是把文件放进系统的驱动目录，并声明它绑定哪个设备，
不重新编译系统。

教程分两部分：

- 第 1 章：安装一个编译好的驱动，不涉及编译器
- 第 2 至 5 章：自己写一个驱动，从程序结构讲到打包进系统镜像

## 各仓库的实现说明

[`contributor/`](contributor/) 按仓库收录实现层的内容：设计取舍、约束，以及修改对应代码时需要知道
的约定。README 规约见 [`contributor/readme.md`](contributor/readme.md)。

- [`contributor/readme.md`](contributor/readme.md) —— README 规约
- [`contributor/libsys.md`](contributor/libsys.md) —— 用户态系统调用库
- [`contributor/libline.md`](contributor/libline.md) —— 行编辑库
- [`contributor/login.md`](contributor/login.md) —— 登录认证程序

## 项目历史

[`Boruix.md`](Boruix.md) 记录这个项目的来龙去脉，包括各代版本的更替与结果。

## 许可

本 wiki 内容采用 MIT License，详见 [LICENSE](LICENSE)。
