---
title: Fedora Backup
date: 2023-10-16 20:46:42
tags:
    - Fedora
categories:
    - [Linux, Fedora]
---

&emsp;&emsp;我的一些系统配置和插件。

<!-- more -->

## Extension

&emsp;&emsp;先用 Flatpak 安装 Extension Manager，或者直接用 Firefox 的 GNOME 插件。

| Name                                         | Function                   |
| -------------------------------------------- | -------------------------- |
| Top Hat                                      | 窗口顶栏的资源可视化工具。 |
| Extension List                               | 窗口顶栏的插件小工具。     |
| Transparent Topbar                           | 窗口顶栏透明背景。         |
| Clipboard Indicator                          | 窗口顶栏的剪切板小工具。   |
| Places Status Indicator                      | 窗口顶栏的快捷地址工具。   |
| User Avatar In Quick Settings                | 用户头像附近的快捷设置。   |
| AppIndicator and KStatusNotifierItem Support | 窗口顶栏的小图标。         |
| Caffeine                                     | 关闭桌面定时关闭。         |
| Blur My Shell                                | 毛玻璃效果。               |
| User Themes                                  | 开启第三方主题支持。       |
| Dash To Dock                                 | 开启 Dock 支持。           |
| RebootToUEFI                                 | 重启到 UEFI 的快捷键。     |
| Coverflow Alt-Tab                            | 工作区切换动画。           |
| Launch new instance                          | 总是启动一个新实例。       |
| Compiz windows effect                        | 窗口拖动动画。             |

## Theme

&emsp;&emsp;先安装 User Themes 插件 (Extension)，然后安装 tweak 工具：gnome-tweak (dnf)。

|        | Theme              | Remark                                                       |
| ------ | ------------------ | ------------------------------------------------------------ |
| GTK    | Orchis gtk theme   | 只对 legacy app 生效。把主题中的 GTK4 文件 (~/.themes/theme/gtk4.0) copy 到 ~/.config/gtk4.0 目录中，可改变 GTK4 应用的主题。 |
| GRUB   | Vimix              | 我这不生效。                                                 |
| Shell  | Marble Shell theme | 可以装在用户目录 ~/.themes 下。                              |
| Cursor | Bibata Modern Ice  | 装在 /usr/share/icons 而不是 ~/.icons 下。                   |

## Fonts

&emsp;&emsp;这不是要修改界面显示字体，而是修复国内部分软件中文乱码的问题。Fedora 默认装的字体大部分是先进的可变字体，可惜很多软件：QQ 音乐，欧陆词典，WPS 等都不支持。导致出现小方块，无法使用。

&emsp;&emsp;两步即可，其实就是把 sans 字体改成非可变版本：

```bash
sudo dnf remove google-noto-sans-cjk-vf-fonts.noarch
sudo dnf install google-noto-sans-cjk-fonts.noarch
```

&emsp;&emsp;WPS 要使用一些其他的字体，装在 /usr/share/fonts/wps-fonts 下。

&emsp;&emsp;注意给目录设置好权限，以使 WPS 可以访问，不然还是检测不到。

## Libpinyin

&emsp;&emsp;输入法还是很重要的，我目前只用过 ibus 系列的：libpinyin 和 rime。我对输入法不怎么感冒，之前在 Windows 上也只用微软拼音。

&emsp;&emsp;直接安装 ibus-libpinyin 即可。如果要导入词库，可以使用深蓝 imewlconverter 转换。

&emsp;&emsp;好像 Ubuntu 22.04 上的 libpinyin 都支持云词库了，难绷。

## Rime

&emsp;&emsp;libpinyin 也就凑活用用，最终答案还是 rime。

&emsp;&emsp;先安装 ibus-rime。