---
title: CMake 04 - 构建工程
date: 2023-06-27 10:47:57
updated: 2023-06-28 19:00:00
tags:
  - CMake
  - CXX
categories:
  - [Build, CMake]
---

&emsp;&emsp;正式开始使用 CMake 来构建 C/C++ 工程。

<!-- more -->

## 项目结构

&emsp;&emsp;保持项目结构清晰十分有利于后续维护和迁移。可能很多工程组织项目的方式略有不同，但是他们至少应遵守某一原则而不随意改变。

&emsp;&emsp;下面是一个使用 CMake 构建的 C/C++ 工程目录结构：

``` 
.
├── bin
├── build
├── CMakeLists.txt
├── doc
├── include
├── libd
│   ├── CMakeLists.txt
│   ├── include
│   └── src
├── libh
│   ├── CMakeLists.txt
│   └── include
├── libs
│   ├── CMakeLists.txt
│   ├── include
│   └── src
├── src
│   ├── CMakeLists.txt
│   ├── main.cpp
└── test
```

1. `build`目录存放 CMake 生成的构建信息。`bin`目录存放二进制文件 (ELF)。
2. `test`目录存放项目测试文件。`doc`存放项目文档。
3. `include`目录是当前主项目的头文件目录。`src`是主项目源代码文件目录。
4. `libs`是子项目，这里它是一个静态库。`libh`是头文件库。`libd`是一个动态链接库。

&emsp;&emsp;关于子项目以及外部库，可能有的工程将他们统一放置在类似`extern`或者`libs`的文件夹下，两种方案都是可行的。

&emsp;&emsp;遵循关注点分离原则，一个大型项目被拆分成多个部分进行组织，具体体现即通过不同目录层级中的`CMakeLists.txt`文件来划分。顶级`CMakeLists.txt`文件作为构建入口，它负责引入这些目录和进行一些全局配置，而具体的构建任务编写于子`CMakeLists.txt`文件中。

### 引入目录

&emsp;&emsp;引入其他目录的方式是使用`add_subdirectory(path)`指令，其中 path 可以是相对路径。这条指令将查找该目录下的`CMakeLists.txt`文件并执行。并且，这种方式会产生`子作用域`，事实上这样做更加安全。

```cmake
# ./CMakeLists.txt
cmake_minimum_required(VERSION 3.21)
project(03)

# ----------------------------------------
# subproject libs libd libh
# ----------------------------------------
add_subdirectory(libs)
add_subdirectory(libd)
add_subdirectory(libh)

# ----------------------------------------
# src directory
# ----------------------------------------
add_subdirectory(src)
```

### 全局目标

&emsp;&emsp;上面的配置中，首先引入了 libs libd libh 子目录，最后引用主项目的 src 目录。这么做的原因是：主项目需要使用子项目的目标，由于 CMake 线性执行代码，所以要确保这些目标在引入 src 之前被定义。

1. 目标有三类：
    1. executable：使用指令`add_executable`创建的可执行文件。
    2. library：使用指令`add_library`创建的库文件。
    3. custome target：使用指令`add_custom_target`创建的自定义目标。
2. 目标是全局可见的：如环境变量和缓存变量一般，只要变量被创建，可以在其他任何作用域内访问它。只要目标被创建，那么在任何作用域都能访问它。

&emsp;&emsp;由此，先引入的几个子项目分别创建了全局可见的 target，然后在 src 目录中访问和使用这些 target。

## 环境配置

&emsp;&emsp;CMake 在构建之初会收集一些必要的环境信息如：系统类型，版本，架构类型等等。并且将这些信息保存在内建变量中 (可能是缓存变量或环境变量)。项目可以使用这些信息，针对特定的环境来控制构建细节。

### 判别系统

&emsp;&emsp;方式之一是使用内置变量`CMAKE_SYSTEM_NAME`：

```cmake
if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
	# do something for linux
elseif(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
	# do something for darwin
elseif(CMAKE_SYSTEM_NAME STREQUAL "Windows")
	# do something for win
else()
	# do something else
endif()
```

