---
title: CSAPP 05 - 程序优化
date: 2023-08-02 14:16:59
updated: 2023-08-05 19:00:00
tags:
  - CSAPP
  - C
categories:
  - CSAPP
---

&emsp;&emsp;一个运行的很快但是给出错误结果的程序没有任何用处。

<!-- more -->

## 概念

&emsp;&emsp;程序优化是一个高级话题，能够有效优化代码需要了解编程语言，编译器，汇编，处理器以及内存等主题。一个运行的很快但是给出错误结果的程序没有任何用处。编写高效的程序需要做到以下三点：

1. 必须选用一组适当的算法和数据结构。
2. 必须编写编译器能够有效优化的的源代码。
3. 探索多核和多处理器组合上的并行计算。

## 妨碍因素

&emsp;&emsp;现代编译器的优化能力十分强大，它基于一组复杂且精细的机制来理解源代码，并且很`小心`的进行代码优化，编译器必需考虑到代码逻辑中的所有情况，以此来保证忠于语言标准和源代码。正因此，即使是细微的逻辑上的差异也可能导致编译器改变优化策略，甚至放弃优化。

### 内存别名使用

&emsp;&emsp;两个指针可能指向同一个内存位置的情况称作内存别名使用，这是一种潜在的情况：

```c
void add1(long *xp, long *yp)
{
    *xp += *yp;
    *xp += *yp;
}

void add2(long *xp, long *yp)
{
    *xp += 2 * (*yp);
}
```

&emsp;&emsp;add1 函数涉及 6 次内存读写，而 add2 仅有 3 次。但是考虑到 xp 和 yp 指针可能指向同一块内存，编译器不会将 add1 优化为 add2 的版本。如果 xp 和 yp 指向同一块内存，那么 add1 将使目的值被扩大四倍，而 add2 会使目的值被扩大三倍。

&emsp;&emsp;另一个例子：

```c
x = 1000; y = 3000;
*q = x;
*p = y;
t1 = *q;
```

&emsp;&emsp;因为无法保证 p 和 q 不指向同一片内存，所以这里对 q 的解引用无法省略。

### 函数的副作用

&emsp;&emsp;如果一个函数修改了外部状态，那么就可以说这个函数是有副作用的。大多数编译器不会试图判断一个函数是否没有副作用，相反，编译器会假设最糟糕的情况，并保持所有的函数调用不变：

```c
long f();

long func1()
{
    return f() + f() + f() + f();
}

long func2()
{
    return 4 * f();
}
```

&emsp;&emsp;考虑到函数 f 可能有副作用，因此编译器不会试图使用 func2 来替换 func1。

### 函数内联展开

&emsp;&emsp;函数的内联展开技术已经被编译器广泛采纳，通常只需要开启编译器扩展即可启用：

```c
extern int counter;
long f() { return counter++; }

long func1in()
{
    long t = counter++;
    t += counter++;      /* +1 */
    t += counter++;      /* +2 */
    t += counter++;      /* +3 */
    return t;
}
```

&emsp;&emsp;现在可以针对 func1 内联展开版本进行下一步的优化：

```c
long func1opt()
{
    long t = 4 * counter + 6;
    counter += 4;
    return t;
}
```

&emsp;&emsp;内联展开函数调用能减少函数调用的开销，但是它也有一些坏处：如果一个函数已经用内联替换了，那么任何对于这个函数的调用追踪和断点都将失效，并且在评估程序性能时，内联展开的函数是无法被剖析的。

## 性能评估

&emsp;&emsp;引入`每元素周期数`来作为表示程序性能的度量标准。处理器的活动是由时钟控制的，时钟提供了某个频率的规律信号，通常用千赫兹 (GHz) 表示。用时钟周期作为度量标准比用时间单位更有效。

