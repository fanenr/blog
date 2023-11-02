---
title: C++ 01 - 入门
date: 2023-07-14 11:18:59
updated: 2023-07-16 19:00:00
tags:
  - CXX
  - C
categories:
  - CXX
---

&emsp;&emsp;一些基础的 C++ 知识。

<!-- more -->

## 基本类型

&emsp;&emsp;C++ 的基本类型体系大体沿用 C 语言，但是经过长期发展，C++ 不断引入一些新类型，并且强化类型的统一的命名。

### 整数类型

&emsp;&emsp;整数类型基本脱胎于 C 语言，但是 C++ 还引入了超长整型`long long (int)`。

&emsp;&emsp;注意整型的大小由编译器决定，这里列举的是 x86-64 Linux 平台下 GCC 实现的大小。

| Type                     | Size   | Usage                |
| ------------------------ | ------ | -------------------- |
| (signed) int / signed    | 4 byte | int i = 1;           |
| (signed) short (int)     | 2 byte | short i = 2;         |
| (signed) long (int)      | 4 byte | long i = 3`L`;       |
| (signed) long long (int) | 8 byte | long long i = 4`LL`; |

&emsp;&emsp;无符号整数和有符号整数对应，无符号整数的大小 >= 0。
| Type                     | Size   | Usage                          |
| ------------------------ | ------ | ------------------------------ |
| unsigned (int)           | 4 byte | unsigned i = 1`U`;             |
| unsigned short (int)     | 2 byte | unsigned short i = 2`U`;       |
| unsigned long (int)      | 4 byte | unsigned long i = 3`UL`;       |
| unsigned long long (int) | 8 byte | unsigned long long i = 4`ULL`; |

### 浮点类型

&emsp;&emsp;不加字面量后缀则表明该字面量是双精度浮点类型。

| Type        | Size      | Usage                 |
| ----------- | --------- | --------------------- |
| float       | 4 byte    | float f = 1.0`f`;     |
| double      | 8 byte    | double f = 1.0;       |
| long double | >= double | long double = 1.0`L`; |

### 字符类型

&emsp;&emsp;`char`，`signed char`，`unsigned char`是三种不同的类型。char 类型可能是有符号的，也可能是无符号的。

&emsp;&emsp;char 基本用途就是表示单个字符，但是它也被广泛用于表示单个字节，从 C++17 开始，考虑使用`std::byte`替代 char (位于`<cstddef>`)。

| Type          | Description                         | Usage                  |
| ------------- | ----------------------------------- | ---------------------- |
| char          | 单个字符                            | char c = 'a';          |
| signed char   |                                     | signed char c = 'b';   |
| unsigned char |                                     | unsigned char c = 'c'; |
| char8_t       | 单个 n 位 UTF-n 编码的 Unicode 字符 | char8_t c = `u8`'a';   |
| char16_t      |                                     | char16_t c = `u`'b';   |
| char32_t      |                                     | char32_t c = `U`'c';   |
| wchar_t       | 单个宽字符，大小取决于编译器        | wchar_t c = `L`'c';    |

### 数值极限

&emsp;&emsp;C 语言中定义的宏 (e.g. INT_MAX) 在这里仍然可用，但是推荐使用类模板`std::numeric_limits`来统一获取。

```cpp
// 1. max 
auto imax = std::numeric_limits<int>::max();

// 2. min
auto imin = std::numeric_limits<int>::min();

// 3. lowest
auto ilow = std::numeric_limits<int>::lowest();
```

&emsp;&emsp;对于整数而言 min 和 lowest 的效果一致。但是对于浮点数而言，min 返回浮点类型能表示的最小正数，lowest 返回浮点类型能表示的最小负数。

## 初始化

&emsp;&emsp;使用统一初始化语法`{}`可以方便的初始化变量以及成员。

```cpp
int  i = { 0 };
int  i = {};
int  i {};

int* p {};
```

&emsp;&emsp;对于零初始化：整型被初始化为 0，浮点型被初始化为 0.0，指针类型被初始化为 nullptr，对象将调用其默认构造函数进行初始化。

&emsp;&emsp;使用统一初始化语法进行零初始化时，可以只初始化部分成员，而其他部分将进行默认的零初始化：

