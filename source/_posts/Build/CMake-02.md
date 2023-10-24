---
title: CMake 02 - 语法基础
date: 2023-06-25 09:57:21
tags:
    - CMake
    - CXX
categories:
    - [Build, CMake]
---

&emsp;&emsp;CMake 使用自己的脚本语言 CMake 来控制整个构建流程。

<!-- more -->

## CMake 语言

&emsp;&emsp;可以单独运行 CMake 脚本文件，虽然脱离了项目构建是毫无意义的，但是方便学习 CMake 语言的语法。

```bash
cmake -P file.cmake
```

## 注释

&emsp;&emsp;CMake 支持单行注释和多行注释，第五行中的`#`可选，但是为了对称美观建议加上：

```cmake
# single-line comment

#[[
	multiple-line comment
#]]
```

&emsp;&emsp;在多行注释中，`[[`和`]]`之间可以插入任意相等数量的`=`，并且多行注释支持嵌套：

```
#[==[
    multi-line comment

    # single-line comment
    
    #[[
        nested comment
    #]]
#]==]
```

## 指令

&emsp;&emsp;CMake 内建了一些指令，并且之前已经用到，如：`add_executable`，`target_sources`。只用注意以下几点：

1. 调用方式：command(arg1 arg2)
2. 参数以一个或多个空格分隔，且指令名不区分大小写。
3. 传递参数的方式有多种：

```cmake
# 1
message(hello world)
# output: helloworld

# 2
message(hello " world")
# output: hello world

# 3
message([[hello world]])
# output: hello world
```

&emsp;&emsp;message 指令用于将传给它的参数打印到控制台上。

&emsp;&emsp;其中第三种使用`[[]]`语法，方括号中的所有内容将原样传递给 message 指令，并且可以在`[[`和`]]`之间插入任意数量相等的`=`，类似多行注释。

## 变量

&emsp;&emsp;CMake 内有三种变量：普通变量、缓存变量和环境变量。在内部，变量一般作为字符串储存，也包含一些特殊变量如：列表等等。

### 基本操作

1. 使用`set`创建变量。
2. 使用`unset`销毁变量。

```cmake
set(name "Arthur")
set(age  19)

unset(name)
unset(age)
```

&emsp;&emsp;变量名可以使用`[[]]`语法和`""`语法，也就是说变量名可以包含空白字符 (目前已经被废除)，但是我个人认为这种写法比较反人类：

```cmake
set([[var 1]] "text 1")
set("var 2" "text 2")

unset([[var 1]])
unset("var 2")
```

### 引用变量

&emsp;&emsp;引用变量的方式比较灵活，下面是几条核心规则：

1. 使用`${varName}`语法引用变量的值，过程可称作求值或展开。
2. 求值时 CMake 将遍历当前作用域符号表，并将`${varName}`替换为对应变量的值，若没有找到变量，则替换为空字符串 (CMake 不会产生错误)。

3. 变量展开时顺序由内而外：

```cmake
set(var_name "text")
set(outer "var")
set(inner "name")

message(${${outer}_${inner}})
```

&emsp;&emsp;先展开 outer 和 inner，经过拼接得到`var_name`，然后外部展开 var_name。

### 普通变量

&emsp;&emsp;普通变量的定义和引用已经研究过了，下面研究下普通变量的作用域和覆盖问题。`function`表示定义一个函数，函数有自己的作用域。

```cmake 
function(outer)
    set(var "outer" PARENT_SCOPE)
    # unset(var2)
    message("in outer,  var : ${var}")
    message("in outer,  var2: ${var2}")
endfunction()

function(inner)
    set(var "inner")
    set(var "inner2" PARENT_SCOPE)
    outer()
    message("in inner,  var : ${var}")
endfunction()

set(var "global")
set(var2 "value")

inner()
message("in global, var : ${var}")
```

&emsp;&emsp;以下内容都是我个人理解，仅供参考：