&emsp;&emsp;或者使用简化变量，其中`WIN32`同时适用于 32 和 64 位 Windows 操作系统，并且当系统为 macOS，Cygwin 时`UNIX`也为真。

```cmake
if (WIN32)
	# do something for Windows
elseif(IOS)
	# do something for IOS
elseif(APPLE)
	# do something for macOS
elseif(ANDROID)
	# do something for Android
elseif(CYGWIN)
	# do something for Cygwin
elseif(UNIX)
	# do something for Linux
else()
	# do something else
endif()
```

&emsp;&emsp;系统版本信息存储在变量`CMAKE_SYSTEM_VERSION`中。

### 系统位数

&emsp;&emsp;要分辨系统是 32 位还是 64 位，只需要使用内置变量`CMAKE_SIZEOF_VOID_P`，该变量存放`sizeof(void*)`的值，在 32 位系统中，该值为 4 (byte)，在 64 位系统中为 8 (byte)。

```cmake
if(CMAKE_SIZEOF_VOID_P EQUAL 8)
	# support 64 bits os	
elseif(CMAKE_SIZEOF_VOID_P EQUAL 4)
	# also support 32 bits os
else()
	# not support
endif()
```

### 架构端序

&emsp;&emsp;一般来说高级语言很少关注架构端序问题，但是在编写某些跨平台代码时可能需要知道端序。Intel/AMD/Arm/Apple 芯片架构使用小端序，少数芯片如 PwerPC/MIPS 使用大端序 (网络字节序也是大端序)。

&emsp;&emsp;针对不同语言，CMake 提供了一个变量集`CMAKE_<LANG>_BYTE_ORDER`来标定端序，其值只有两个`LITTLE_ENDIAN`或者`BIG_ENDIAN`，对于 C++ 具体而言就是：

```cmake
if (CMAKE_CXX_BYTE_ORDER STREQUAL "LITTLE_ENDIAN")
    # little endian
else()
    # big endian
endif()
```

## 工具链配置

&emsp;&emsp;CMake 在构建前，会自动收集工具链信息如：Generator，Compiler，Linker。大部分时间默认选项都是可用的，但并不是最适合的，所以 CMake 也提供了控制工具链的机会。

###  C++ 标准

&emsp;&emsp;在编译编译单元时，编译器提供了`-std`选项来指定使用的 C++ 标准。如果没有在 CMake 中指定标准，那么目标的编译指令也就没有这一项，实际使用的标准也就是编译器的默认值。

&emsp;&emsp;推荐在项目中使用一个 C++ 标准来构建所有目标，要做到这一点，只需要在创建目标前设置`CMAKE_CXX_STANDARD`变量。而后所有生成的目标都将读取此值来配置`-std`选项。一般标准在顶级`CMakeLists.txt`中定义，确保其后所有目标都能读取到，如果子项目中修改它来改变标准，也不会影响父作用域 (除非使用 PARENT_SCOPE)。

```cmake
set(CMAKE_CXX_STANDARD 20)
```

&emsp;&emsp;这只是让编译指令中生成了`-std`选项，但是编译器是否支持该标准是不一定的。开启标准检查可以确保编译器一定支持目标标准。

```cmake
set(CMAKE_CXX_STANDARD_REQUIRED ON)
```

&emsp;&emsp;可选是否启用编译器扩展，对应的如：GNU++标准。

```cmake
set(CMAKE_CXX_EXTENSIONS off)
```

### 更改编译器

&emsp;&emsp;CMake 仅仅在第一次构建时确定编译器，然后将使用的编译器写入缓存变量`CMAKE_C_COMPILER`和`CMAKE_CXX_COMPILER`中供构建目标使用，如果没有自定义，CMake 将使用系统默认编译器 (Linux 下通常是 GCC)。官方强调不要更改这两个变量 (可以更改但是不推荐)。有以下几种办法修改项目使用的编译器：