```c
/* Compute prefix sum of vector a */
void psum1(float a[], float p[], long n)
{
    long i;
    p[0] = a[0];
    for (i = 1; i < n; i++)
        p[i] = p[i-1] + a[i];
}

void psum2(float a[], float p[], long n)
{
    long i;
    p[0] = a[0];
    for (i = 1; i < n-1; i+=2) {
        float mid_val = p[i-1] + a[i];
        p[i] = mid_val;
        p[i+1] = mid_val + a[i+1];
    }
    /* For even n, finish remaining element */
    if (i < n)
        p[i] = p[i-1] + a[i];
}
```

&emsp;&emsp;例如，对于两个计算向量`前置和`的函数，通过测算元素数量 n 和其对应花费周期数，列出图表：

![](01.png)

&emsp;&emsp;其中 psum1 每次迭代计算向量中一个元素，而 psum2 通过`循环展开`技术每次计算向量中的两个元素。他们的运行时间都包括初始化，准备循环以及完成部分。并且开销时间同近似接近`C + k * n`这个函数值，其中 C 是常数值，n 是元素数量，k 被称作`线性因子`即每元素周期数量`CPE`。

&emsp;&emsp;当元素数量 n 足够大时，此时耗费时间主要是由 CPE 值来决定的，因此优化程序的目的就是要降低 CPE。

## 优化策略

&emsp;&emsp;使用一个计算向量前置和的例子来说明一些优化策略：

```c
/* Create abstract data type for vector */
typedef struct {
    long len;
    data_t *data;
} vec_rec, *vec_ptr;
```

&emsp;&emsp;上面定义了两个数据类型，是向量的容器，其中`data_t`在后文分别可以是 int，long，float，double。

```c
/* Create vector of specified length */
vec_ptr new_vec(long len)
{
    /* Allocate header structure */
    vec_ptr result = (vec_ptr) malloc(sizeof(vec_rec));
    data_t *data = NULL;
    if (!result)
        return NULL; /* Couldn’t allocate storage */
    result->len = len;
    /* Allocate array */
    if (len > 0) {
        data = (data_t *)calloc(len, sizeof(data_t));
        if (!data) {
            free((void *) result);
            return NULL; /* Couldn’t allocate storage */
        }
    }
    /* Data will either be NULL or allocated array */
    result->data = data;
    return result;
}

/*
* Retrieve vector element and store at dest.
* Return 0 (out of bounds) or 1 (successful)
*/
int get_vec_element(vec_ptr v, long index, data_t *dest)
{
    if (index < 0 || index >= v->len)
        return 0;
    *dest = v->data[index];
    return 1;
}

/* Return length of vector */
long vec_length(vec_ptr v)
{
    return v->len;
}
```

&emsp;&emsp;下面是测试函数 combine1 的第一个版本：

```c
#define INDENT 0
#define OP     +

/* Implementation with maximum use of data abstraction */
void combine1(vec_ptr v, data_t *dest)
{
    long i;
    *dest = IDENT;
    for (i = 0; i < vec_length(v); i++) {
        data_t val;
        get_vec_element(v, i, &val);
        *dest = *dest OP val;
    }
}
```

&emsp;&emsp;分别测量 combine1 的未优化版本和开启 O1 优化版本，得到数据：

![](02.png)

### 代码移动

&emsp;&emsp;代码移动优化包括识别要执行多次，但是计算结果不会改变的计算。因而可以将计算移动到代码前面不会被多次求值的部分。combine1 中的 vec_length 就可以移动到循环外部。

```c
/* Move call to vec_length out of loop */
void combine2(vec_ptr v, data_t *dest)
{
    long i;
    long length = vec_length(v);

    *dest = IDENT;
    for (i = 0; i < length; i++) {
        data_t val;
        get_vec_element(v, i, &val);
        *dest = *dest OP val;
    }
}
```

&emsp;&emsp;移动代码后的 combine2 和 combine1 的性能对比：

![](03.png)

&emsp;&emsp;编译器不能可靠的发现一个函数是否有副作用，因而假设函数有副作用。此时就需要程序员进行显示的移动。

