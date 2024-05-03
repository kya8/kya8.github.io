---
title: "C++实现系列: 环形缓存"
date: 2023-11-16
tags:
    - c++
    - container
---

# 背景
环形缓存(ring buffer)是一种简单高效的数据结构, 想必不需要过多介绍.
这篇文章主要是以实现一个环形缓存容器类为例, 讲述实现C++容器类过程中的一些注意事项和最佳实践.

已有的环形缓存实现有`boost::circular_buffer`, 提供与STL容器兼容的API.

## 实现要求及细节考虑
* 首先, 环形缓存的自身基本功能.
* 尽量提供与STL容器相同或相似的API.
* 支持用户自定义的allocator. 默认使用std::allocator. 如果是stateless的allocator, 不占用额外空间.
* ...

# 实现
```cpp
template <typename T, typename Allocator = std::allocator<T>>
class RingBuffer {
private:
    using alloc_traits = std::allocator_traits<Allocator>;
public:
    using size_type       = typename alloc_traits::size_type;
    using difference_type = typename alloc_traits::difference_type;
    using value_type      = T;
    using reference       = value_type&;
    using const_reference = const value_type&;
    using pointer         = typename alloc_traits::pointer;
    using const_pointer   = typename alloc_traits::const_pointer;
    //iterator = ...;
    using allocator_type  = Allocator;

    RingBuffer(size_type max, const allocator_type& alloc = allocator_type{})
    : alloc_and_data_{alloc, {}}, first_(0), count_(0), max_(max) {
        alloc_and_data_.second() = alloc_traits::allocate(alloc_(), max);
    }

    ~RingBuffer() noexcept {
        clear();
        alloc_traits::deallocate(alloc_(), data_(), max_);
    }

    RingBuffer(RingBuffer&& rhs) noexcept :
    alloc_and_data_{std::move(rhs.alloc_and_data_)}, // ensure moving the allocator. The pointer is copied.
    first_{rhs.first_},
    count_{rhs.count_},
    max_{rhs.max_}
    {
        rhs.alloc_and_data_.second() = nullptr;
        rhs.count_ = 0;
        rhs.max_ = 0;
        rhs.first_ = 0;
        // rhs shouldn't be used anymore
    }

    RingBuffer& operator=(RingBuffer&& rhs) noexcept {
        clear();
        alloc_traits::deallocate(alloc_(), data_(), max_);

        alloc_and_data_ = std::move(rhs.alloc_and_data_); // ensure moving the allocator
        first_ = rhs.first_;
        count_ = rhs.count_;
        max_ = rhs.max_;
        rhs.alloc_and_data_.second() = nullptr;
        rhs.count_ = 0;
        rhs.max_ = 0;
        rhs.first_ = 0;
        // rhs shouldn't be used anymore
        return *this;
    }

private:
    template<class V>
    bool push_impl(V&& value) {
        if (count_ == max_) return false;
        alloc_traits::construct(alloc_(), data_() + (first_+count_)%max_, std::forward<V>(value));
        ++count_;
        return true;
    }
    template<class V>
    void push_overwrite_impl(V&& value) {
        if (count_ == max_) pop();
        alloc_traits::construct(alloc_(), data_() + (first_+count_)%max_, std::forward<V>(value));
        ++count_;
    }

public:
    bool push(const T& value) {
        return push_impl(value);
    }
    bool push(T&& value) {
        return push_impl(std::move(value));
    }
    void push_overwrite(const T& value) {
        return push_overwrite_impl(value);
    }
    void push_overwrite(T&& value) {
        return push_overwrite_impl(std::move(value));
    }
    template <typename ...Args>
    auto& emplace(Args&& ...args) {
        if (count_ == max_) pop();
        const auto ptr = data_() + (first_+count_) % max_;
        alloc_traits::construct(alloc_(), ptr, std::forward<Args>(args)...);
        ++count_;
        return *ptr;
    }

    void pop() noexcept { // UB if empty
        alloc_traits::destroy(alloc_(), data_()+first_);
        first_ = (first_+1) % max_;
        --count_;
    }
    void pop_back() noexcept {
        alloc_traits::destroy(alloc_(), data_() + (first_+count_-1) % max_);
        --count_;
    }

    const auto& operator[](size_type i) const noexcept {
        return data_()[(first_+i) % max_];
    }
    auto& operator[](size_type i) noexcept {
        return data_()[(first_+i) % max_];
    }
    auto& at(size_type i) {
        if (i >= count_) throw std::out_of_range("RingBuffer subscription out of range");
        return (*this)[i];
    }
    auto& at(size_type i) const {
        if (i >= count_) throw std::out_of_range("RingBuffer subscription out of range");
        return (*this)[i];
    }
    auto& front() const noexcept { return (*this)[0]; }
    auto& front()       noexcept { return (*this)[0]; }
    auto& back()  const noexcept { return (*this)[count_ - 1]; }
    auto& back()        noexcept { return (*this)[count_ - 1]; }

    bool empty() const noexcept { return count_ == 0; }
    size_type size()     const noexcept { return count_; }
    size_type capacity() const noexcept { return max_; }

    void clear() noexcept {
        for (auto i=first_; i<first_ + count_; ++i) {
            alloc_traits::destroy(alloc_(), &data_()[i % max_]);
            --count_;
        }
    }

private:
    CompressPair<allocator_type, T*> alloc_and_data_;
    auto& alloc_() noexcept { return alloc_and_data_.first(); }
    auto data_() const noexcept { return alloc_and_data_.second(); }
    size_type first_;
    size_type count_;
    size_type max_;
};
```