```cpp
int arr[3] { 1 };   // arr: 1 0 0

struct data {
    int a;
    double b;
};

data val { 1 };     // val: .a = 1, .b = 0.0
```

&emsp;&emsp;统一初始化除了可以方便的零初始化外，还可以阻止窄化：

```cpp
int var { 2.2 };    // wrong: double to int
```

&emsp;&emsp;现在可以使用指派初始化器来方便的初始化类和结构体，其中未制定的成员将被默认零初始化 (用于跳过某些成员)：

```cpp
struct data {
    int a;
    double b;
};

data val { .a = 1 };    // val: .a = 1, .b = 0.0
```

## 枚举类型

&emsp;&emsp;C 语言中的非限域枚举在 C++ 中依然可用，但是现在更推荐使用限域枚举类型。

1. 非限域枚举

```cpp
enum Color {
    Red,
    Green,
    Blue
};

Color col = Red;

int Red = 1;    // wrong
```

2. 限域枚举

```cpp
enum class Color {
    Red,
    Green,
    Blue
};

Color col = Color::Red;

int Red = 1;    // ok
```

&emsp;&emsp;枚举类型底层默认是整型 (int)，可以通过以下方式改变：

```cpp
enum class Color : unsigned {
    Red,
    Green,
    Blue
};

Color col = Color::Red;
col = static_cast<Color>(1);

auto val = static_cast<unsigned>(col);
```

&emsp;&emsp;枚举项默认从 0 开始排序，可以人为指定：

```cpp
enum class Color : unsigned {
    Red = 1,
    Green,      // 2
    Blue = 4
};
```

&emsp;&emsp;现在 using 语法支持将限域枚举的枚举项引入到当前作用域：

```cpp
using enum Color;
// using Color::Red;

auto red = static_cast<unsigned>(Red);
```



## if 语句

&emsp;&emsp;现在 if 语句允许包含一个初始化器，初始化器中初始化的变量作用域仅限 if 语句内：

```cpp
int i = 1;

if (int i = 2; i == 2) {
	std::cout << i; // 2
} else {
	std::cout << i; // 2
}

std::cout << i;     // 1
```

## switch 语句

&emsp;&emsp;现在 switch 语句也允许包含一个初始化器，初始化器中初始化的变量作用域仅限 if 语句内：

```cpp
int i = 1;

switch (int i = 2; i) {
    case 2: ;
        
    default: ;
}
```

&emsp;&emsp;switch 语句表达式只接受整型，可转换为整型的类型，枚举和强类型枚举。

&emsp;&emsp;现在 switch 语句内可以使用 using 语句来引入强类型枚举的枚举项作为符号名：

```cpp
Color col = Color::Red;

switch (col) {
    using enum Color;
        
    case Red: ;
    case Green: ;
    case Blue: ;
}
```

&emsp;&emsp;多数编译器在忘掉 break 语句时会给出警告，现在可以使用 attribute 语法来告诉编译器这是有意为之的：

```cpp
switch (col) {
    using enum Color;
        
    case Red: [[fallthrough]];
    case Green: [[fallthrough]];
    case Blue: ;
}
```

## 三向比较

&emsp;&emsp;三向比较是一种新语法，同时它也引入了一个新的运算符`<=>`，三向比较返回两个值的顺序。

&emsp;&emsp;三向比较返回类枚举类型 (并非枚举)，引入`<compare>`来使用结果：

```cpp
#include <compare>

auto res = 1 <=> 2;
```

&emsp;&emsp;对于整数类型的三向比较，结果类型是`std::strong_ordering`，值包含三类：

1. std::strong_ordering::`less`：operand1 < operand2
2. std::strong_ordering::`greater`：operand1 > operand2
3. std::strong_ordering::`equal`：operand1 = operand2

&emsp;&emsp;对于浮点类型的三向比较，结果类型是`std::partial_ordering`，值包含四类：

1. std::partial_ordering::`less`：operand1 < operand2
2. std::partial_ordering::`greater`：operand1 > operand2
3. std::partial_ordering::`equal`：operand1 = operand2
4. std::partial_ordering::`unordered`：有一个或两个操作数是非数字。

