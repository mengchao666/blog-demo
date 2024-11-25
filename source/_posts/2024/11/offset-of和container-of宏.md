---
title: offset_of和container_of宏
tags: [kernel]
categories: [kernel]
permalink: 
poster:
  topic: null
  headline: null
  caption: null
  color: null
date: 2024-11-25 22:51:03
topic: linux_kernel
description:
cover:
banner:
references:
---
## 一、container_of 宏介绍

到这里假设大家都懂了 **typeof** 和 **语句表达式**，那么我们就开始一睹 Linux 内核第一宏 container_of 的芳容吧：

```C
#define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)
#define container_of(ptr, type, member) ({ \
    const typeof( ((type *)0)->member ) *__mptr = (ptr); \
    (type *)( (char *)__mptr - offsetof(type,member) );})
```

作为 Linux 内核第一个宏，绝对是实至名归的，看看它外表斯文而内藏八块腹肌的身形，就知道它是不好惹的。宏中有宏，作为 GNU C 高端扩展特性的综合运用，那么它有什么作用呢？它的主要**作用**是：**根据结构体某一成员的地址，获取这个结构体的首地址。** 根据宏定义，可知这个宏有三个参数：

* ptr：结构体内成员 member 的地址
* type：结构体类型
* member：结构体内的成员

也就是说，当我们知道了一个结构体的类型，结构体内某一成员的地址，也就可以直接获得到这个结构体的首地址。container_of 宏返回的就是这个结构体的首地址。

## 二、container_of 宏的使用示例

这个宏在内核中非常重要。在内核中会经常有这样的需求：我们传递给某个函数的参数是某个结构体的成员变量，然后在这个函数中，可能还会用到此结构体的其它成员变量，那么这个时候怎么办呢？我们可以使用 container_of 先通过结构体某一成员的访问找到这个结构体的首地址，然后就可以访问其它成员变量了。

```C
struct _box_t
{
    double length;   // 盒子的长度
    double breadth;  // 盒子的宽度
    double height;   // 盒子的高度
};
 
int main(void)
{
    struct _box_t box = {30.0, 20.0, 10.0};
    struct _box_t *p_box = NULL;
 
    p_box = container_of(&box.height, struct _box_t, height);
    printf("%p\n", p_box);
    printf("length: %f\n", p_box->length);
    printf("breadth: %f\n", p_box->breadth);
 
    return 0;
}
```

在这个程序中，我们定义一个结构体变量 box，知道了它的成员变量 height 的地址 &box.height，就可以通过 container_of 宏直接获得 box 结构体变量的首地址，然后直接访问 box 结构体的其它成员 p_box->length 和 p_box->breadth。

## 三、container_of 宏实现原理分析

container_of 宏的实现主要用到的知识为：语句表达式和 typeof，再加上结构体存储的基础知识。为了帮助大家更好地理解这个宏，我们先复习下结构体存储的基础知识。

### 3.1 结构体在内存中的存储

我们知道，结构体作为一个复合类型数据，它里面可以有多个成员。当我们定义一个结构体变量时，编译器要给这个变量在内存中分配存储空间。除了考虑数据类型、字节对齐等因素之外，编译器会按照结构体中各个成员的顺序，在内存中分配一片连续的空间来存储它们。

```C
struct _box_t
{
    double length;   // 盒子的长度
    double breadth;  // 盒子的宽度
    double height;   // 盒子的高度
};
 
int main(void)
{
    struct _box_t box = {30.0, 20.0, 10.0};
    printf("&box = %p\n", &box);
    printf("&box.length = %p\n", &box.length);
    printf("&box.breadth = %p\n", &box.breadth);
    printf("&box.height = %p\n", &box.height);
 
    return 0;
}
```

在这个程序中，我们定义一个结构体，里面有三个 double 型数据成员，我们定义一个变量，然后分别打印结构体的地址、各个成员变量的地址，运行结果如下：

```C
&box         = 2b6c3dd0
&box.length  = 2b6c3dd0
&box.breadth = 2b6c3dd8
&box.height  = 2b6c3de0
```

从运行结果我们可以看到，结构体中的每个成员变量，从结构体首地址开始，依次存放。每个成员变量相对于结构体首地址，都有一个固定偏移。比如 breadth 相对于结构体首地址偏移了8个字节。height 的存储地址，相对于结构体首地址偏移了16个字节。

### 3.2 计算成员变量在结构体内的偏移

