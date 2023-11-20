---
title: Shell 01 - 基础
date: 2023-11-20 19:47:14
updated: 2023-11-20 20:22:51
tags:
  - Fedora
categories:
  - [Linux, Fedora]
  - [Linux, Shell]
---

&emsp;&emsp;我的 Linux Shell 笔记，但是带有一些 Fedora 色彩。

<!-- more -->

## 进入 CLI

&emsp;&emsp;现在用户级的 Linux 发行版几乎都带有 DE，用户一开机就进入了 GUI。虽然 GUI 是趋势 (Windows 已经证明了这一点)，但既然选择了 Linux，怎能不了解 CLI。

&emsp;&emsp;早期的 Unix 系统只有 CLI 界面，所有与系统的交互任务都要通过终端进行。Unix 系统使用的终端是哑终端：由一组通信电缆连接到 Unix 系统的键盘和显示器。而现代的 Linux 系统使用的是`虚拟终端`，它模拟的是早期的硬接线控制台终端，并且`虚拟终端`是和 Linux 系统交互的直接接口。

&emsp;&emsp;Linux 系统在启动时会创建多个`虚拟控制台`，虚拟控制台是运行在内存中的终端会话。Fedora 在启动时会创建 6 个虚拟控制台，通过单个键盘和显示器就能访问这些虚拟控制台。

&emsp;&emsp;Fedora 启动的 6 个虚拟控制台中：第一个常常被用来运行 Login GUI。剩余几个要么空闲，要么是已登录用户的 DE 工作空间。从 GRUB2 引导内核开始，到 Fedora 启动 Login GUI，中间的时间很短。但即使已经进入了 GUI，还是有一种方法可以快速进入 CLI：

&emsp;&emsp;按下组合快捷键 (即使是在 GUI 环境中)：

```
Ctrl + Alt + Fx
```

&emsp;&emsp;这里的 Fx 可以是 F1 到 F6，它分别对应 6 个虚拟控制台。

&emsp;&emsp;Linux 提供的虚拟控制台并不依赖 GUI 和任何 DE。这意味着：当 DE 崩溃时，虚拟控制台将是少有的几种能访问系统的途径之一。
