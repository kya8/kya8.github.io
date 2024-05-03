---
title: "不用动态内存分配如何实现opaque struct和PIMPL?"
date: 2024-12-06
tags:
    - c++
    - c
---

C语言编程中一个常见的practice是将某个struct的definition隐藏, 仅通过指针使用它, 而具体的定义放在一个单独的TU之中,
叫opaque struct. 这种做法的目的一般是为了隐藏struct中的成员, 作为加速编译的隔离手段, 并且还能保证ABI稳定性(因为API只有指针).

在C++中, 当然也可以用C风格的opaque struct, C++并不阻止你这么做, 但通常的做法是用**PIMPL**, 方法也是类似的.

例如, 我们有一个C风格的库`libfoo`:
```c
// foo.h
typedef struct FooCtx FooCtx;

FooCtx* foo_create_context();
void foo_destroy_context(FooCtx*);

int foo_some_function(FooCtx*);

// foo.c
// 包含FooCtx和上述函数的定义
```
使用者只需要拿着这个`FooCtx*`的handle, 不需要关心它内部是什么.

# 避免动态内存分配
常见的opaque struct或者PIML做法, 免不了动态内存分配, 因为`FooCtx`的size在头文件里未知.
有没有什么办法可以绕过这个要求? 作为C/C++程序员, 总是对动态内存分配比较敏感. 有些场景, 比如某些嵌入式平台还没有一般的动态分配可用.

答案是可以, 其实就是把`FooCtx`放在栈上, 用一段栈上的buffer提供存储空间.
```c
#define FOO_CTX_SIZE /*...*/
#define FOO_CTX_ALIGNMENT /*...*/

typedef struct FooCtxStorage {
    alignas(FOO_CTX_ALIGNMENT) unsigned char buf[FOO_CTX_SIZE];
} FooCtxStorage;

FooCtx* foo_init_storage(FooCtxStorage* buf);
void foo_finish(FooCtx* buf); // cleans up additional resources held by FooCtx, no need to free memory.

int foo_some_function(FooCtx*);
```
显然, `FooCtxStorage::buf`需要能够装下`FooCtx`, 并满足内存对齐要求.
这里用了`alignas`, 如果没有C11, 也可以把`FooCtxStorage`换成一个union, 加入一个member, 用于确保`buf`的对齐.

`foo_init_storage`用于在已有的一段缓存上创建一个`FooCtx`, 并且返回所创建对象的指针.
其它函数只接收`FooCtx*`作为参数.
`foo_finish`用于释放`FooCtx`可能管理的资源, 但不需要释放它本身占用的存储, 因为存储是由外部提供.

这个方法使用时需要先在栈上声明一个`FooCtxStorage`(当然实际上放在哪里都行), 然后用一个初始化函数获得创建好的`FooCtx*`, 有两个变量, 需要一致.
而且, 如果对象被move, 还需要保持指针和storage对象同步. (当然, 可以提供专门的move/copy函数避免错误)

## 编译期确保大小和对齐
在对应的`foo.c`的TU里, 我们可以用一些静态assert确保`FOO_CTX_SIZE`和`FOO_CTX_ALIGNMENT`确实符合`FooCtx`的定义.

## Strict aliasing
很遗憾, 实际上这个方法仍然是违反strict aliasing rule的, 因为`buf`的实际类型是`char`数组, 且永远不会改变, 把一个`FooCtx`给`memcpy`进去也没用.
`foo_init_storage`必定会把`buf`这片区域当做`FooCtx`来访问, 就算传`void*`也没区别, 因此这实际是UB.

(注意, 用`char*`访问`FooCtx`可以, 但反过来不行)

# Inline PIMPL
A.K.A. fast PIMPL, 也就是不需要经过动态内存分配的PIMPL.
原理上和上面的方法类似, 但由于C++提供了**placement new**和`std::launder`, 可以非UB地实现.
(这里假设读者已经知晓PIMPL的一般用法.)

```c++
// foo.h
class Foo {
public:
    Foo();
    ~Foo() noexcept;
    // 有需要的话, 声明(或delete)copy和move

    // 其它method...

private:
    struct Impl;
    static constexpr int impl_alignment = /* write it down, or just use max_align_t if no extended alignment */;
    static constexpr int impl_size = /* .. */
    alignas(impl_alignment) unsigned char buf[impl_size];

    //Impl* handle; //这一项可以不要, 省掉一点空间, 换成下面的函数.
    Impl* get_impl() noexcept;
}

// foo.cpp中的内容

struct Foo::Impl {
    //...
}

static_assert(Foo::impl_size >= sizeof(Foo::Impl));
static_assert(Foo::impl_alignment >= alignof(Foo::Impl));

Foo::Foo() {
    // 如果有handle, 就这时候记录它.
    // handle = new(buf) Impl{};
    // 没有的话, 直接new, 丢掉指针, 一会用std::launder找回来.
    new(&buf[0]) Impl{};
}

Foo::Impl* Foo::get_impl() noexcept {
    return std::launder(reinterpret_cast<Impl*>(&buf[0]));
}

Foo::~Foo() noexcept {
    get_impl()->~Impl();
}
```

创建`Foo`时, 我们用placement new在静态的`buf`上创建`Impl`对象. 这样会隐式地结束原本`unsigned char`的生命周期,
并开始`Impl`对象的生命周期.
placement new返回的指针可以记录着, 以后就可以通过它访问`Impl`了.

但这里为了省掉这个额外的`Foo::handle`成员变量, 我们额外定义一个`Foo::get_impl()`函数, 帮我们安全地从`buf`获取`Impl`对象.
这里需要用到`std::launder`, 否则从原本的`buf`指针直接转换得到的`Impl*`指针并不能用于安全地访问新构建的`Impl`对象.

(也就是说, 如果没有`std::launder`可用, 就需要用一个成员变量记录placement new产生的指针.)

析构函数中, 需要我们显式地调用`Impl`上的析构函数.

## ABI稳定性
需要注意, 这种做法也部分舍弃了普通PIMPL带来的ABI稳定性的好处, 因为`Foo::buf`的大小和对齐可能随着`Foo::Impl`变化.

一种workaround是把`impl_alignment`调整得足够大, 并且给`impl_size`预留足够的空间, 这样`Impl`增长并不总需要改变`buf`.
源文件里有`static_asset`检查, 所以不用担心大小或对齐不够的情况.
