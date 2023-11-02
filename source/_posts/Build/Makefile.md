---
title: Makefile
date: 2023-10-01 19:57:29
updated: 2023-11-02 21:00:00
tags:
  - Make
  - C
categories:
  - [Build, Make]
---

&emsp;&emsp;学习一下 GNU Make。

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

- target

&emsp;&emsp;target 可以是一个 object file，也可以是可执行文件，还可以是一个标签。

- prerequisites

&emsp;&emsp;prerequisites 是生成该 target 所依赖的文件或者目标。

- recipe

&emsp;&emsp;recipe 是该 target 要执行的命令。

&emsp;&emsp;规则说明了一些文件之间的依赖关系：target 这一个或多个目标依赖于 prerequistes 中的文件或目标，command 即生成目标所需的指令。如果 prerequistes 中有一个以上的文件比 target 文件要新或 target 不存在，则 recipe 所定义的命令就会被执行。

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

&emsp;&emsp;在默认的方式下，只输入 make 命令，那么：

1. make 会在当前目录下寻找名为 Makefile 或 makefile 的文件。如果找到，它会把文件中第一个目标 edit 作为最终目标。
3. 如果最终目标 edit 不存在，或是 edit 所依赖的文件修改时间比 edit 要新，那么它就会执行后面的命令来生成 edit。
4. 如果 edit 所依赖的文件 (object file) 也不存在，那么 make 就会在当前文件中寻找目标为该文件的规则，然后则根据该规则来生成缺失的依赖文件 (这是一个递归过程)。
5. 因为 C 文件和头文件总是存在的，所以 make 会先生成中间文件，然后用中间文件生成 edit 文件。

&emsp;&emsp;像 clean 这种没有被第一个目标直接或间接关联的目标，它后面所指定的命令将不会被自动执行。但是，可以通过显式的要求让 make 生成某个目标：make clean。

## 内容

&emsp;&emsp;Makefile 中主要包含 5 部分：显式规则，隐式规则，变量定义，指令和注释。

- 显式规则

&emsp;&emsp;显式规则说明如何生成一个或多个目标文件，Makefile 书写者应明显指出要生成的文件，文件的依赖文件以及生成命令。

- 隐式规则

&emsp;&emsp;由于 GNU make 有自动推导功能，所以隐式规则允许书写简略的 Makefile。

- 变量定义

&emsp;&emsp;可以在 Makefile 中定义一系列变量，变量一般都是字符串。当 Makefile 被执行时，其中的变量会被扩展到相应的引用位置上。这十分类似 C 中的宏定义和预处理。

- 指令

&emsp;&emsp;指令包含 3 部分：其一是在一个 Makefile 中引用另一个 Makefile，像 C 中的 include。另一个是根据条件指定 Makefile 中的有效部分，像 C 中的 #if。最后是可以定义一个多行的命令。

- 注释

&emsp;&emsp;Makefile 中只有行注释，使用 # 字符。如果要使用该字符，应该转义`\#`。

## 书写规则

&emsp;&emsp;规则包含两个部分：依赖关系和生成目标的方法。

&emsp;&emsp;在 Makefile 中，规则的定义顺序很重要。因为 Makefile 中只应该有一个最终目标，其它目标都是生成最终目标的中间过程。所以，一定要让 make 知道最终目标是什么。

&emsp;&emsp;一般来说，定义在 Makefile 中的目标可能会有很多，但第一条规则中的目标将被确立为最终的目标。如果第一条规则中的目标有很多个，那么，第一个目标会成为最终的目标，make 所完成的也就是这个目标。

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

&emsp;&emsp;targets 是文件名，以空格分开。command 是命令行命令，如果不与 targets 在同一行，必须以 Tab 键开头。如果要与 targets 在同一行，那么以符号 ; 分隔。

&emsp;&emsp;prerequistes 是目标所依赖的文件或目标，如果其中某个依赖文件比目标文件要新 (修改时间)，那么目标就是`过时`的，被确认为要被重新生成。

&emsp;&emsp;生成目标的命令可以有多个：可以有多行 command。如果命令太长，可以使用 \ 换行。

### 通配符

&emsp;&emsp;在 Makefile 中可以使用通配符：\~，\*，?，它们的含义和在 sh 中一致。

&emsp;&emsp;规则中的 target 和 prerequisties 可以包含通配符：

```makefile
app: *.c
	gcc -o app $^
```

### 伪目标

&emsp;&emsp;如果一个目标不关联实际文件，则该目标就是伪目标。

```makefile
.PHONY: clean
clean:
	rm -f *.o
```

&emsp;&emsp;目标 clean 并不关联一个文件，所以它是一个伪目标。注意：伪目标的取名不能和文件重名，否则就变成了一个正常的文件间依赖规则。可以使用`.PHONY`向 make 指明哪些目标是伪目标，一旦指明某个目标是伪目标，那么 make 将忽略可能存在重名文件。

&emsp;&emsp;伪目标可以有依赖文件，也可以被其他目标依赖。