一个结构体数据类型，在同一个编译环境下，各个成员相对于结构体首地址的偏移是固定的。我们可以修改一下上面的程序，当结构体的首地址为 0 时，结构体中的各成员地址在数值上等于结构体各成员相对于结构体首地址的偏移。

```C
struct _box_t
{
    double length;   // 盒子的长度
    double breadth;  // 盒子的宽度
    double height;   // 盒子的高度
};
 
int main(void)
{
    printf("&length = %p\n", &((struct _box_t*)0)->length);
    printf("&breadth = %p\n", &((struct _box_t*)0)->breadth);
    printf("&height = %p\n", &((struct _box_t*)0)->height);
 
    return 0;
}
```

在上面的程序中，我们没有直接定义结构体变量，而是将数字 0 通过强制类型转换，转换为一个指向结构体类型为 _box_t 的常量指针，然后分别打印这个常量指针指向的结构体的各成员地址。运行结果如下：

```C
&length  = ox0
&breadth = 0x8
&height  = 0x10
```

因为常量指针为 0，即可以看做结构体首地址为 0，所以结构体中每个成员变量的地址即为该成员相对于结构体首地址的偏移。container_of 宏的实现就是使用这个技巧来实现的。

### 3.3 container_of 宏的原理实现

container_of 宏整体的实现原理如图所示：

![](https://raw.githubusercontent.com/mengchao666/picture/main/blog20241125232615.png)

从语法角度来看，container_of 宏的实现由一个语句表达式构成：

```C
#define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)
#define container_of(ptr, type, member) ({ \
    const typeof( ((type *)0)->member ) *__mptr = (ptr); \
    (type *)( (char *)__mptr - offsetof(type,member) );})
```

**语句表达式的值即为最后一个表达式的值**：

```C
(type *)( (char *)__mptr - offsetof(type,member) );
```

以上这个语句的意义就是，拿结构体某个成员 member 的地址，减去这个成员在结构体 type 中的偏移，结果就是结构体 type 的首地址。因为语句表达式的值等于最后一个表达式的值，所以这个结果也是整个语句表达式的值，container_of 最后就会返回这个地址值给宏的调用者。

内核中定义了 offset 宏来计算结构体某个成员在结构体内的偏移，它的定义如下：

```C
#define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)
```

这个宏有两个参数，一个是结构体类型 TYPE，一个是结构体的成员 MEMBER，它使用的技巧跟我们上面计算 0 地址常量指针的偏移是一样的：将 0 强制转换为一个指向 TYPE 的结构体常量指针，然后通过这个常量指针访问成员，获取成员 MEMBER 的地址，其大小在数值上就等于 MEMBER 在结构体 TYPE 中的偏移。

因为结构体的成员数据类型可以是任意数据类型，所以为了让这个宏兼容各种数据类型。我们定义了一个临时指针变量 \_\_mptr ，该变量用来存储结构体成员 MEMBER 的地址，即存储 ptr 的值。那么如何获取 ptr 指针类型呢？通过下面的方式：

```C
typeof( ((type *)0)->member ) *__mptr = (ptr);
```

以上宏的参数 ptr 代表的是一个结构体成员变量 MEMBER 的地址，所以 ptr 的类型是一个指向 MEMBER 数据类型的指针。为了确保临时变量 \_\_mptr 的指针类型也是一个指向 MEMBER 类型的指针变量，通过 typeof( ((type \*)0)->member ) 表达式，使用 typeof 关键字来获取结构体成员 member 的数据类型，然后使用 typeof( ((type \*)0)->member ) *\_\_mptr 就可以定义一个指向该类型的指针变量了。

注意：在语句表达式的最后，因为返回的是结构体的首地址，所以数据类型还必须强制转换为 TYPE \*，即返回一个指向 TYPE 结构体类型的指针，所以你会在最后一个表达的offset宏中看到一个强制类型转换(TYPE \*)。

## 四、总结

通过对 container_of 宏的整体分析后，这个过程到底对我们有什么启发呢？

对于任何一个复杂的技术，我们都可以把它由上而下的逐步分解，然后运用所学的基础知识一点一点剖析：先进行小模块分析，然后再进行综合分析。

比如 container_of 宏的定义，就运用了结构体的存储、语句表达式、typeof 等知识点。

当我们掌握了这些基础知识，并且有了分析方法，以后在内核中再遇到这样类似的宏，我们就可以自信从容地去自己分析，而不必总是依赖网上大海捞针式的搜索了。

这就是你的核心竞争力，也是你超越其他工程师、脱颖而出的机会。

原文链接：https://blog.csdn.net/m0_37383484/article/details/129244244