### 减少过程调用

&emsp;&emsp;通常来说，过程调用会带来开销，而且妨碍大多数形式的程序优化。在 combine2 中，循环内每次都调用 get_vec_element 来获取当前元素。其中产生了函数调用以及边界检查的开销。

&emsp;&emsp;作为替代，引入一个 get_vec_start 函数来获取数组的起始地址，并使用数组索引的方式来访问元素。这种做法去除了过程调用的开销，但是损害了程序的模块性，实际开发中应仔细权衡。

```c
/* Direct access to vector data */
void combine3(vec_ptr v, data_t *dest)
{
    long i;
    long length = vec_length(v);
    data_t *data = get_vec_start(v);

    *dest = IDENT;
    for (i = 0; i < length; i++) {
        *dest = *dest OP data[i];
    }
}
```

&emsp;&emsp;去除过程调用的 combine3 和 combine2 性能对比：

![](04.png)

&emsp;&emsp;惊人的是 combine3 的性能非但没有提高，在整数求和部分还有下降。

### 减少内存引用

&emsp;&emsp;combine3 的代码将合并运算的计算结果累计在指针 dest 指定的位置中。反复的访问内存会降低程序的性能，一个解决方案是：引入一个临时变量，将计算结果存储在临时变量中，只在最后写入到内存中。

```c
/* Accumulate result in local variable */
void combine4(vec_ptr v, data_t *dest)
{
    long i;
    long length = vec_length(v);
    data_t *data = get_vec_start(v);
    data_t acc = IDENT;

    for (i = 0; i < length; i++)
        acc = acc OP data[i];
    
    *dest = acc;
}
```

&emsp;&emsp;观察 combine4 中循环部分对应的汇编代码就能发现：临时结果保存在寄存器 %xmm0 中，省去了每次计算后都要访问内存的开销，性能对比：

```nasm
# Inner loop of combine4. data_t = double, OP = *
# acc in %xmm0, data+i in %rdx, data+length in %rax
.L25:
    vmulsd     (%rdx), %xmm0, %xmm0
    addq       $8, %rdx
    cmpq       %rax, %rdx
    jne        .L25
```

![](05.png)

&emsp;&emsp;注意：combine3 和 combine4 的行为在某些情况时是不同的，如果 dest 指向的是数组的最后一个元素，那么二者得出的结果将不同。这也是编译器选择保守优化的原因，因为编译器无法判断内存别名使用的情况。

## 现代处理器

&emsp;&emsp;以上几个优化策略都不依赖目标机器的任何特性，这些优化都只是简单地减少了一些`妨碍优化`的因素。要进一步提升性能，必须充分考虑利用处理器`微体系结构`的优化。

&emsp;&emsp;现代处理器最伟大的功绩之一是：他们采用复杂而奇异的微处理器结构，其中，多条指令可以并行地执行，同时又呈现出一种简单的顺序执行指令的表象。

&emsp;&emsp;描述程序最大性能的两种下界：

1. 延迟界限：当代码中的数据相关限制处理器利用指令级并行的能力时，延迟界限能够限制程序的性能。
2. 吞吐量界限：刻画了处理器功能单元的原始计算能力，这个界限是程序性能的终极限制。

### 整体操作

&emsp;&emsp;现代处理器已经发展到`超标量`阶段，能够在单个时钟周期内完成多个操作，并且是`乱序`的，即指令的执行顺序不一定要和他们在机器级程序中的顺序一致。

&emsp;&emsp;整体设计有两个主要部分：

1. ICU 指令控制单元：负责从内存中读出指令序列，并根据序列产生一组针对程序数据的微操作。
2. EU 执行单元：负责执行上述操作。

&emsp;&emsp;和`按序`流水线相比，乱序处理器需要更大，更复杂的硬件，但是后者能达到更高的指令级并行度。

![](06.png)

- ICU

