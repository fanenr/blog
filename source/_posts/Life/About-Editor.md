---
title: 关于编辑器
date: 2023-11-02 16:00:00
updated: 2023-11-02 19:00:00
tags:
    - Life
categories:
    - Life
---

&emsp;&emsp;写 Blog 已经有几个月了，关于编辑器的使用我也有了一些体会。

&emsp;&emsp;我最先接触的是语雀的编辑器，应该跟市面上大部分的编辑器类似：都是类 Markdown 的富文本 Web Editor 包装。然后是 Typora，这也是 Electorn 打包的应用。最后是现在使用的 VS Code。

&emsp;&emsp;写完才后知后觉，原来都是 Electorn 应用...

<!-- more -->

## Typora

1. 优点

&emsp;&emsp;Typora 是付费应用，只有一个作者在维护。对于一般用户来说，其功能肯定是够用了。

&emsp;&emsp;Typora 出色的地方是它的即时渲染功能，无论性能怎么样，这种所见即所得的体验还是让人很省心。此外，我还非常喜欢 Typora 的可视化表格组件，插入和删除表项都很方便。

2. 问题

&emsp;&emsp;因为闭源和作者问题，Typora 可扩展性很差，没有插件系统和自定义函数等功能。虽然基础功能足够，但对于非标准单文件的 Markdown 环境来说还是很难应对。

&emsp;&emsp;就写 Blog 来说，Typora 和 Hexo 是完全不对付。为了解决图片路径问题，Hexo 社区出现了一大堆适配 Typora 的插件 (虽然我没试过)。但其实我最在乎的是扩展性和性能问题，当编写 10k 左右文字时，Typora 就会出现卡顿现象 (不明显但可感)。因为文章里常常涉及很多代码和列表。所以我经常查看文章的 Markdown 源码，来回切换之间能也感觉到明显的卡顿。

&emsp;&emsp;最终劝退我的是扩展性问题：没法多文件搜索，文件树操作不方便，没法添加钩子，不支持插件...

---

&emsp;&emsp;最后，虽然我不以 Typora 为主力了，但我仍认为 Typora 是一款很好的 Markdown 编辑器。

## VS Code

&emsp;&emsp;VS Code 其实是一个代码编辑器...

&emsp;&emsp;虽然我一直用 VSC 来写代码，但现在我也把它当作主力 Markdown 编辑器。主要原因是：VSC 的高度可定制性，以及优异的性能。

1. 优点

&emsp;&emsp;首先自然是 VSC 强大的可扩展性了：各种插件和主题。加上我时刻要用终端，内置的 panel 也很舒服。此外，VSC 内置的 Git 版本控制系统 (虽然我不用) 也很不错。

&emsp;&emsp;VSC 能成为当前最主流的编辑器，一个重要因素就是它的性能。虽然把 VSC 这样的巨型项目和 Typora 来对比很不合适，但是：VSC 的编辑和渲染性能数倍于 Typora (二者都是 Electorn)。

2. 问题

&emsp;&emsp;VSC 主要问题还是：它是一个代码编辑器。虽然有各种插件，包括高仿 Typora 的插件。VSC 仍然难以匹敌 Typora 所见即所得的效果。

&emsp;&emsp;此外，VSC 作为一个 Code Editor，只有在编辑源码时才能体现其威力。所以使用 VSC 要求用户了解一定的 Markdown 标准和规范。最痛苦的当然是：打的字更多了。

---

&emsp;&emsp;虽然 Typora 很让人省心，但 VSC 这种让人掌控一切的感觉更适合我。
