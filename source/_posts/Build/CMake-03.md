---
title: CMake 03 - 语法补充
date: 2023-06-25 17:17:31
tags:
    - CMake
    - CXX
categories:
    - [Build, CMake]
---

&emsp;&emsp;这里总结其他 CMake 语言细节，来作为对 CMake 02 的补充。

<!-- more -->

## 控制流程

### 条件

&emsp;&emsp;if 指令用于条件判断，下面是其基本格式。可以有多条`elseif`语句：

```cmake
if(cond)
	# do something
elseif(cond)
  	# do something
else()
	# do something
endif()
```

&emsp;&emsp;&emsp;对于 if 指令，其基本的判断形式有以下几种，在写程序时也可以按下面的顺序构思：

1. if(constant)：判断结果为真仅当常量为`1`,`ON`,`YES`,`TRUE`,`Y`，非 0 数字。判断为假仅当常量为`0`, `OFF`, `NO`,`FALSE`,`N`,`IGNORE`,`NOTFOUND`,`""`,或以`-NOTFOUND`结尾的字符串。如果给定的参数不能断定真假，就当作变量或字符串来处理。
2. if(variable)：如果变量未定义，则判断结果为假。当且仅当变量定义了，并且变量的值不是上面`false constant`的，那么判断结果为真。注意：不能用来判断环境变量是否定义：if (ENV{varName}) 总是为假。
3. if("string")：判断结果为真当且仅当字符串的值是上面列举的`true constant`，否则一个字符串总是被判断为假。

&emsp;&emsp;对于检测变量是否定义，请使用下面推荐的存在性检测方法。

#### 逻辑运算符

&emsp;&emsp;if 也支持逻辑运算符，也可以控制优先级：

1. NOT: if (NOT cond)
2. AND: if (cond1 AND cond2)
3. OR : if (cond1 OR  cond2)

4. () : if (cond1 AND (cond2 OR cond3))

#### 存在性检测

1. if(DEFINED name | CACHE{name} | ENV{name})：用于检测普通变量，缓存变量，环境变量是否定义。
2. if(TARGET target)：用于检测是否存在目标。
3. if(var|str IN_LIST variable)：用于检测列表中是否存在元素。

#### 文件检测

1. if(EXISTS path)：检测给定路径是否是文件或者目录，官方建议使用绝对路径 (~/ 是相对路径)。如果路径是符号连接，仅当符号连接的目标存在时返回 true。
2. if(IS_DIRECTORY path)：检测给定路径是否是目录，也要求是绝对路径。
3. if(IS_ABSOLUTE path)：检测给定路径是否是绝对路径。
4. if(IS_SYMLINK path)：检测给定路径是否是符号连接，要求是绝对路径。

#### 比较操作

1. if(var|str MATCHES var|str)：检测给定字符串或者变量的值是否满足正则表达式。

&emsp;&emsp;以下是数学比较，如果一方操作数不是合法的数字那么返回 false：

2. if(var|str LESS var|str)：判断前者是否小于后者。

3. if(var|str GREATER var|str)：判断前者是否大于后者。

4. if(var|str EQUAL var|str)：判断前者是否等于后者。

5. if(var|str LESS_EQUAL var|str)：判断前者是否小于等于后者。