1. 在 CMake 中，普通变量的作用域有两种：当前作用域和父作用域。
2. 在执行函数时，会将当前作用域的所有普通变量拷贝一份到目标函数的作用域。类似的在包含目录时 (调用指令 `add_subdirectory`) 也会创建子作用域。
3. 在引用普通变量时，只会从当前作用域寻找，如果当前作用域不存在说明父作用域也不存在，因为当前作用域的符号集是父作用域的超集。
4. 默认修改变量总是针对当前作用域，`PARENT_SCOPE`也只能修改上一级的父作用域变量。

&emsp;&emsp;分析以上程序的结果：

![作用域分析](01.png)

### 环境变量

&emsp;&emsp;CMake 在运行时也会将系统环境变量拷贝一份到当前运行环境中。然后，在 CMake 中对环境变量的操作都基于此副本进行。也就是说：读取的值只保证是刚才拷贝的值，修改也只是修改副本而不会影响系统。

1. 访问环境变量：使用`$ENV{varName}`语法引用环境变量的值。
2. 设置或修改环境变量：`set(ENV{varName} value)`来操作环境变量。

```cmake
function(func)
    set(ENV{name} "fanenr")
    message("name: $ENV{name}")
endfunction()

set(ENV{name} "arthur")
message("name: $ENV{name}")
func()
message("name: $ENV{CMAKE_TOOLCHAIN_FILE}")
```

&emsp;&emsp;因为环境变量表只储存一份，所以没有作用域限制。在任何地方修改环境变量都将导致其后获取的是修改后的值。

### 缓存变量

&emsp;&emsp;缓存变量只存在于构建项目时。所有的缓存变量都存放在构建目录下的`CMakeCache.txt`文件中。所以它也是一份，并且没有作用域限制。注意：创建缓存变量将删除同名的普通变量。

1. 设置缓存变量：`set(varName value CACHE STRING description)`，其中`CACHE`表示这是一个缓存变量，`STRING`是变量类型，支持的类型有：STRING, BOOL, FILEPATH, INTERNAL。设置类型是为了给 GUI 提供信息，命令行环境不用管，全部看作字符串即可。
2. 引用缓存变量：`$CACHE{varName}`或者`${varName}`。因为缓存变量将删除同名的普通变量，所以能保证按普通变量的引用方式引用到的也是唯一的值。

3. 修改缓存变量：`set(varName value CACHE STRING description FORCE)`。这会导致最终写入`CMakeCache.txt`文件中的变量值和描述改变，并且随后访问到的值也是新值。

4. INTERNAL, STRING 都表示字符串。但是前者自带 FORCE 属性。

```cmake
# ./CMakeLists.txt
cmake_minimum_required(VERSION 3.21)
project(02_2)

set(cvar "a" CACHE STRING "a cache variable")
# set(cvar "a" CACHE INTERNAL "a cache variable")
add_subdirectory(sub)

message("in top, cvar: ${cvar}")
```

```cmake
# ./sub/CMakeLists.txt
set(cvar "b" CACHE STRING "a cache variable modified" FORCE)
# set(cvar "b" CACHE INTERNAL "a cache variable modified")
message("in sub, cvar: ${cvar}")
```

## 列表

&emsp;&emsp;CMake 支持的复杂类型不多，但是对于构建系统来说也不需要多复杂的数据结构。CMake 的列表使用比较简单，并且支持的操作也很多。

1. 创建列表有两种语法：

```cmake
set(ml "a;b;c;d")
# or
set(ml "a" "b" "c" "d")

list(GET ml 2 elem)
message(${elem})
```

2. 支持的操作：
    1. 直接打印：`message(list_name)`可以直接打印列表。
    2. 追加元素：`list(APPEND list_name elem...)`支持一个或多个元素。
    3. 访问元素：`list(GET list_name index out_var)`通过索引访问元素并导出到变量中。
    4. 其他操作：list 还有一些其他操作譬如查找元素，通过索引访问，排序等 (个人觉得不常用)。具体参考官方手册和其他教程。