1. 修改环境变量`CC`和`CXX`：CMake 最先检测这两个环境变量 (使用绝对路径)，来确定 C/C++ 的默认编译器。

```bash
export CC=/usr/bin/clang
export CXX=/usr/bin/clang++
```

2. 在构建之初定义这两个变量：如果不想使用上面确定的默认编译器，可以手动覆盖这两个变量。

```cmake
cmake -B build . -DCMAKE_C_COMPILER=/path/to/cc -DCMAKE_CXX_COMPILER=/path/to/cxx    
```

3. 在运行期修改缓存变量 (不推荐)：很长而且不灵活。

```cmake
set(CMAKE_C_COMPILER "/path/to/cc" CACHE FILEPATH "C Compiler" FORCE)
```

### 更改生成器

&emsp;&emsp;和编译器一样，CMake 在第一次构建时就要确定构建系统。CMake 会自动检测系统可用的构建系统，然后根据系统环境变量`CMAKE_GENERATOR`来确定默认构建系统，如果未指定，那么由 CMake 自动选择 (GNU Linux 上是 Make)。

&emsp;&emsp;推荐使用下面两种方式更改生成器：

1. 修改环境变量：

```bash
export CMAKE_GENERATOR="Ninja"
```

2. 在构建时指定：这会覆盖上面确定的默认选项。

```bash
cmake -B build . -G Ninja
```

&emsp;&emsp;如果不知道系统中有哪些可用的构建系统，可以使用命令`cmake --help`来查询。

## 目标

&emsp;&emsp;这部分内容是 CMake 中比较抽象和混杂的，也是我觉得 CMake 设计最差的一部分。

### 属性

&emsp;&emsp;之前已经说过目标有三类，目标的属性类似 class 中的字段。实际在 CMake 中的表现就是很多键值对。因为官方提供的属性列表十分长，而且大部分很难直接用到，这里只介绍部分和项目强相关的属性和使用方法。

1. 读取目标属性：第一种方法是早期通用的方法，这里推荐使用第二种方法。

```cmake 
# method 1
get_property(var TARGET <target> PROPERTY <name>)

# method 2
get_target_properties(var <target> <name>)
```

2. 设置目标属性：推荐使用第二种方法。

```cmake
# method 1
set_property(TARGET <target> PROPERTY <name> <value>)

# method 2
set_target_properties(<target1> <target2> ...
				PROPERTIES <p1> <v1> <p2> <v2> ...)
```

&emsp;&emsp;实际项目中，可能需要关注目标的以下几个属性：

1. 目标的编译选项 (如 -g)

```cmake
target_complie_option(<target> <PRIVATE | PUBLIC | INTERFACE> <options>)
```

2. 目标的预处理宏 (如 -D var=val)

```cmake
target_compile_definitions(<target> <PRIVATE | PUBLIC | INTERFACE> <defs>)
```

3. 目标的头文件目录 (如 -I dir)

```cmake
target_include_directories(<target> <PRIVATE | PUBLIC | INTERFACE> <dirs>)
```

4. 目标的链接库 (如 -l -L)

```cmake
target_link_libraries(<target> <PRIVATE | PUBLIC | INTERFACE> <libs>)
```

5. 目标的链接选项 (如 -fuse-ld)

```cmake
target_link_options(<target> <PRIVATE | PUBLIC | INTERFACE> <options>)
```

&emsp;&emsp;这里没有说目标的源代码列表属性，目标的源代码使用指令`target_sources`设置，之所以不说这个，是因为目标的源代码列表属性一般不涉及属性传播。

### 属性传播

&emsp;&emsp;属性的传播很好理解，就是当一个目标 A 依赖另一个目标 B 时，B 目标的某些属性可能会被传递到 A 上，具体的行为控制，就通过指定上面的关键字`PRIVATE | PUBLIC | INTERFACE`。

1. PRIVATE：此属性只供自己使用。
2. PUBLIC：此属性不仅供自己使用，并且会传递给依赖它的目标。
3. INTERFACE：此属性只供依赖它的目标使用。

