---
title: Customize
date: 2023-10-16 20:46:42
updated: 2024-07-21 10:25:21
tags:
  - Fedora
  - Linux
categories:
  - Fedora
---

&emsp;&emsp;让 Fedora 变得更好用。

<!-- more -->

## Extension

&emsp;&emsp;标准 GNOME 可谓是相当难用了，想提升体验必须装插件。启用 GNOME 插件可以使用 Extension Manager 工具，这个工具可以用 Flatpak 安装。

&emsp;&emsp;下面是我使用的一些插件：

| Name                                         | Function             |
| -------------------------------------------- | -------------------- |
| Top Hat                                      | 顶栏的资源可视化工具 |
| Extension List                               | 顶栏的插件管理工具   |
| Clipboard Indicator                          | 剪切板工具           |
| User Avatar In Quick Settings                | 头像附近的快捷设置   |
| AppIndicator and KStatusNotifierItem Support | 应用托盘             |
| Caffeine                                     | 控制休眠定时         |
| User Themes                                  | 第三方主题支持       |
| RebootToUEFI                                 | 重启到 UEFI          |
| Dash to Dock                                 | Dock 支持            |
| Dash to Panel                                | 更好的顶栏           |
| GSConnect                                    | KDEConnect 移植版    |
| Blur my Shell                                | 毛玻璃效果           |
| Burn My Windows                              | 窗口开关动画         |
| Coverflow Alt-Tab                            | 工作区切换动画       |
| Compiz windows effect                        | 窗口拖动动画         |

&emsp;&emsp;大部分插件都能正常工作，但是`Dash to Dock`和`Dash to Panel`会有一点冲突。

## Theme

&emsp;&emsp;GNOME 支持用户自定义主题，包括 Shell，Cursor，GTK 等。使用 User Themes 插件可以启用自定义主题功能，然后可以使用 tweaks 工具切换主题 (使用 DNF 安装`gnome-tweaks`)。

&emsp;&emsp;下面是我使用的一些主题：

|        | Theme              | Remark                   |
| ------ | ------------------ | ------------------------ |
| GTK    | Orchis gtk theme   | 只对 legacy app 生效     |
| GRUB   | Vimix              | 我这不生效               |
| Shell  | Marble Shell theme | 装在 ~/.themes 下        |
| Cursor | Bibata Modern Ice  | 装在 /usr/share/icons 下 |

## Fonts

&emsp;&emsp;Fedora 预装的是可变字体，很多软件，如 QQ音乐，欧陆词典，WPS 等都不支持可变字体，导致界面出现小方块，无法使用。安装非可变版本的 sans 字体可以解决：

```bash
sudo dnf install google-noto-sans-cjk-fonts
```

&emsp;&emsp;WPS 需要使用一些其他的字体，这些字体要装在 /usr/share/fonts/wps-fonts 下。注意给目录设置好权限，以使 WPS 可以访问，不然还是检测不到。

## Input

&emsp;&emsp;输入法目前我只用过 ibus 系列的：`libpinyin`和`rime`。

### Libpinyin

&emsp;&emsp;libpinyin 可以在 Settings 里直接安装，并且配置修改都在 GUI 里进行，用起来比较简单。

&emsp;&emsp;这个输入法在 Ubuntu 上比较好用，在我印象中，Ubuntu 上的 libpinyin 支持云词库。

### Rime

&emsp;&emsp;libpinyin 虽然简单，但是扩展性不够好 (稳定性也一般)。rime 是另一个较好的选择，它支持插件以及很多其他扩展功能，并且词库同步也很方便，使用 DNF 安装 ibus-rime。

## Desktop

&emsp;&emsp;无论是 DNF 还是 Flatpak，提供的软件包其实都相当有限。很多工具，如 Eudic，Nekobox，Chatbox 等都没有提供 rpm 或者 flatpak 包，这时候就需要手动配置 desktop 文件，使其像其他应用一样正常显示在 App 菜单中。

1. 获得可执行文件：通常在官网或者 GitHub 上下载可执行文件或 Apppimage 文件。
2. 制作 desktop 文件，下面是我常用的模板：

```ini
# ~/.local/share/applications/eudic.desktop

[Desktop Entry]
Name=Eudic
Terminal=false
Type=Application

Comment=Eudic
Categories=Utility;
StartupWMClass=Eudic

Exec=/home/arthur/.local/share/apps/eudic/apprun
Icon=/home/arthur/.local/share/apps/eudic/icon.png
```

&emsp;&emsp;重点有三个：Exec，Icon 和 StartupWMClass。Exec 是应用的启动命令，我一般都会在相应文件夹内创建一个 apprun 脚本来统一处理。Icon 是应用的图标路径，我将其和 apprun 放在一起。

&emsp;&emsp;StartupWMClass 是系统用来区分运行中应用的标识：如果某个应用启动了两个窗口，若其真实的 StartupWMClass 与 desktop 文件中的相同，则在 Dock 栏中只占一个位置，否则会占两个 Dock 栏位置。

&emsp;&emsp;对于 Wayland 应用，查询 StartupWMClass 可以使用插件`Looking Glass Button`。
