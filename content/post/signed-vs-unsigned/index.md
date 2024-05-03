---
title: Signed与Unsigned整型
date: 2024-08-02
tags:
    - c++
    - c
---

# Signed or unsigned?
在C与C++编程中的一个经典又经常被误解的问题是, 应当使用signed还是unsigned整数类型? 这个问题很早就有标准答案:
* C++核心规范: [Arithmetic - ES.100 - ES.107](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#arithmetic)
    > You should not use the unsigned integer types such as `uint32_t`, unless there is a valid reason such as representing a bit pattern rather than a number, or you need defined overflow modulo $2^N$. **In particular, do not use unsigned types to say a number will never be negative. Instead, use assertions for this**.
* [Subscripts and sizes should be signed - Bjarne Stroustrup](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1428r0.pdf)

即: 只要是有**算数意义**的数字, 除非一些例外情况, 都应当使用signed整型. 算数意义是指, 这个数字如果被用于加减乘除等数学运算或者比较大小, 是有意义的.

# 理由和例子

## 非负逻辑
**尤其是, 不要用unsigned整型来表示一个数字逻辑上不能是负数.** 这可能是初学者最常见的错误. 这种情况应该在设计时用contract/pre-condition/post-condition来表示, 并在实现代码内用assert来检查.

例如, 一个函数的参数需要接收一个带有算数意义的整型, 且逻辑上不能为负数. 如果使用Unsigned整型, 调用者传入负数, 那么实际得到的实参也只会被wrap成一个**无有效意义的**正数,
此时函数内甚至无法判断调用者是不小心传入了负数, 还是真的mean to传入这个被wrap的正数. 并且这个wrap过程是well-defined, 并不能被ubsan之类的工具抓到.

关于这一条, 最常见的例子就是数组的下标, offset, 以及数组大小. 但在C与C++标准库, 以及其它的OS原生API中, 基本都是用`size_t`来表示.
* POSIX规定了`ssize_t`, 但这个类型仅仅是用来保证能够返回表示错误的一个sentinel值, 它保证能表示的负数只有-1. (当然, 实际上它一般就是`size_t`的signed版本, 即`std::make_signed_t<size_t>`).

C++标准库中的index与size类型为`size_t`也可以归为历史遗留问题. 对于自己的代码, 建议使用`std::ptrdiff_t`或者`gsl::index`. 

## 混用signed与unsigned
由于C与C++中关于算术操作的类型转换规则, 在算数操作中混用signed和unsigned整型可能会导致一些不直观的后果,
例如最简单的`unsigned(1) < int(-1)`.

至于为什么不用unsigned arithmetics, 理由是unsigned整形的算数规则并不直观(如wrap-around), 容易silently导致错误, 难以发现. 并且缺少负数的算术操作也不方便. 如上一点所说, 这类silent的错误经常无法分辨, 对于溢出类错误(严格说, unsigned整型不存在overflow, 超出范围只有well-defined的wrap-around)尽管不会UB, 但也没有任何帮助, 只会让程序带着错误的数据silently继续运行, 引发不可预测的后果. UB至少还可以用一些工具抓到.

## 优化
signed整形的overflow是UB, 这一点使得编译器能更好的优化, 例如一些index循环. 关于这一点, 引用Zig的一段介绍:
>  Carefully chosen undefined behavior. For example, in Zig both signed and unsigned integers have undefined behavior on overflow, contrasted to only signed integers in C. This [facilitates optimizations that are not available in C](https://godbolt.org/z/n_nLEU).
>
>\- [ziglang overview](https://ziglang.org/learn/overview)

当然, 在Release build里, UB的后果不可预测, 因此, 最好用一些额外手段确保安全, 例如使用checked arithmetics, 产生溢出错误就直接abort, 或者逻辑上确保不会出现overflow.

**无论如何, 用wrap-around的做法逃避算数溢出都不是一个解决方案!**

## 其它
对于正常逻辑上非负的数字, 使用signed整形就多出了负数的范围, 可用来作为sentinel值返回错误. 这是十分常见的做法.
至于这种做法本身适不适合(相比于`std::optional`等其它方式), 属于另一个问题.

# 例外情况
这些情况下, 可以允许(或必要)使用unsigned

## 非算数意义
### bit/byte, 位操作
众所周知, 如果我们只关心访问一串byte, 此时应该使用`unsigned char`来表示字节. 此外, 对于位操作, unsigned整数也更加方便, 例如用于表示一个bit-field.

### 其它
作为一个无算数意义的标记使用. 例如, 一个64位的hash函数一般返回`uint64_t`. 还有一些格式里的`FOURCC`标记等magic number.
一个随机发生器的原始输出一般也是unsigned, 作为一串随机的bit使用.

## 需要额外的1-bit位宽范围 / 省空间
对于32位, 64位的整形, signed整形正数部分少掉的这一位位宽一般不是问题.
但是, 如果我们要表示0-255范围的数字, 那就需要用`uint8_t`(不考虑`CHAR_BIT > 8`的情况), 否则就需要两倍大小. 这种情况下额外的1-bit位宽就matter了.
8bit图像的表示就是常见的例子, 这里的数值显然是具有算数意义的.

### char / signed char / unsigned char
注意区分这三个不同的基础类型. `unsigned char`表示raw字节. `signed char`用于带符号的算数数值. 如上所述, 为了正数位宽也可使用`unsigned char`. `char`应仅用于字符, 不应依赖它实际是signed还是unsigned.

## 需要wrap-around/modular逻辑
比较少见, 但有时我们事实上确实需要用到wrap-around/modular逻辑...

## 二进制序列化
二进制序列化需要统一的数值二进制表示.
不考虑端序, 对于正整数的表示一般没有异议, 但负整数可能存在差异(尽管也十分少见, 绝大多数平台都是2's complement表示).
因此, 二进制序列化/文件格式中, 对于非负的整数数值一般都是直接使用unsigned表示, 包括定点数的存储. 这样做也可以多出1-bit的正数位宽.

# Rust
这些规则大部分并不适用于Rust. Rust中基本的整数类型都适用于算数操作, 可以理解为它们只是具有不同的取值范围, 不允许隐式转换, 并且它们之间的显示转换也是安全的, *除了 `as`*. 对于不安全的转换以及溢出行为, 可以选择使用wrap-around或者panick.