&emsp;&emsp;只解释`INTERFACE`：这种传播属性一般用于一些 header-only 的库，因为没有编译单元，也不会产生实际的编译命令。所以就直接将属性传播给依赖其的目标。

### 伪目标和别名

&emsp;&emsp;伪目标实际上就是利用上面`INTERFACE`属性的特点，只需一个例子就能明白：

```cmake
add_library(show_warning INTERFACE)

target_compile_options(show_warning
	INTERFACE
	-Wall -Wextra -Wpedantic
)

target_link_libraries(app show_warning)
```

&emsp;&emsp;目标别名和 C++ 中的引用类似，就是给目标起了一个别名：

```cmake
add_executable(<name> ALIAS <target>)
add_library(<name> ALIAS <target>)
```

### 自定义目标

&emsp;&emsp;自定义目标主要用于完成一些其他任务，如数据校验，中间文件的清理等。在 CMake 中，目标是一个抽象概念，它包含可执行文件，库文件等有实际输出的内容，也可以代表没有输出文件的一组指令。相应的，构建这种目标也仅仅是执行对应的指令而已。

```cmake
add_custom_target(Name [ALL] [command1 [args1...]]
                   [COMMAND command2 [args2...] ...]
                   [DEPENDS depend depend depend ... ]
                   [BYPRODUCTS [files...]]
                   [WORKING_DIRECTORY dir]
                   [COMMENT comment]
                   [JOB_POOL job_pool]
                   [VERBATIM] [USES_TERMINAL]
                   [COMMAND_EXPAND_LISTS]
                   [SOURCES src1 [src2...]])
```

&emsp;&emsp;添加的自定义目标默认是不会被构建的，除非它被其他要求构建的目标所依赖。但是，通过指定`ALL`参数，可以让其直接被构建。官方提供的参数太多，只解释部分常用的参数：

1. COMMAND：构建目标所执行的指令。
2. DEPENDS：构建目标所依赖的其他目标。
3. VERBATIM：禁止 CMake 对上面指令的参数进行转义。

&emsp;&emsp;生成的构建系统文件中应该包含该目标的信息，如果它没有被其他要求构建的目标依赖，手动构建目标的方法有两种：第二种其实就是直接使用相应的构建系统来构建目标。

1. cmake --build build --target target_name
2. cd build && make target_name

### 自定义命令

&emsp;&emsp;使用自定义指令，可以复用一些重复动作，也能当作钩子在目标构建的过程中插入一些指定动作。

```cmake
 add_custom_command(OUTPUT output1 [output2 ...]
                    COMMAND command1 [ARGS] [args1...]
                    [COMMAND command2 [ARGS] [args2...] ...]
                    [MAIN_DEPENDENCY depend]
                    [DEPENDS [depends...]]
                    [BYPRODUCTS [files...]]
                    [IMPLICIT_DEPENDS <lang1> depend1
                                     [<lang2> depend2] ...]
                    [WORKING_DIRECTORY dir]
                    [COMMENT comment]
                    [DEPFILE depfile]
                    [JOB_POOL job_pool]
                    [VERBATIM] [APPEND] [USES_TERMINAL]
                    [COMMAND_EXPAND_LISTS])
```

&emsp;&emsp;这里也只解释部分参数：

1. OUTPUT：指令输出的文件，注意这里是否真正输出文件取决于后面的指令。
2. COMMAND：指令要执行的动作。
3. VERBATIM：同样是禁止转义参数。

&emsp;&emsp;指令定义好了，触发与否，以及触发的时机取决于它的 OUTPUT 是否被其他要求构建的目标所依赖：`print_invoke`目标定义为`ALL`，所以被要求默认被构建。它依赖一个虚拟文件`print`，构建这个虚拟文件的是自定义目标`print`，所以`echo`指令将会在构建`print_invoke`前被调用。

```cmake
add_custom_command(
	OUTPUT  print
	COMMAND echo "nothing"
	VERBATIM
)

add_custom_target(
	print_invoke ALL
	DEPENDS print
)
```

