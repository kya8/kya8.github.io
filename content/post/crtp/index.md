---
title: CRTP的各种姿势
date: 2025-01-10
draft: false

categories:
tags:
    - c++
    - template metaprogramming
---

# CRTP
CRTP是啥就不多介绍了, 总之是C++中常见的一种静态多态或Mixin的实现方式. 它的用法比较灵活, 但最常见的还是用于做Mixin和静态多态界面.

在C++标准库中, 也能看到CRTP的使用:
* [std::enable_shared_from_this](https://en.cppreference.com/w/cpp/memory/enable_shared_from_this.html)
* [std::ranges::view_interface](https://en.cppreference.com/w/cpp/ranges/view_interface.html)

## Mixin
这里Mixin的意思就是, 对已经实现了满足一定条件的一组方法的类, 自动实现另一组方法. 类似于Rust里的`impl<T: TraitA> TraitB for T`

简单的Mixin例子:
```c++
template<typename Derived>
struct Greet {
    void greet() const {
        std::cout << "Hello, my name is " << static_cast<const Derived*>(this)->name() << '\n';
    }
};

struct Bob : Greet<Bob> {
    std::string_view name() const {
        return "Bob";
    }
};
```

对于任何实现了`name()`方法的类, 只要继承`Greet`, 就能直接获得`greet()`方法, 而`greet()`中会用到`name()`. CRTP的核心思想其实就是: 通过静态的模板参数, `Greet`这个Mixin类可获取到实际需要Mixin的子类, 并调用其方法.

## 静态多态界面

例子:
```c++
template<typename Derived>
struct Interface {
    decltype(auto) interface() {
        return static_cast<Derived*>(this)->implementation();
    }
};

struct MyImpl {
    void implementation() {
        std::cout << "Hello!\n";
    }
};

template<typename Derived>
void use_interface(Interface<Derived>& a) {
    a.implementation();
}
```

这里, `Interface<Derived>`作为静态界面的基类, 任何实现了`implementation()`方法的类继承于它, 就能通过`Interface`的引用访问`interface()`, 并静态派发到具体的`implementation()`上.

# C++23 deducing this

这里有必要介绍下C++23的*deducing this*新特性所带来的新CRTP写法. 这种写法下, 甚至都不能叫Curiously Recurring Template Pattern了.

```c++
struct Greet_Cpp23 {
    template<typename Self>
    void greet(this Self&& self) {
        std::println("Hello, my name is {}.", std::forward<Self>(self).name());
    }
};

struct Alice : Greet_Cpp23 {
    const auto& name() const {
        return "Alice";
    }
};
```

不难猜到, 如果`Greet_Cpp23`被继承, 在子类上调用`greet()`时, `Self`就会被推导为子类. 因此, 我们不再需要以子类作为模板参数了. 这种写法无疑简化了很多.

不过, 如果要用这种写法实现静态多态界面, 就不能用`Greet_Cpp23`的引用了, 如:
```c++
void use_greet(Greet_Cpp23& a) {
    a.greet();
}
```
这显然不会work, 因为`a`的类型只有`Greet_Cpp23`, `Self`不会被推导为真正的子类类型, 如`Alice`.

但是, 我们可以这样写, 达到类似的约束效果:
```c++
template<std::derived_from<Greet_Cpp23> T>
void use_greet(const T& a)
{
    a.greet();
}
```
这样就是一个不通过基类引用, 仅通过模板参数约束的泛型界面了.

# 防止错误使用

CRTP(传统写法)如果使用不慎, 会导致`static_cast<Derived*>(this)`非法. 也就是, 如果CRTP基类未以自己的一个子类为模板参数, 那么这个cast行为是危险的.

(对于Deducing this写法, 不存在cast, 因此不存在该问题.)

一般来说, 有两种方法防止错误使用CRTP:

## Private constructor + friend
把CRTP基类的构造函数设置为private, 并给模板参数里的`Derived`类friend. 这样, 就只有当这个`Derived`类继承它时, 才可能将它实例化为对象.
如果写错模板参数, 或者单独使用这个基类, 会发现无法构造对象.

以上面的`Greet`为例:
```c++
template<typename Derived>
struct Greet {
    void greet() const {
        std::cout << "Hello, my name is " << static_cast<const Derived*>(this)->name() << '\n';
    }
private:
// common CRTP trick to prevent misuse
    Greet() = default;
    friend Derived;
};
```

这种方法有两个需要注意的点:
* 子类如果是个aggregate类, 将不能进行aggregate初始化, 因为aggregate初始化不能访问基类的private构造函数.
对于上面的`Bob`, 应改成:
```c++
struct Bob : Greet<Bob> {
    std::string_view name() const {
        return "Bob";
    }
    // Add a defaulted ctor, so Bob is not an aggregate.
    // Otherwise Bob{} initialization won't work if Greet has private ctor.
    Bob() = default;
};
```

* 如果这样的CRTP基类被继续继承为另一个CRTP基类, 那么最后的实际类仍无法实例化, 因为第二个CRTP基类并不是friend, 无法访问基类的private构建函数. 这个没有很好的解决办法. 也就是说, 这样的CRTP基类必须直接被实际类继承.

## static_assert is_base_of
初学者很可能会想到, 我在CRTP基类模板里加个static_assert不就行了? 但是加在class scope里是不work的, 因为`Derived`类在作为模板参数时, 它的定义还未完成, 属于incomplete type. 因此, 没法在CRTP基类的class scope里写static_assert检查`Derived`是否继承了该类.

但是, 我们仍然可以在方法内, 或者构建函数内部检查:
```c++
template<typename Derived>
struct Greet {
    void greet() const {
        std::cout << "Hello, my name is " << static_cast<const Derived*>(this)->name() << '\n';
    }
    Greet() noexcept {
        static_assert(std::is_base_of_v<Greet, Derived>);
    }
};
```

不过, 这个方法仍然有个缺陷, 那就是, 它没法防止如下的问题:
```c++
struct Alice : Greet<Alice> {/* ... */};
struct Bob : Greet<Alice> {/* ... */};
```
第二行的模板参数写错了, 但是我们的static_assert并不会报错, 因为`Alice`确实是`Greet<Alice>`的子类!

---

# (未完待续)
