---
title: CMake 05 - 补充内容
date: 2023-07-06 19:01:30
updated: 2023-07-08 19:00:00
tags:
  - CMake
  - CXX
categories:
  - [Build, CMake]
---

&emsp;&emsp;补充一些 CMake 的其他知识。

<!-- more -->

## Install

&emsp;&emsp;CMake 提供内建指令`install`来生成程序的安装指令。在 Linux 系统中，需要安装程序可能包含可执行文件，库文件，头文件，文档手册等等。

&emsp;&emsp;在构建信息，可执行文件构建完毕后。可以使用下面两种方式执行安装动作，如果没有使用`install`指令配置安装信息，安装操作将无效。

```bash
cmake -B build .
cmake --build build
cmake --install build

# or 

cd build 
cmake ..
make && make install
```

&emsp;&emsp;关于安装信息的配置，注意一下几点：

1. 安装前首先考虑安装目录

&emsp;&emsp;CMake 安装时会读取变量`CMAKE_INSTALL_PREFIX`的值确定安装目录，修改安装目录的方式之一就是直接修改该变量，另外一种方法是在安装时传入安装目录：

```cmake
cmake --install build --prefix /home/arthur/install
```

2. 安装可执行目标

```cmake
install(TARGETS <target>... [...])

# example
add_executable(app main.cc)
install(TARGETS app)
```

&emsp;&emsp;CMake 会自动识别目标类型，如果是可执行文件 (RUNTIME) 的话，会放在安装目录中的`bin`目录下。如果是静态库 (ARCHIVE) 或者动态库 (LIBRARY)，CMake 会将他们放在安装目录中的`lib(64)`目录下。

&emsp;&emsp;`install`指令也提供一些额外的参数来修改默认安装属性，指定`DESTINATION`的值将使 app 目标被安装在 qbin 安装目录中的 qbin 目录下，而不是默认的 bin 目录。`DESTINATION`的值可以是相对路径 (相对于 ${CMAKE_INSTALL_PREFIX})，也可以是绝对路径。

```cmake
install(
	TARGETS app
	RUNTIME DESTINATION qbin
)
```

3. 安装文件和目录

&emsp;&emsp;除了安装目标之外，还可以自定义安装文件和目录，语法类似目标的安装。

* 安装文件：

```cmake
install(FILES "/path/to/file" DESTINATION "dir")

install(FILES "/path/to/file" DESTINATION "dir" RENAME "new name")
```

* 安装目录：

```cmake
install(DIRECTORY "include/" DESTINATION "dir")
```

&emsp;&emsp;在 Linux 系统上，安装目录有着一些默认的规范，比如：可执行文件放在 bin 目录下，头文件放在 include 目录下，库文件放在 lib/lib64 目录下。CMake 内部提供这些目录的变量，也提供了简便写法。

| 目标类型       | GNUInstallDirs 变量         | 默认位置 |
| -------------- | --------------------------- | -------- |
| RUNTIME        | ${CMAKE_INSTALL_BINDIR}     | bin      |
| ARCHIVE        | ${CMAKE_INSTALL_LIBDIR}     | lib(64)  |
| LIBRARY        | ${CMAKE_INSTALL_LIBDIR}     | lib(64)  |
| PRIVATE_HEADER | ${CMAKE_INSTALL_INCLUDEDIR} | include  |
| PUBLIC_HEADER  | ${CMAKE_INSTALL_INCLUDEDIR} | include  |

* 通过引入内部模块`GNUInstallDirs`来使用这些变量：

```cmake
include(GNUInstallDirs)

install(
	TARGETS app
	DESTINATION "${CMAKE_INSTALL_BINDIR}"
)

# install directory
install(
	DIRECTORY "include/"
	DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
)
```

* 通过使用`TYPE`属性来简化写法 (用于替代 DESTINATION)：

```cmake
install(TARGETS app TYPE BIN)

# install directory
install(DIRECTORY "include/" TYPE INCLUDE)
```

| TYPE    | GNUInstallDirs           | VALUE   |
| ------- | ------------------------ | ------- |
| BIN     | ${CMAKE_INSTALL_BINDIR}  | bin     |
| LIB     | ${CMAKE_INSTALL_LIBDIR}  | lib(64) |
| INCLUDE | ${CMAKE_INSTALL_LIBDIR}  | lib(64) |
| DATA    | ${CMAKE_INSTALL_DATADIR} | share   |

## Package

&emsp;&emsp;除了构建和安装程序外，对于库来说，一个关键问题是能否简单的分发给其他用户使用。在 CMake 中使用一个库，无非就是要知道其可链接文件 (.a .so)，和头文件在哪。

&emsp;&emsp;用户下载这个库的源代码，并且使用 CMake 构建，编译，然后安装到本地。用户在其他项目中使用这个库时，当然可以直接从安装目录中找到可链接文件和头文件，然后通过 link 和 include 指令来使用。但是这样做，既增加了使用者的管理成本，也使得用户项目再分发时不灵活。

&emsp;&emsp;CMake 提供了包机制来解决这一问题，库作者通过配置包信息，让用户在构建，安装后除了得到原始的可链接文件和头文件，还得到了一个包。在用户的项目中使用该库，也只使用一系列与包相关的指令。