&emsp;&emsp;ICU 从`指令高速缓存`中读取指令，指令高速缓存包含最近访问的指令。通常，ICU 会在当前正在执行的指令很早之前取指，以此保留足够的时间对指令译码。当指令遇到`分支`时，处理器的`取指控制`模块采用一种`分支预测`的技术来猜测是否选择分支，并确定跳转到哪里，再进行译码。处理器甚至在确定分支预测是否正确之前就开始执行指令，如果分支预测错误，处理器会将状态设置为分支点的状态，开始取出并执行另一分支的指令。

- 指令译码

&emsp;&emsp;指令译码逻辑接收实际的程序指令，并将他们转化成一组微操作。每个微操作都完成某个简单地计算任务如：两个数相加，从内存中读数据，向内存写数据等。一条复杂 x86 指令可以被译码成多个操作，译码逻辑对指令分解，允许任务在一组专门的硬件单元之间分割。这些单元可以并行的执行多条指令的不同部分。

- 加载存储

&emsp;&emsp;读写内存是由`加载和存储`单元实现的。他们都分别包含一个加法器，用来完成地址计算。加载和存储单元通过`数据高速缓存`来访问内存，数据高速缓存存放着最近访问的数据值。

&emsp;&emsp;使用`投机执行`技术对操作求值，但是最终结果不会存放到程序寄存器或者数据内存中，直到处理器能确定应该实际执行这些指令时。分支操作被送往 EU，不是确定分支应该往哪走，而是确定分支预测是否正确。如果预测错误，EU 会丢弃投机执行的结果，并通知分支单元来指示正确目的地。

### 功能单元

&emsp;&emsp;随着时间推移，在单个微处理器上能集成的晶体管越来越多，后续的处理器都增加了功能单元的数量以及每个单元能执行的操作组合，还提升了每个单元的性能，一些描述功能单元的特征：

1. 延迟：执行实际运算所需要的时钟周期总数。
2. 发射时间：两次运算之间间隔的最小周期数。
3. 容量：同时能发射多少个这样的操作。

![](07.png)

&emsp;&emsp;从整数运算到浮点运算，延迟是增加的。这种很短的发射时间是用`流水线`实现的，发射时间为 1 的功能单元称为`完全流水线化`：每个时钟周期可以开始一个新的运算。

&emsp;&emsp;表达发射时间的一种更常见的方法是指明该功能单元的最大`吞吐量`，定义为发射时间的倒数 (每个时钟周期可以完成多少个运算)。一个完全流水化的功能单元有最大吞吐量 1。而具有多个功能单元可以进一步提高吞吐量，对于一个容量为 C，发射时间为 I 的操作来说，处理器具有 C / I 的吞吐量。

### 循环展开

&emsp;&emsp;循环展开是一种程序变换，通过增加每次迭代计算的元素数量，减少循环的迭代次数。循环展开能从两方面改进程序的性能：减少了不直接有助于结果计算的操作，以及提供了帮助进一步变换代码的方法。

&emsp;&emsp;一个原本要迭代 n 次的循环，对其按任意因子 k 进行展开，由此产生 k * 1 循环展开。迭代上限为 n - k - 1，在循环内对元素 i 到 i + k - 1 应用合并运算，每次迭代后循环索引 i 加 k。

```c
/* 2 x 1 loop unrolling */
void combine5(vec_ptr v, data_t *dest)
{
    long i;
    long length = vec_length(v);
    long limit = length - 1;
    data_t *data = get_vec_start(v);
    data_t acc = IDENT;

    /* Combine 2 elements at a time */
    for (i = 0; i < limit; i+=2)
        acc = (acc OP data[i]) OP data[i+1];

    /* Finish any remaining elements */
    for (; i < length; i++)
        acc = acc OP data[i];

    *dest = acc;
}
```

&emsp;&emsp;分别测算 2 * 1 和 3 * 1 循环展开结果的性能：

![](08.png)

