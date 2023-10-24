---
title: Makefile
date: 2023-10-01 19:57:29
tags:
    - Make
    - C
categories:
    - [Build, Make]
---

&emsp;&emsp;学习一下 GNU Make 的语法 Makefile。

<!-- more -->

## 简介

&emsp;&emsp;make 是一个命令工具，它解释 Makefile 中的指令，大多数 IDE 都有这个命令，包括 VC++ 的 nmake，Linux 下的 GNU make。Makefile 关系到整个工程的编译规则，它定义了一系列规则来指定，哪些文件需要先编译，哪些文件需要后编译，哪些文件需要重新编译。

&emsp;&emsp;make 命令执行时需要一个 Makefile 文件，以告诉 make 命令怎样编译和链接程序。

&emsp;&emsp;Makefile 的规则很简单：

```makefile
target ... : prerequisites ...
	recipe
	...
	...
```

-   target

&emsp;&emsp;target 可以是一个 object file，也可以是可执行文件，还可以是一个标签。

-   prerequisites

&emsp;&emsp;prerequisites 是生成该 target 所依赖的文件或者目标。

-   recipe

&emsp;&emsp;recipe 是该 target 要执行的命令。

&emsp;&emsp;这是一个文件的依赖关系：target 这一个或多个目标依赖于 prerequistes 中的文件或目标，其生成规则定义在 command 中。如果 prerequistes 中有一个以上的文件比 target 文件要新，recipe 所定义的命令就会被执行。

## 流程

&emsp;&emsp;一个 Makefile 的示例：

```makefile
edit : main.o kbd.o command.o display.o \
		insert.o search.o files.o utils.o
	cc -o edit main.o kbd.o command.o display.o \
		insert.o search.o files.o utils.o
		
main.o : main.c defs.h
	cc -c main.c

kbd.o : kbd.c defs.h command.h
	cc -c kbd.c

command.o : command.c defs.h command.h
	cc -c command.c

display.o : display.c defs.h buffer.h
	cc -c display.c

insert.o : insert.c defs.h buffer.h
	cc -c insert.c

search.o : search.c defs.h buffer.h
	cc -c search.c

files.o : files.c defs.h buffer.h command.h
	cc -c files.c

utils.o : utils.c defs.h
	cc -c utils.c

clean :
	rm edit main.o kbd.o command.o display.o \
		insert.o search.o files.o utils.o
```

&emsp;&emsp;在默认的方式下，只输入 make 命令，那么

1.   make 会在当前目录下寻找名为 Makefile 或 Makefile 的文件。
2.   如果找到，它会把文件中第一个目标 edit 作为最终目标。
3.   如果最终目标 edit 不存在，或是 edit 所依赖的文件修改时间比 edit 要新，那么它就会执行后面的命令来生成 edit。
4.   如果 edit 所依赖的文件 (object file) 也不存在，那么 make 就会在当前文件中寻找目标为该文件的依赖性，然后则根据该规则来生成缺失的依赖文件。
5.   因为 C 文件和头文件总是存在的，所以 make 会先生成中间文件，然后用中间文件生成 edit 文件。

&emsp;&emsp;像 clena 这种没有被第一个目标直接或间接关联的目标，它后面所指定的命令将不会被自动执行。但是，可以通过显示的要求，让 make 生成某个目标：make clean。

## 内容

&emsp;&emsp;&emsp;&emsp;Makefile 中主要包含 5 个东西：显式规则，隐式规则，变量定义，指令和注释。

-   显式规则

&emsp;&emsp;显式规则说明如何生成一个或多个目标文件，Makefile 书写者应明显指出要生成的文件，文件的依赖文件以及生成命令。

-   隐式规则

&emsp;&emsp;由于 GNU make 有自动推导功能，所以隐式规则允许简略的书写 Makefile。

-   变量定义

&emsp;&emsp;可以在 Makefile 中定义一系列变量，变量一般都是字符串。当 Makefile 被执行时，其中的变量会被扩展到相应的引用位置上。这十分类似 C 中的宏定义和预处理。

-   指令

&emsp;&emsp;指令包含 3 部分：其一是在一个 Makefile 中引用另一个 Makefile，像 C 中的 include。另一个是根据条件指定 Makefile 中的有效部分，像 C 中的 #if。还有一个是定义一个多行的命令。

-   注释

&emsp;&emsp;Makefile 中只有行注释，和 UNIX 中的 shell 脚本一样使用 # 字符。如果要使用该字符，应该转义`\#`。

## 书写规则

&emsp;&emsp;规则包含两个部分：依赖关系和生成目标的方法。

&emsp;&emsp;Makefile 中，规则顺序很重要，因为 Makefile 中只应该有一个最终目标，其它目标都被这个目标连带出来。所以一定要让 make 知道最终目标是什么。一般来说，定义在 Makefile 中的目标可能会有很多，但是第一条规则中的目标将被确立为最终的目标。如果第一条规则中的目标有很多个，那么，第一个目标会成为最终的目标，make 所完成的也就是这个目标。

### 规则语法

```makefile
targets : prerequisites
	command
	...
```

&emsp;&emsp;或是这样：

```makefile
targets : prerequisites ; command
	command
	...
```

&emsp;&emsp;targets 是文件名，以空格分开，可以使用通配符。

&emsp;&emsp;command 是命令行命令，如果不与 targets 在同一行，必须以 Tab 键开头。如果要与 targets 在同一行，那么以符号 ; 分隔，即第二种写法。

&emsp;&emsp;prerequistes 是目标所依赖的文件或目标，如果其中某个依赖文件比目标文件要新 (修改时间)，那么目标就是`过时`的，被确认为要被重新生成。

&emsp;&emsp;生成一个目标的命令可以有多个，即可以列多行 command。如果命令太长，可以使用 \ 作为换行符。