&emsp;&emsp;还有一种弱排序`std::weak_ordering`，一般用于实现自己类型的排序；

1. std::weak_ordering::`less`：operand1 < operand2
2. std::weak_ordering::`greater`：operand1 > operand2
3. std::weak_ordering::`equal`：operand1 = operand2

&emsp;&emsp;可以直接使用上面的值来判断排序结果，标准库也提供了一系列函数来辅助判断：

| Function          | Meaning | Function          | Meaning |
| ----------------- | ------- | ----------------- | ------- |
| std::is_eq(res)   | =       | std::is_neq(res)  | !=      |
| std::is_gt(res)   | >       | std::is_lt(res)   | <       |
| std::is_gteq(res) | >=      | std::is_lteq(res) | <=      |

```cpp
auto res = 1 <=> 2;

if (std::is_gteq(res))
    ;
else
    ;
```

## 属性

&emsp;&emsp;现在使用统一的`[[]]`语法来声明属性，有以下几个通用属性：

1. [[nodiscard("reason")]]：可用于函数返回值，类和枚举类型等

&emsp;&emsp;修饰为 nodiscard 的类型只有按值返回时被丢弃才会发出警告，返回引用，指针等无效。

```cpp
struct [[nodiscard("must handle")]] data { };

data res { };
data get() { return res; }
data& get_ref() { return res; }
data* ger_ptr() { return &res; }

int num = 42;
[[nodiscard]] int* fun() { return &num; }

int main()
{
    get();          // warning
    get_ref();      // ok
    get_ptr();      // ok
    fun();          // warning
}
```

2. [[maybe_unused]]：可用于变量，函数参数，类和枚举等

&emsp;&emsp;修饰为 maybe_unused 的未使用变量不会引起编译器的警告，但是它不影响编译器的后续优化逻辑。

```cpp
void func([[maybe_unused]] int data)
{ }

int main()
{
    [[maybe_unused]] int i = 1;
}
```

3. [[noreturn]]：只可用于函数

&emsp;&emsp;修饰为 noreturn 的函数必须不返回，即永远不将控制权返还给调用点，如果返回了将产生未定义行为。

```cpp
[[noreturn]] void quit()
{
    std::abort();
}

int func(bool cond)
{
	if (cond)
        return 1;
    else
        quit();    // ok
}
```

4. [[deprecated("reason")]]：可用于函数，类和枚举等

&emsp;&emsp;使用标记为 deprecated 的函数和类型时，编译器将产生警告。

```cpp
[[deprecated]] void old();

[[deprecated("use new instead")]] void old2();

int main()
{
    old();
    old2();
}
```

5. [[likely]] 和 [[unlikely]]：只可用于分支优化

&emsp;&emsp;现代编译器优化能力已经相当惊人了，可能在部分极端情况下才需要人为辅助优化。

```cpp
int i = 1;

// in if
if (i == 2) [[unlikely]] {
    
} else
    ;

// in switch
switch (i) {
    [[likely]] case 1: ;
    
    [[unlikely]] case 2: ;
}
```

## 数组

&emsp;&emsp;声明数组时，其大小必须是常量或者常量表达式。

```cpp
constexpr std::size_t len = 10;
int arr[len];
```

&emsp;&emsp;初始化数组可以使用统一初始化语法：

```cpp
int arr[10] = { 0 };
int arr[10] = {};
int arr[10] {};
```

&emsp;&emsp;有两种方式可以获取基于栈的 C 风格数组的大小：

1. sizeof(arr) / sizeof(arr[0])
2. std::size(arr)

&emsp;&emsp;因为数组在传递时会发生退化，丢失大小信息，考虑使用标注库中的`std::array`来替换静态数组。

## 循环

&emsp;&emsp;总共有四种循环结构：while 循环，do while 循环，for 循环，range-based for 循环。

&emsp;&emsp;在循环体中可以使用 break 语句跳出当前循环，使用 continue 语句进入下一次循环。

&emsp;&emsp;for 循环结构包含初始化器，循环条件和循环控制三个部分：

```cpp
for (int i = 0; i < 10; ++i)
    std::cout << i;
```