&emsp;&emsp;可以发现两次展开的结果只有微小的提高，但是都没有突破其延迟界限：

```nasm
.L35:
    vmulsd    (%rax,%rdx,8), %xmm0, %xmm0
    vmulsd    8(%rax,%rdx,8), %xmm0, %xmm0
    addq      $2, %rdx
    cmpq      %rdx, %rbp
    jg        .L35
```

&emsp;&emsp;观察汇编结果可以发现，虽然迭代次数减半了，但是两条关键的指令`vmulsd`还是连续的，他们都将结果写入临时变量 acc 中，并且第二条乘法指令对第一条指令的结果有数据依赖，正是这种`顺序`关系导致性能无法突破延迟界限。

### 提高并行性

&emsp;&emsp;之前，程序的性能受运算单元的延迟限制。但是执行加法和乘法的功能单元是完全流水化的，这意味着他们可以每个时钟周期开始一个新操作，并且有些操作可以被多个功能单元执行。

&emsp;&emsp;之前的例子中，累计值都放在一个变量 acc 中，在前一步计算结果出来之前，后面的计算都无法进行，正是这种顺序相关导致程序性能无法突破延迟界限。

#### 多个累积变量

&emsp;&emsp;对于一个`可结合`和`可交换`的合并运算来说，比如加法和乘法。可以通过将一组合并运算分割成两个或更多的部分，并在最后合并结果来提高性能。

&emsp;&emsp;对于一个引入多个累积变量的循环展开，可以把其称作 K * P 展开，其中 K 是展开因子，P 是累积变量的数量。一般都使用 K * K 展开来逼近吞吐量界限。

```c
/* 2 x 2 loop unrolling */
void combine6(vec_ptr v, data_t *dest)
{
    long i;
    long length = vec_length(v);
    long limit = length-1;
    data_t *data = get_vec_start(v);
    data_t acc0 = IDENT;
    data_t acc1 = IDENT;

    /* Combine 2 elements at a time */
    for (i = 0; i < limit; i+=2) {
        acc0 = acc0 OP data[i];
        acc1 = acc1 OP data[i+1];
    }

    /* Finish any remaining elements */
    for (; i < length; i++) 
        acc0 = acc0 OP data[i];

    *dest = acc0 OP acc1;
}
```

&emsp;&emsp;测试 combine6 的 2 * 1 展开版本性能：

![](09.png)

&emsp;&emsp;通常，只有保持能够执行该操作的所有功能单元的流水线都是满的，程序才能达到这个操作的吞吐量界限。对于延迟为 L，容量为 C 的操作而言，要求循环展开因子 k >= C * L。

#### 重新结合变换

&emsp;&emsp;理论上说，只要打破数据顺序相关性，就能提高指令的并发度。下面的 combine7 相比较 combine5 只是改变了运算的结合顺序，将其称作 2 * 1a 展开版本。

```c
/* 2 x 1a loop unrolling */
void combine7(vec_ptr v, data_t *dest)
{
    long i;
    long length = vec_length(v);
    long limit = length-1;
    data_t *data = get_vec_start(v);
    data_t acc = IDENT;

    /* Combine 2 elements at a time */
    for (i = 0; i < limit; i+=2)
        acc = acc OP (data[i] OP data[i+1]);

    /* Finish any remaining elements */
    for (; i < length; i++)
        acc = acc OP data[i];

    *dest = acc;
}
```

&emsp;&emsp;combine7 版本的性能除了整数加法，其他方面都轻易打破了延迟界限：

![](10.png)

&emsp;&emsp;这是因为：针对向量元素 1 和 2 的乘法并不依赖临时结果 acc，每次迭代中第一条乘法指令都不需要依赖上一次的迭代结果就可以被执行。

&emsp;&emsp;注意：在执行重新结合变换时需要考虑改变结合方式对结果的影响。

#### TODO 利用向量指令

&emsp;&emsp;待更新。

### TODO 限制因素

&emsp;&emsp;待更新。