&emsp;&emsp;还有更多就不一一列举了，推荐参考官方文档：[if 指令](https://cmake.org/cmake/help/latest/command/if.html#)。

&emsp;&emsp;注意：if 中代码块的作用域是执行方的作用域。也就是说，如果 if 在全局执行，那么它可以创建，修改，删除全局作用域的变量。

### 循环

&emsp;&emsp;CMake 支持两种循环：while 和 foreach。和大多数循环一致，在循环体中可以中断跳出，也可以中断当前执行流直接开始下一次循环，分别使用`break()`和`continue()`指令。

#### while 循环

&emsp;&emsp;while 是最基础，最通用的循环结构：

```cmake
while (cond)
	<commands>
endwhile()
```

&emsp;&emsp;通过在外部定义其他的循环变量，可以控制 while 的行为。

#### foreach 循环

&emsp;&emsp;foreach 循环增强了循环控制的能力，更加方便使用，foreach 循环有多种形式，其相比 while 循环的一大优势在于对列表遍历更加方便。 

1. range：注意默认 range 的区间是两端闭合的。

```cmake
foreach(loop_var RANGE max)
    message(loop_var)
endforeach()
```

&emsp;&emsp;完整形式，可以设置起始值，终止值，可选设置步长。

```cmake
foreach(loop_var RANGE min max [step])
```

2. 遍历列表：这里的用法很灵活，形式有很多种。

```cmake
foreach(loop_var item...)
foreach(loop_var IN LIST list... ITEMS items...)

# basic
foreach(v a b c d)
    message("v: ${v}")
endforeach()

# traverse list
set(ls "a;b;c;d")
foreach(v IN ls)
    message("v: ${v}")
endforeach()

# traverse list with some added itmes
foreach(v IN ls ITEMS e)
    message("v: ${v}")
endforeach()
```

3. 遍历压缩列表：在一个 foreach 中同时遍历多个列表。

```cmake
foreach(loop_var IN ZIP_LISTS list1...)

# method 1
set(L1 "one;two;three;four")
set(L2 "1;2;3;4;5")
foreach(num IN ZIP_LISTS L1 L2)
	message("num_0=${num_0}, num_1=${num_1}")
endforeach()
```

&emsp;&emsp;通过 loop_var_number 来访问本次循环中，指定第几个列表的元素。当然还有另一种方式：定义多个 loop_var 来分别接收对应列表的元素。

```cmake
foreach(loop_var1... IN ZIP_LISTS list1...)

# method 2
foreach(word num IN ZIP_LISTS L1 L2)
    message("word=${word}, num=${num}")
endforeach()
```

&emsp;&emsp;注意：循环代码块的作用域是执行方的作用域。也就是说，如果循环在全局执行，那么它可以创建，修改，删除全局作用域的变量。但是：loop_var 会在循环结束后被注销。

## 定义指令

&emsp;&emsp;前面已经使用了多个内建指令，CMake 允许用户自定义指令，这跟 C/C++ 中的定义宏，函数是相似的。

### 宏

&emsp;&emsp;宏类似 C/C++ 中的概念。也是执行文本替换，宏调用不会有作用域限制，它的当前作用域是和调用方相同的。这代表着宏的执行效率很高，但是可能产生副作用。

```cmake
macro(name arg...)
	<command>
endmacro()

# example
macro(mac arg)
    set(arg "new")
    message("arg: ${arg}")
endmacro()

set(arg "old")
mac(${arg})
message("arg: ${arg}")

#[[ output:
	arg: old
	arg: new
#]]
```

&emsp;&emsp;因为宏没有自己的作用域，所以传递给他的参数并不会创建真正的变量，而是做了一种常量折叠，即常量替换。第 12 行可能会被展开成为这样而直接嵌入代码中：

```cmake
set(arg "new")
message("arg: old")
```

### 函数

&emsp;&emsp;函数调用时会产生自己的作用域，它享有调用方符号表的副本。所以函数对比宏，执行效率略低，但是相对安全 (通过 PARENT_SCOPE 还是可以修改调用方的变量)。

```cmake
function(name arg...)
	<command>
endfunction()

# example
function(fn1 arg)
    set(arg 2 PARENT_SCOPE)
    return()
endfunction()

function(fn2 arg)
    fn1(${arg})
    message("arg: ${arg}")
endfunction()

fn2(1)

# output:
# arg: 2
```

&emsp;&emsp;这里不再赘述函数调用过程中变量作用域变化了。每个函数还有一些内置的变量，这些变量的值由 CMake 提供：

1. CMAKE_CURRENT_FUNCTION：当前函数名称。
2. CMAKE_CURRENT_FUNCTION_LIST_DIR：定义函数文件的目录。
3. CMAKE_CURRENT_FUNCTION_LIST_FILE：定义函数的文件路径。

## 内建指令

&emsp;&emsp;`include`指令用于包含并执行另一个 CMake 文件。效果类似直接将另一个文件的内容插入到包含位置。下面是一些使用注意事项：

1. include 不会为包含的文件创建单独作用域，可能会有副作用。
2. include 不能循环包含，会因递归层数过深而报错。

```cmake
# a.cmake
set(var 1)
include(b.cmake)

message("var: ${var}")

# output:
# var: 2
```

```cmake
# b.cmake
set(var 2)
```

&emsp;&emsp;CMake 提供了`include_guard()`指令用于保证一个文件只被包含一次，用于防止多次包含和递归包含。只需要在每个文件头部加上该指令即可。

```cmake
# a.cmake
include_guard()
set(var 1)
include(b.cmake)

message("var: ${var}")

# output:
# var: 2
```

```cmake
# b.cmake
include_guard()
set(var 2)
include(a.cmake)
```