`RingBuffer`的最大容量必须在构建时指定. 并且, 所有的动态内存分配也在构建时完成.
此外用户也可以指定自定义的Allocator.

## 空基类优化 (Empty Base Optimization)
众所周知, C++中, 一个空的struct一般情况下也要占用至少1字节的空间, 否则无法将它与前后的对象地址区分开 (不过, 对于C++20, 也可以直接使用`[[no_unique_address]]`, 省去空基类优化的技巧).
但是, 对于某个对象内的基类对象, 如果基类为空, C++编译器可以选择使用空基类优化, 即, 去除基类对象的内存表示.

这个技巧在编译器STL实现的代码中经常可以看到, 一般是为了将容器的allocator和其它成员共同表示, 省去allocator为空时的额外空间占用,
因为大部分情况下, 我们会使用默认allocator (即std::allocator), 或者仅依赖某些全局变量的allocator, 而这些allocator都是无状态的, 没有成员变量.

上面的实现中为了利用空基类优化, 使用了`CompressPair<T1, T2>`. 它的实现方法, 是通过类模版的偏特化, 分别处理`T1`为空和`T2`为空的情形.
如果`T2`为空, 那么就以`T2`为基类. 否则默认以`T1`为基类. 如此便实现了对空类的"压缩".
我写了一个简单的实现:
```c++
template <typename T1, typename T2, typename = void>
class CompressPair : protected T1 {
    T2 t2_;

public:
    template<typename U1 = T1, typename U2 = T2>
    constexpr CompressPair(U1&& x, U2&& y) : T1(std::forward<U1>(x)), t2_(std::forward<U2>(y)) {}

    constexpr T1& first() noexcept        { return *this; }
    constexpr const T1& first() const noexcept  { return *this; }
    constexpr auto& second() noexcept       { return t2_; }
    constexpr auto& second() const noexcept { return t2_; }
};

template <typename T1, typename T2>
class CompressPair<T1, T2, std::enable_if_t<std::is_empty_v<T2>>> : protected T2 {
    T1 t1_;

public:
    template<typename U1 = T1, typename U2 = T2>
    constexpr CompressPair(U1&& x, U2&& y) : T2(std::forward<U2>(y)), t1_(std::forward<U1>(x)) {}

    constexpr auto& first() noexcept        { return t1_; }
    constexpr auto& first() const noexcept  { return t1_; }
    constexpr T2& second() noexcept       { return *this; }
    constexpr const T2& second() const noexcept { return *this; }
};
```
实例化`CompressPair`时, 会自动匹配合适的类模版. 此后可通过`first()`和`second()`访问储存的两个对象.
和普通的`std::pair`不同, 这里的`first()`和`second()`只能写成成员函数, 返回相应的引用.

在`RingBuffer`中, `CompressPair`用于存储allocator和data指针.