1. 初始化变量。
2. 判断条件是否成立：T -> 执行循环体，F -> for 循环结束。
3. 执行完循环体后：执行控制语句，重复步骤 2。

&emsp;&emsp;如果循环体内调用 continue，那么就立即结束本次循环体的执行，跳转到步骤 3。

&emsp;&emsp;range-based for 循环使用十分频繁，其可用于 C 风格的数组，初始化列表，任何具有返回迭代器 begin 和 end 方法的容器。

```cpp
std::array<int, 3> arr { 1, 2, 3 };

for (auto v : arr)
    std::cout << v;
```

&emsp;&emsp;现在 range-based for 循环允许添加一个初始化器：

```cpp
for (std::array<int, 3> arr { 1, 2, 3 }; auto v : arr)
    std::cout << v;
```

## 命名空间

&emsp;&emsp;使用命名空间的目的是防止符号冲突。

```cpp
int get();

namespace my {

int get();

};
```

&emsp;&emsp;现在可以方便的嵌套声明命名空间：

```cpp
namespace my::net::tcp { };
```

&emsp;&emsp;可以使用 using 语句引入命名空间中的符号到当前作用域：

```cpp
using namespace my;     // all

using my::get;          // only one identifier
```

&emsp;&emsp;使用命名空间中的符号使用域解析运算符`::`，对于全局符号的访问，可以直接使用符号名，也可以加上`::`前缀访问。

&emsp;&emsp;当一个全局作用域和匿名命名空间中同时出现一个符号时：

```cpp
int get() { return 1; }

namespace {

int get() { return 2; }
    
}

get();      // wrong
::get();    // ok: 1
```

&emsp;&emsp;命名空间也可以内联，内联的命名空间除了有普通命名空间的性质外，被内联的命名空间中的符号也会导出到父作用域中，使得使用上有一些方便。

&emsp;&emsp;例如：`std::string`的标准字面量`s`定义在多个内联的命名的空间中：

```cpp
namespace std {
    inline namespace literals {
        inline namespace string_literals {
            // some defination
        }
    }
}
```

&emsp;&emsp;以下方式都可以导入标准字面量`s`的符号来使用：

```cpp
using namespace std;
using namespace std::literials;
using namespace std::string_literals;
using namespace std::literials::string_literals;
```

## 结构化绑定

&emsp;&emsp;结构化绑定允许使用一个对象的成员或者元素一次性实例化多个实体：

```cpp
auto [elem1, elem2, ...] = object;

auto const& [elem1, elem2, ...] = object;

auto& [elem1, elem2, ...] = object;
```

&emsp;&emsp;结构化绑定实质上创建了一个匿名对象，这个匿名对象由 object 来初始化，解构时指明的 elem1 等都是这个匿名对象成员 (元素) 的别名，注意是别名，不是引用。

```cpp
struct data {
    int i;
    float f;
};

data obj;
auto [a, b] = obj;
a = 10;
b = 0.0;
decltype(a) x = 0;

// maybe it will be handled like this

data obj_temp = obj;
obj_temp.i = 10;
obj_temp.f = 0.0;
decltype(obj_temp.i) x = 0;
```

&emsp;&emsp;一些使用结构化绑定的例子：

1. C 风格数组

&emsp;&emsp;对原生数组使用结构化绑定时，上下文不能丢失数组长度的信息，即如果数组发生退化时就无法使用。

```cpp
int arr[3] = {};
// will copy arr
auto [a, b, c] = arr;
auto [a, b, c] (arr);
auto [a, b, c] { arr };
```

2. 结构体和类

&emsp;&emsp;结构化绑定只能用于有 public 成员的类和结构体，并且解构时按照声明顺序匹配。

```cpp
class data {
public:
    int i;
    float f;
};

data obj;
auto [a, b] = obj;
```

3. pair，tuple 和 array

&emsp;&emsp;现在配合 if，range-based for，switch 的初始化器，使用结构化绑定来遍历标准容器十分方便。

```cpp
std::map<std::string, int> map;
// some actions
for (auto const& [k, v] : map)
    std::cout << format("key: {}, value: {}\n", k, v);
```

## TODO 字符串

&emsp;&emsp;待更新。