&emsp;&emsp;因此库的分发方需要在安装库文件时，额外配置包信息，并且完成包脚本的安装。CMake 中有三种包：Config-file 包，Find-module 包和 pkg-config 包。Config-file 包是最容易集成的，因为这代表项目本身是使用 CMake 构建或提供了 CMake 构建的版本。而 Find-module 包则是在没有 Config-file 包提供时的下策，可能是其他用户编写的，用于查找库的 CMake 脚本。

```cmake
find_package(fmt REQUIRED)
target_link_libraries(app PRIVATE fmt::fmt)
```

&emsp;&emsp;通过内建指令`find_package`来查找一个包，CMake 会在预订目录中查找相关脚本 (也就是库分发方提供的)，使用包中库也十分简单清晰，使用 fmt 库甚至不需要再手动包含它的头文件目录，它在链接时自动包含 (库作者的功劳)。

### Config-file

&emsp;&emsp;对于库作者来说，要做的就是在库安装后，额外提供一个包脚本。CMake 提供了内建指令来帮助生成脚本，作者只需要配置好库信息，然后调用指令生成脚本，并且在安装到指定目录即可。

&emsp;&emsp;如果现在有一个动态库要提供给其他用户使用，大概有以下步骤：

1. 安装可执行文件和头文件：

```cmake
# build target d
add_subdirectory(src)

# install library and headers
include(GNUInstallDirs)
install(TARGETS d EXPORT d_export)
install(DIRECTORY "include/" TYPE INCLUDE)
```

&emsp;&emsp;注意安装目标时要使用`export`指令导出目标供下一步使用，这里其实是把安装目标导出到一个所谓的导出组中，后面配置`Config-file`时需要提供导出组信息。

2. 调整目标包含的头文件目录：

```cmake
# add library
add_library(d SHARED)

# bind s headers
target_include_directories(
    d PUBLIC
    $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
)
```

&emsp;&emsp;之前例子中是直接使目标包含 include 目录，但是这是等于将路径硬编码了。用户在使用时肯定没办法寻找到头文件目录 (include 目录存在于 install 目录中)，因此这里使用生成器表达式来延迟目录的包含。

&emsp;&emsp;这两条语句的意思很简单：如果项目执行`cmake --build`指令构建，那么就包含源码路径中的 include 目录，如果此时项目执行`cmake --install`指令安装，就包含 install 目录中的 include 目录。

3. 导出必要文件`lib.cmake`：

```cmake
# export libs
include(CMakePackageConfigHelpers)

install(
    EXPORT d_export
    NAMESPACE Arthur::
    FILE "d.cmake"
    DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/Arthur"
)
```

&emsp;&emsp;`lib.cmake`文件是必须要使用的，而它具体写法也不用我们关心，只需要使用`install`指令导出即可。这里就使用到了上面生成的导出组，还可以自己指定库的`namespace`。最后是导出目录，这个导出目录位于安装目录中的`/usr/lib(64)/cmake`，如果想分开管理，可以添加子目录。

4. 生成脚本：

&emsp;&emsp;生成脚本需要一个模板，模板的基本内容很简单：

```cmake
# file: libd/cmake/config.cmake.in

@PACKAGE_INIT@
include("${CMAKE_CURRENT_LIST_DIR}/d.cmake")
```

&emsp;&emsp;这个模板第一行是包的初始化，由 CMake 负责展开，接着是使用上一步导出的`lib_name.cmake`文件。根据模板生成脚本文件也要用到一个内建指令：

```cmake
configure_package_config_file(
    "cmake/config.cmake.in"
    "d-config.cmake"
    INSTALL_DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/Arthur"
)
```

&emsp;&emsp;生成了`lib_name-config.cmake`基本上就能提供给外部使用了，有需要的话还可以添加版本管理脚本：

```cmake
write_basic_package_version_file(
    "${CMAKE_CURRENT_BINARY_DIR}/d-config-version.cmake"
    VERSION ${PROJECT_VERSION}
    COMPATIBILITY SameMajorVersion
)
```

5. 安装脚本文件：

&emsp;&emsp;上面生成了三个必要的文件：`lib_name.cmake`，`lib_name-config.cmake`，`lib_name-config-version.cmake`。最后一步就是把脚本文件安装到安装目录中：

```cmake
install(
    FILES
    "${CMAKE_CURRENT_BINARY_DIR}/d-config.cmake"
    "${CMAKE_CURRENT_BINARY_DIR}/d-config-version.cmake"

    DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/Arthur"
)
```

### Usage

&emsp;&emsp;如果库文件和脚本文件都已经安装成功了，就可以在其他项目中方便的使用该库。

```cmake
add_executable(app)

find_package(d REQUIRED 0.0.1)

target_link_libraries(d PRIVATE Arthur::d)
```

&emsp;&emsp;第二行的指令`find_package`将在安装目中寻找文件`d-config.cmake`，然后生成目标 d，然后简单地使用它。

&emsp;&emsp;一般来说：find_package 指令将在安装目录的 lib(64)/cmake 子目录下寻找这些文件，我们的脚本也安装在这些目录中，如果希望在非安装目录寻找这些文件的话，可以如下操作：

```cmake
set(Arthur_DIR "/path/to/cmake/Arthur")
set(CMAKE_PREFIX_PATH ${Arthur_DIR} ${CMAKE_PREFIX_PATH})
```

&emsp;&emsp;这样的话，find_package 将会检测这个列表变量中包含的目录。还有一种方法是：直接指定库脚本所在的目录。

```cmake
# before find_package
set(d_DIR "/directory/to/d-config.cmake")
```
