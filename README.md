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

[`contributor/`](contributor/) 按仓库收录实现层的内容：设计取舍、约束，以及修改对应代码时需要
知道的约定。每个仓库一份；README 规约见 [`contributor/readme.md`](contributor/readme.md)。

守护进程与系统服务：

- [`audiod`](contributor/audiod.md) —— 音频混音
- [`consoled`](contributor/consoled.md) —— 键盘事件到控制台字节
- [`driverd`](contributor/driverd.md) —— 驱动自动装载
- [`userd`](contributor/userd.md) —— 账户与家目录同步
- [`volumed`](contributor/volumed.md) —— 卷的挂载与拔除

验收与测试程序：

- [`acee2e`](contributor/acee2e.md) —— 访问控制规则的系统调用链路
- [`audioe2e`](contributor/audioe2e.md) —— 音频阻塞唤醒往返
- [`blkdemo`](contributor/blkdemo.md) —— 旧字节路径的对照诊断
- [`consoled-e2e`](contributor/consoled-e2e.md) —— 控制台环阻塞唤醒
- [`evdemo`](contributor/evdemo.md) —— 事件流的诊断形态
- [`evsrcdemo`](contributor/evsrcdemo.md) —— 事件源组件
- [`focusdemo`](contributor/focusdemo.md) —— 焦点门禁的对抗验收
- [`fpcheck`](contributor/fpcheck.md) —— 浮点路径观察
- [`pwde2e`](contributor/pwde2e.md) —— 账户查询链路
- [`selftest`](contributor/selftest.md) —— 按需自检宿主
- [`spinburn`](contributor/spinburn.md) —— 终止压测的目标进程
- [`synce2e`](contributor/synce2e.md) —— 同步字阻塞往返
- [`threaddemo`](contributor/threaddemo.md) —— 多线程与线程本地存储
- [`tokendemo`](contributor/tokendemo.md) —— 控制台令牌真值
- [`trave2e`](contributor/trave2e.md) —— 目录遍历权限
- [`yielder`](contributor/yielder.md) —— 忙让出压测

库与系统程序：

- [`libline`](contributor/libline.md) —— 行编辑库
- [`libsys`](contributor/libsys.md) —— 用户态系统调用库
- [`login`](contributor/login.md) —— 登录认证

## 项目历史

[`Boruix.md`](Boruix.md) 记录这个项目的来龙去脉，包括各代版本的更替与结果。

## 许可

本 wiki 内容采用 MIT License，详见 [LICENSE](LICENSE)。