### 多目标

&emsp;&emsp;GNU make 支持多目标，这能省略很多重复工作：

```makefile
app1 app2: common.h
```

&emsp;&emsp;现在两个目标 app1 和 app2 都依赖文件 common.h，其等价于：

```makefile
app1: common.h
app2: common.h
```

&emsp;&emsp;如果规则中包含多目标和命令，则命令的书写有些不同：此时要在一条规则中书写生成每个目标的通用命令，所以要使用一些`自动化变量`：

```makefile
all: a b

a b: common.c 
	gcc -o $@ $(addsuffix .c,$@) common.c
```

&emsp;&emsp;这里的 $@ 代表目标集合中的目标，类似使用 foreach 展开时的中间变量。

### 静态模式

&emsp;&emsp;静态模式是多目标的强化，其语法是：

```makefile
<targets ...> : <target-pattern> : <prereq-patterns ...>
	<commands>
	...
```

&emsp;&emsp;其中 \<target-pattern> 描述目标集合满足的公共特征，可以使用 % 通配符。\<prereq-patters> 是对目标集合的重定义，它基于目标集合描述了依赖集合的特征，也可以使用 % 通配符。

```makefile
objects = foo.o bar.o

all: $(objects)

$(objects): %.o: %.c
	$(CC) -c $(CFLAGS) $< -o $@
```

&emsp;&emsp;除了自动化变量 \$@，这里使用的 \$< 也是一个自动化变量，它代表依赖集合中的第一个文件。上面的静态模式展开后效果类似：

```makefile
foo.o: foo.c
	$(CC) -c $(CFLAGS) foo.c -o foo.o
bar.o: bar.c
	$(CC) -c $(CFLAGS) bar.c -o bar.o
```

&emsp;&emsp;静态模式是基于模式匹配的，所以这里的 foo 只依赖 foo.c 而不包含 bar.c。此外，依赖模式可以有多个，但是 $< 只能取第一个依赖文件。

## 书写命令

&emsp;&emsp;规则中的每条命令都和 shell 中的命令一致。make 会按顺序一条条的执行命令，除非命令紧跟着依赖规则后的分号，否则每条命令必须以 Tab 键开头。在命令之间的空格或空行会被忽略，但如果该空格或空行是以Tab键开头的，那么make会认为其是一个空命令。

### 显示命令

&emsp;&emsp;通常，make 会在将要执行命令前输出要执行的命令行。如果希望 make 不输出该命令行，可以在该命令行前加上字符 `@`：

```makefile
all:
	@echo nothing
```

&emsp;&emsp;此外，在运行 make 时可以使用 -n 选项。这意味着只输出将被执行的命令，而不实际执行：

```bash
make -n
```

### 命令执行

&emsp;&emsp;默认情况下，make 会为每一条命令创建独立的环境。也就是说，后一条命令只是在时间上比前一条命令后执行，而不是在前一条命令的基础上被执行的：

```makefile
all:
	cd sub
	pwd
```

&emsp;&emsp;pwd 命令并不会显示当前位于 sub，这是因为它们之间是被隔离的。

&emsp;&emsp;如果希望命令在同一环境中被执行，可以将它们放在一行，并以分号隔开：

```makefile
all:
	cd sub; pwd

# or

all:
	cd sub; \
	pwd
```

### 命令出错

&emsp;&emsp;每执行完一条命令，make 会检测 shell 的终止状态。如果异常终止，make 就会结束工作。

&emsp;&emsp;如果希望忽略错误继续执行，则可以在命令行前加上字符`-`：

```makefile
clean:
	-rm -f *.o
```

### 嵌套 make

&emsp;&emsp;一个复杂的工程可能包含很多子模块，它们分别被独立管理和编译。在一个 Makefile 中编写所有的规则会增加维护难度，所以可以进行适当的拆分。

&emsp;&emsp;根目录下的 Makefile 可当作总控文件，它会管理每个子 Makefile。当需要编译时，总控文件会调用 make 执行每个子 Makefile：

```makefile
all:
	cd sub && make
	...
```

&emsp;&emsp;上面的写法可以简化为：

```makefile
all:
	$(MAKE) -C sub
```

&emsp;&emsp;当父 Makefile 嵌套执行子 Makefile 时，父文件中的变量默认会被传递给子文件，但是不会覆盖子 Makefile 中已存在的变量，子文件可以重定义这些变量。

&emsp;&emsp;可以显式的要求传递或不传递某个变量到子文件中：

```makefile
export
export var1
export var2  = true
export var3 := true

unexport var1 var2 var3
```

&emsp;&emsp;一个空的 export 表示传递所有变量。

### 命令包

&emsp;&emsp;命令包效果类似将命令放到变量中，适合复用命令：

```makefile
define cmds
cd sub; \
pwd
endef

all:
	$(cmds)
```

&emsp;&emsp;使用 define 和 endef 关键字包裹命令，并在 define 后跟上命令包的名字。使用命令包就和使用变量一样，它的效果类似直接将变量展开。