## allocator的使用
可以看到, `RingBuffer`中, 我们总是通过`allocator_traits`去调用实际的allocator, 而不是直接调用allocator的成员.
因为`allocator_traits`能够基于用户提供的allocator类, 自动提供默认实现. 否则, 用户提供的allocator需要实现所有要求的成员函数等, 即使我们只需要自定义内存分配/释放的操作.

事实上, STL中需要动态分配内存的容器都是[_`AllocatorAwareContainer`_](https://en.cppreference.com/w/cpp/named_req/AllocatorAwareContainer), 规定必须使用`allocator_traits`.

此外, C++17开始, `std::allocator`的`allocate`, `construct`成员函数已经是deprecated.

### 内存分配与对象构建
众所周知, 一般情况下C++中的对象并不是简单的一串字节, 不能假装某处已分配的内存上有一个对象, 然后去访问它 (隐式构建的对象是例外).
构建对象和分配内存是分开的两种操作, 对于新手来说, 在编写容器类的实现时很容易体会到这一点.
而一些STL容器的实现中, 将两者解耦是必须的, 例如`std::vector`.

上面的实现中, `RingBuffer`仅在构建时一次性分配所需内存. 对于默认的`std::allocator`, 其行为是用`operator new`, 分配所需长度的内存. 此时`RingBuffer`中没有任何对象.
之后, 添加对象时, 我们转发参数给`T`的构建函数, 在合适的地点*就地*构建新对象 (一般是使用placement new).
如果需要销毁某个对象, 我们通过`allocator_traits`执行析构, 结束对象的生命周期.

在整个过程中, 内存分配和释放仅在`RingBuffer`构建和析构时发生. 此外, 操作`RingBuffer`也不会移动其中的对象.

## Move操作
显而易见, `RingBuffer`是一个直接管理非RAII资源的类 (因为我们把数据用裸指针表示). 根据[0/3/5规则](https://en.cppreference.com/w/cpp/language/rule_of_three),
我们应当给它定义析构函数, 以正确释放资源. 此外, 拷贝操作理应禁止, 或者实现深拷贝. Move操作则是最好能提供.

这里我们只给`RingBuffer`实现了Move. 注意Move的手动定义会删除编译器隐式定义的Copy操作, 因此`RingBuffer`无法被Copy.

在Move操作中, 我们把右端的`CompressPair`成员直接Move过来. 这里如果allocator有自己的Move操作, 也会被利用. 其它成员直接拷贝即可.
右端被Move后, 我们需要将它置为空值, 至少得避免它析构时出现double-free之类的错误.

可以看出, 在这个实现中, 被Move后的`RingBuffer`是无法再被使用的, 最大容量是0.

## 左值与右值引用的重载
`push_impl`这个helper函数是为了给`push`使用, 减少代码重复. 众所周知, `std::vector`等容器的`push`函数有两个重载, 分别接收左值和右值.
他们的逻辑是一样的, 只是一个转发左值, 一个转发右值.
为了避免这类导致重复的情况, 我们定义一个接收forwarding reference的`push_impl`, 将两个重载统一转发给它. `push_impl`内部实现了`push`的真正逻辑, 且将参数完美转发.
由于`push_impl`仅会内部使用, 不用考虑类型不对的情况.

这是一种减少左值/右值重载代码重复的常见方法. 对于较长, 逻辑复杂的函数更有用.

# 未完成的部分
* Copy操作. 由于我们定义了Move操作, 默认的Copy操作会被删除.
Copy的实现只需把成员进行拷贝, 其中对`data_`部分特殊处理(深拷贝)即可.
当然, 能Copy的前提是, allocator是可Copy的. 对于allocator的Copy, 我们可以直接利用`std::allocator_traits<Alloc>::select_on_container_copy_construction`.
* 更改最大容量, 以及按需分配内存. 更改最大容量相对容易实现一些, Move后不能扩容的问题也能一并解决. 但按需分配内存可能会引起性能问题 (在容器被填满之前).
* Iterator. 这里没法偷懒用`T*`作为iterator类型, 因为这不是一个连续容器. 因此, 我们的iterator需要记录一些额外信息, 才能正确地从头到尾遍历.
* 其它STL容器的Requirement. 简洁起见, 这里并没有实现STL容器的所有Requirement, 有一些函数的signature也并不一致.
