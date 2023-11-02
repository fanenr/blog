---
title: CMake 01 - 概念
date: 2023-06-24 18:55:57
updated: 2023-06-25 19:00:00
tags:
  - CMake
  - CXX
categories:
  - [Build, CMake]
---

&emsp;&emsp;简单记录一下学习 CMake 工具的过程。

<!-- more -->

## CMake 概览

&emsp;&emsp;CMake 不仅是构建系统 — 还是一个构建系统生成器，可以为其他构建系统 (Makefile, Ninja...) 生成相应的工程。CMake 不仅限于构建软件 — 还支持安装、打包和测试。

&emsp;&emsp;CMake 主要由以下三个工具组成：

1. cmake: CMake 本身，用于生成构建系统。

2. ctest: CMake 的测试程序，用于检测和运行测试。

3. cpack: CMake 的打包程序，用于将程序打包成安装程序。

## 简单示例

&emsp;&emsp;一个最简单 CMake 工程结构大概是这样，其中`build`目录存放构建文件，这些构建文件由 CMake 生成。`src`存放的是代码源文件。`CMakeLists.txt`文件是使用 CMake 语言编写的，控制 CMake 构建行为的配置文件。

```
.
├── build
├── CMakeLists.txt
└── src
    └── main.cpp
```

&emsp;&emsp;`main.cpp`文件内容就是初学 C++ 时写的在控制台上打印`Hello World`，着重分析一下`CMakeLists.txt`文件的内容：

``` cmake
# ./CMakeLists.txt
cmake_minimum_required(VERSION 3.21)
project(
    "cmake-example-01"
    VERSION 0.0.1
    DESCRIPTION "just a example"
    LANGUAGES CXX
)

add_executable(01)
target_sources(01 PRIVATE src/main.cpp)
```

1. `cmake_minimum_required(VERSION 3.21)`定义了项目所需 CMake 的最低版本。顶级`CMakeLists.txt`文件必须以这个命令开始。如果项目使用了 CMake 的某个特性，那么此值应为支持该特性的最低 CMake 版本。
2. `project(...)`定义的内容是本项目的属性，他们分别是：项目名称，项目版本，项目描述，项目语言。
3. `add_executable(01)`告诉 CMake 要构建一个可执行文件，并且文件名是`01``
4. `target_sources(...)`告诉`01`可执行文件的源代码文件有哪些，这里只有一个`src/main.cpp`，其中`PRIVATE`限制这些源代码文件的可见性仅限于`01`目标文件。

## 构建项目

&emsp;&emsp;确定好了一个 CMake 工程的基本结构，接着就是使用 CMake 来生成构建文件，和开始构建目标程序。

``` bash
cmake -S . -B build
cmake --build build
./build/01
```

1. 第一行的目的是：使用 CMake 生成构建文件如 Makefile。其中`-S`选项告诉 CMake 源文件的位置 (. 即当前目录下)，注意：顶级`CMakeLists.txt`文件必须包含在此目录下，事实上可以将真正的源代码文件和`CMakeLists.txt`分离，但是一般不这么做。`-B`告诉 CMake 生成将生成的构建文件存放在哪个目录。
2. 第二行的目的是：让 CMake 调用相应的构建系统来生成目标文件和可执行程序。如果是 Makefile 构建系统，那么 CMake 将会调用 make (make 调用 g++)，来构建程序。`--build`告诉 CMake 构建文件在哪里，即上一步骤中指明的位置。
3. 第三行即运行程序。

&emsp;&emsp;上面的写法是官方推荐的，完整的写法，其实还可以使用更简略的写法。

``` bash
cd build
cmake ..
make
```

&emsp;&emsp;以上操作都在构建文件目录中进行，第一行`cmake ..`只指明了源文件的位置 (.. 表示上一级目录)，那么隐式的 CMake 将当前目录 (build) 当作构建文件的存放目录。第二行同理，表明构建文件就位于当前目录。

## 构建过程

&emsp;&emsp;关于 CMake 的构建过程，有的资料说它有两个，有的说有三个。但是过程的描述基本是相似的，也不必纠结。

![构建过程](01.png)

1. 配置阶段：收集基本信息如：架构，工具链，编译器，链接器等等。
2. 生成阶段：解析 CMakeLists.txt 文件，生成相应构建系统的构建文件。
3. 构建阶段：调用相应构建系统来构建程序。

&emsp;&emsp;如果项目结构是这样，其中`add_subdirectory`表示包含一个子目录：

```
.
├── build
├── CMakeLists.txt
└── src
    ├── CMakeLists.txt
    ├── main.cpp
    └── sub_src
        └── CMakeLists.txt
```

``` cmake
# ./CMakeLists.txt
cmake_minimum_required(VERSION 3.21)
project(
    "cmake-example-01_2"
    VERSION 0.0.1
    DESCRIPTION "just a example"
    LANGUAGES CXX
)

# add_subdirectory(src)
# add_subdirectory(src/sub_src)
```

``` cmake
# ./src/CMakeLists.txt
add_subdirectory(sub_src)
```

``` cmake
# ./src/sub_src/CMakeLists.txt
add_executable(01)
target_sources(01 PRIVATE ../main.cpp)
```

1. 构建出的目标文件的位置：和源文件中包含`add_executable`命令的`CMakeLists.txt`的位置是对应的。对于上面的例子来说：01 可执行文件是由`./src/sub_src/CMakeLists.txt`配置的，所以 01 将总是生成在`build/src/sub_src/01`。
2. 对于保存构建信息的子目录，即`CMakeFiles`目录：并不是所有`CMakeLists.txt`都会对应一个子目录，只有被执行的`CMakeLists.txt`文件才会生成子目录。并且子目录的位置规则也遵从由源目录到构建目录的映射关系。

&emsp;&emsp;如果我启用顶层`CMakeLists.txt`的第 10 行，那么 src 目录将被包含，`./src/CMakeLists.txt`将被执行，由于`./src/CMakeLists.txt`中再次包含了子目录 sub_src，`./src/sub_src/CMakeLists.txt`也会被执行。那么 build 目录的结构将是这样：

```
build
├── CMakeFiles -> ./CMakeLists.txt
└── src
    ├── CMakeFiles -> ./src/CMakeLists.txt
    └── sub_src
        └── CMakeFiles -> ./src/sub_src/CMakeLists.txt
```

&emsp;&emsp;如果我只启用顶层`CMakeLists.txt`的第 11 行，跳过 src 目录直接包含子目录 sub_src，那么 build 目录的结构将是这样，区别仅在于`./src/CMakeLists.txt`没有被执行，所以它没有对应的`CMakeFiles`目录。

```
build
├── CMakeFiles -> ./CMakeLists.txt
└── src
    └── sub_src
        └── CMakeFiles -> ./src/sub_src/CMakeLists.txt
```