&emsp;&emsp;自定义指令的另一种常用的场景是作为目标钩子，这种操作允许在目标的构建过程中的一些时机触发指令。

```cmake
add_custom_command(TARGET <target>
					PRE_BUILD | PRE_LINK | POST_BUILD
					COMMAND command1 [ARGS] [args1...]
					[COMMAND command2 [ARGS] [args2...] ...]
					[BYPRODUCTS [files...]]
					[WORKING_DIRECTORY dir]
					[COMMENT comment]
					[VERBATIM] [USES_TERMINAL]
					[COMMAND_EXPAND_LISTS])
```

1. PRE_BUILD：目标开始构建之前。
2. PRE_LINK：目标开始链接之前。
3. POST_BUILD：目标构建完成之后。

```cmake
add_executable(app main.cpp)

add_custom_command(
    TARGET app
    PRE_BUILD
    COMMAND echo "start building"
    PRE_LINK
    COMMAND echo "start linking"
    POST_BUILD
    COMMAND echo "finish the build"
    VERBATIM
)
```

## 编译细节

### 预处理配置

&emsp;&emsp;CMake 可以根据模板来生成一个存储配置信息的头文件，头文件中主要包含一些宏定义。核心指令就是一个`configure_file(input output)`下面是使用流程：

1. 定义一个头文件模板：

```c++
// file: ./config.h.in

#cmakedefine DEBUG
#cmakedefine01 IS_LTO
#cmakedefine AUTHOR "${AUTHOR}"
```

&emsp;&emsp;可以看见这里使用的是`#cmakedefine`而不是`#define`，这主要是提供给 CMake 一个配置模板。CMake 在生成头文件时，会根据 CMake 中现有的变量，来替换模板中的内容。

&emsp;&emsp;`DEBUG`宏没有值，如果此时 CMake 中执行`if (DEBUG)`为真的话，`DEBUG`宏就会被保存下来。否则这一行就会被注释掉。`#cmakedefine01`也是根据 if 指令来生成宏，只不过此时宏会有值，且只能是 0 或 1。最后一个宏`AUTHOR`定义了值，并且尝试引用 CMake 中的变量，还是先用 if 检测引用的变量，如果结果为真就会保留该宏并且展开，否则这一行也会被注释掉 (个人感觉不合理)。

2. 使用指令`configure_file`生成头文件：

```cmake
set(DEBUG on)
set(IS_LTO off)
set(AUTHOR "Arthur")

configure_file("config.h.in" "config.h")
```

3. 使用生成的头文件：

```c++
// file: ./build/config.h

#define DEBUG
#define IS_LTO 0
#define AUTHOR "Arthur"
```

&emsp;&emsp;如果没有指定 output 的绝对路径，默认会将头文件保存到当前的二进制输出目录：

```cmake
target_include_directories(app PRIVATE "${CMAKE_CURRENT_BINARY_DIR}")
```

### 头文件预编译

&emsp;&emsp;主流的编译器都提供了头文件的预编译技术，通过复用一些稳定且被反复包含的头文件来加速编译过程。CMake 使用预编译头文件还是比较简单，至少比手写编译指令要简单一些。


```cmake
# command 1
target_precompile_headers(<target>
	<INTERFACE|PUBLIC|PRIVATE> [header1...]
	[<INTERFACE|PUBLIC|PRIVATE> [header2...] ...])
	
# command 2
target_precompile_headers(<target> REUSE_FROM <other_target>)
```
1. 目标使用预编译头文件：

```cmake
add_executable(app main.cpp)
target_precompile_headers(main PRIVATE <vector>)
```

&emsp;&emsp;在 mian.cpp 中甚至不需要包含有关 vector 的头文件文件，由 CMake 自动强制包含预编译出来的文件。

2. 复用其他目标的预编译头文件：

```cmake
target_precompile_headers(app2 REUSE_FROM app)
```

&emsp;&emsp;官方文档说复用预编译头文件的两个目标会存在依赖关系，并且要求目标之间的编译属性都严格匹配 (emmm...)。
