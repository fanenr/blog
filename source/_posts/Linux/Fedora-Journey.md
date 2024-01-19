---
title: Fedora Journey
date: 2023-10-16 20:46:42
updated: 2024-01-19 12:17:30
tags:
  - Fedora
categories:
  - [Linux, Fedora]
---

&emsp;&emsp;我的 Fedora 折腾之旅。

<!-- more -->

## Extension

&emsp;&emsp;标准 GNOME 可谓是相当难用了，想提升体验必须装插件。启用 GNOME 插件可以使用 Extension Manager 工具，这个工具可以用 Flatpak 安装。

&emsp;&emsp;下面是我使用的一些插件：

| Name                                         | Function               |
| -------------------------------------------- | ---------------------- |
| Top Hat                                      | 顶栏的资源可视化工具   |
| Extension List                               | 顶栏的插件管理小工具   |
| Clipboard Indicator                          | 剪切板小工具           |
| User Avatar In Quick Settings                | 用户头像附近的快捷设置 |
| AppIndicator and KStatusNotifierItem Support | 启用应用托盘           |
| Caffeine                                     | 控制桌面暂停定时       |
| User Themes                                  | 开启第三方主题支持     |
| RebootToUEFI                                 | 快捷重启到 UEFI        |
| Dash To Dock                                 | 开启 Dock 支持         |
| Blur My Shell                                | 毛玻璃效果             |
| Coverflow Alt-Tab                            | 工作区切换动画         |
| Compiz windows effect                        | 窗口拖动动画           |

## Theme

&emsp;&emsp;GNOME 支持用户自定义主题，包括 Shell，Cursor，GTK 等。使用 User Themes 插件可以启用自定义主题功能，然后可以使用 tweaks 工具切换主题，可以使用 DNF 安装 gnome-tweaks。

&emsp;&emsp;下面是我使用的一些主题：

|        | Theme              | Remark                   |
| ------ | ------------------ | ------------------------ |
| GTK    | Orchis gtk theme   | 只对 legacy app 生效     |
| GRUB   | Vimix              | 可惜在我这不生效         |
| Shell  | Marble Shell theme | 装在 ~/.themes 下        |
| Cursor | Bibata Modern Ice  | 装在 /usr/share/icons 下 |

## Fonts

&emsp;&emsp;Fedora 默认预装的字体大部分是可变字体，可惜很多软件：QQ 音乐，欧陆词典，WPS 等都不支持，导致界面出现小方块，无法使用。

&emsp;&emsp;解决方案是：安装非可变版本的 sans 字体：

```bash
sudo dnf install google-noto-sans-cjk-fonts.noarch
```

&emsp;&emsp;WPS 需要使用一些其他的字体，这些字体要装在 /usr/share/fonts/wps-fonts 下。注意给目录设置好权限，以使 WPS 可以访问，不然还是检测不到。

## Libpinyin

&emsp;&emsp;输入法目前我只用过 ibus 系列的：libpinyin 和 rime。libpinyin 可以在 Settings 里直接安装，并且配置修改都在 GUI 里进行，用起来比较简单。

&emsp;&emsp;这个输入法在 Ubuntu 上比较好用，我印象中在 Ubuntu 上的 libpinyin 支持云词库。

## Rime

&emsp;&emsp;libpinyin 虽然简单，但是扩展性不够好。rime 是另一个较好的选择，它支持插件以及很多其他扩展功能，并且词库同步也很方便。

&emsp;&emsp;可以使用 DNF 安装 ibus-rime。