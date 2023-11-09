---
title: Fedora Dnf
date: 2023-10-21 10:52:18
updated: 2023-11-09 18:02:09
tags:
  - Fedora
categories:
  - [Linux, Fedora]
---

&emsp;&emsp;一些必备且常用的 Dnf 用法。

<!-- more -->

## 安装

&emsp;&emsp;命令：`install`

&emsp;&emsp;别名：`in`

```bash
sudo dnf [options] install <spec>...
```

&emsp;&emsp;例子：

```bash
sudo dnf install tito

# install from local file
# also support network URL
sudo dnf install ~/Downloads/tito.rpm

# install specify version
sudo dnf install tito-0.5.6-1.fc22

# install Group or Module
sudo dnf install '@docker'
```

&emsp;&emsp;命令：`reinstall`

&emsp;&emsp;别名：`rei`

```bash
sudo dnf [options] reinstall <package-spec>...
```

## 卸载

&emsp;&emsp;命令：`remove`

&emsp;&emsp;别名：`rm`

```bash
sudo dnf [options] remove <package-spec>...
```

&emsp;&emsp;remove 默认启用 clean_requirements_on_remove 选项。启用这个选项时，remove 将会自动卸载与该软件包相关的但是`不被使用`的其他软件包。

&emsp;&emsp;命令：`autoremove`

```bash
# remove all unused packages
sudo dnf [options] autoremove

sudo dnf [options] autoremove <spec>...
```

&emsp;&emsp;如上所言，默认的 remove 就是 autoremove 的别名。

## 更新

&emsp;&emsp;命令：`upgrade`

&emsp;&emsp;别名：`up`

```bash
# upgrade all upgradeable packages
sudo dnf [options] upgrade

sudo dnf [options] upgrade <package-spec>...

# upgrade module
sudo dnf [options] upgrade @<spec>...
```

&emsp;&emsp;其实`update`是一个已经废弃的别名。

## 搜索

&emsp;&emsp;命令：`search`

&emsp;&emsp;别名：`se`

```bash
dnf [options] search [--all] <keywords>...
```

&emsp;&emsp;搜索结果以列表形式展示，要查看包的详细信息使用下面的命令。

&emsp;&emsp;命令：`info`

&emsp;&emsp;别名：`if`

```bash
dnf [options] info [<package-file-spec>...]
```

## 历史

&emsp;&emsp;命令：`history`

&emsp;&emsp;别名：`hist`

```bash
dnf history [list] [--reverse] [<spec>...]
```

## 清理

&emsp;&emsp;命令：`remove`

&emsp;&emsp;别名：`rm`

```bash
# will take a long time
sudo dnf rm --duplicates

# can clean old kernels
sudo dnf rm --oldinstallonly
```

&emsp;&emsp;一般情况下，remove 一个特定软件包时会自动删除相关无用依赖。oldinstallonly 选项可以用来删除旧内核镜像。

## 更新系统

1. 更新发行版

&emsp;&emsp;命令：`upgrade --refresh`

```bash
sudo dnf upgrade --refresh
```

&emsp;&emsp;Docs 说运行后要 reboot，不确定是否需要。

2. 下载更新插件

&emsp;&emsp;命令：`install dnf-plugin-system-upgrade`

```bash
sudo dnf install dnf-plugin-system-upgrade
```

&emsp;&emsp;如果已经安装了，可以跳过。

3. 下载新软件包

&emsp;&emsp;命令：`system-upgrade download --releasever=39`

```bash
sudo dnf system-upgrade download --releasever=39
```

&emsp;&emsp;可以跨版本更新，如从 37 直接到 39。

4. 重启更新

&emsp;&emsp;命令：`system-upgrade reboot`

```bash
sudo dnf system-upgrade reboot
```

5. 清理

&emsp;&emsp;参见清理。