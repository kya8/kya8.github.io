---
title: "C++实现系列: Iterator adaptor"
date: 2024-01-29
tags:
    - c++
    - iterator
    - template metaprogramming
draft: false
---

# 前言

C++20开始, 标准库中提供了`ranges`模块, 其核心就是各种iterator adaptor, 通过它们的灵活组合, 变换, 能够以简练清晰的代码实现相当复杂的遍历逻辑, 并且没有额外性能开销. 与它们对应的就是Rust的`std::iter`, 或是Python的`itertools`.

这篇文章会通过对几个iterator adaptor / range的简单实现, 浅窥iterator adaptor的背后思想和一些实现细节.

# C++ Iterator和Range
C++中的Iterator相当灵活, 但也比较繁琐. 简单来说, 我们把能够用于range-for的对象称为是一个range, 也就是说能从这个对象上调用`begin()`, `end()`等函数, 获取一对迭代器, 然后用这组迭代器进行一一遍历. 而具体的迭代逻辑, 就由`operator++, operator*`等实现.
`end()`获取的iterator, 仅是一个sentinel值, 用于区分迭代是否应当结束.

这个从range对象取出一对iterator的逻辑, 刚接触时会有点别扭. 我们若是要自己实现一个range(不管是对其它range的一个adaptor还是自己生成), 就需要实现一个range类, 包含遍历所需的所有信息, 并能够生成`begin`和`end`两个起止的iterator. 这两个iterator一般也需要一个单独的类, 记录迭代时的状态信息等.

# enumerate
我们从比较简单的enumerate开始. 这个iterator adaptor会在输入的iterator的基础上, 返回一个`std::pair`的range, 包含每个元素的序号和内容(引用原range的目标).

这里直接给出代码:
```c++
namespace details {

template <typename T, typename = std::void_t<decltype(std::begin(std::declval<T&>())), decltype(std::end(std::declval<T&>()))>>
constexpr auto enumerate(T&& t)
{
    using IterT = decltype(std::begin(std::declval<T&>()));
    using IterTraits = std::iterator_traits<IterT>;
    struct EnumerateIterator {
        typename IterTraits::difference_type index_;
        IterT it_;

        auto operator*() const { // This returns a temporary value, not by reference.
            return std::pair<decltype(index_), decltype(*it_)>{index_, *it_};
        }
        bool operator==(const EnumerateIterator& rhs) const {
            return it_ == rhs.it_;
        }
        bool operator!=(const EnumerateIterator& rhs) const {
            return it_ != rhs.it_;
        }
        auto& operator++() {
            ++index_;
            ++it_;
            return *this;
        }
    };

    struct EnumerateIterableProxy {
        T t_; // rvalue t will be moved in.
              // Cannot use T&& here, since "a temporary bound to a reference parameter in a function call exists until the end of the full expression containing that function call",
              // so enumerate(rvalue_here()) won't work as expected.

        EnumerateIterator begin() {
            return {0, std::begin(t_)};
        }
        EnumerateIterator end() {
            return {-1, std::end(t_)};
        }
    };
    return EnumerateIterableProxy{std::forward<T>(t)};
}

} // namespace details

template <typename T>
constexpr auto enumerate(T&& t)
{
    return details::enumerate(std::forward<T>(t));
}
```
这里的`enumerate`是用函数的形式实现, 返回了函数内部所定义的一个类`EnumerateIterableProxy`.
在函数外部, 用户不能创建这个类, 但仍可以正常访问这个类的成员.

我们把外部的`enumerate`转发到了`details`命名空间下的一个内部实现函数, 主要是为了让外部函数的模版参数不带有额外参数.
`details::enumerate`在模版参数上, 通过检查参数类型`T`是否具有需要的`begin`和`end`, 限制了输入`T`的类型. (这里用了C++17的`std::void_t`)

`std::declval<T&>`中加了一个左值引用: 因为如果函数接收的是右值, 那么`std::declval<T>()`会返回一个右值引用, 其本身是一个右值. 而`std::begin`不允许右值参数.
因此我们强制确保它是左值.

最终返回的`EnumerateIterableProxy`对象是直接由输入的`T&& t`构建, 不难看出, 经过完美转发后, `EnumerateIterableProxy`中的成员`t_`要么直接通过move, 将`t`转移至`EnumerateIterableProxy`内部的值类型, 要么成为一个对`t`的左值引用. 前者是接收右值的情况, 后者是接收左值. 没错, 如果是左值我们就引用它, 如果是右值我们就直接把它搬进来, 把所有权交给`EnumerateIterableProxy`.
`EnumerateIterableProxy`引用或占有`t`后, 由它产生的iterator就能安全的引用`t`中的元素.

具体的enumerate逻辑位于`EnumerateIterator`之中. `EnumerateIterator`持有两个成员: 序号和当前的原range的iterator. 递增时, 序号和原iterator一起递增. 比较操作就是比较原iterator. 由于`EnumerateIterableProxy`给出的end iterator持有原先的end, 因此`EnumerateIterator`会随着原range一起终止.

`EnumerateIterator`的解引用操作是把序号和对相应元素的引用, 装进`pair`里返回. 注意这里是**按值**返回, 和常见的iterator的`operator*()`按引用返回不同.
这里给出[cppreference](https://en.cppreference.com/w/cpp/named_req/InputIterator)上的一段说明:
>For an input iterator `X` that is not a _LegacyForwardIterator_, `std::iterator_traits<X>::reference` does not have to be a reference type: dereferencing an input iterator may return a proxy object or `std::iterator_traits<X>::value_type` itself by value (as in the case of `std::istreambuf_iterator`).
对于仅满足Input Iterator的iterator, `operator*()`可以按值返回.

## iterator_traits
这里利用了`std::iterator_traits`, 主要是为了能自动支持普通数组, 例如string literal.

## 使用例
```c++
for (auto&& [i, c] : enumerate("Hello!")) {
    std::cout << i << ": " << (c? c : '%') << '\n'; // % indicates NUL
}
```
运行结果:
```
0: H
1: e
2: l
3: l
4: o
5: !
6: %
```

# zip
zip的作用是, 输入**任意**多个range, 并将它们合并遍历, 即每次迭代所有的range. 直到某个主range终止, 或者其中任意一个range终止.
类似于Python的zip, 以及Rust的zip.
不过Rust由于缺少Variadic Generics, 标准库中的zip只能接收2个输入. 如果要多个输入, 需要用宏.

我们实现的版本中, 会把第一个range作为主range, 以它的终止作为终止. 也就是要求其它Range需要至少和它一样长.
此外, 这次我们改用`begin`和`end`对来接收range. 没有什么本质区别.
```cpp
template <typename Iter0, typename ...Iters>
constexpr auto zip(Iter0 begin0, Iter0 end0, Iters ...Begins)
{
    struct ZipIterator {
        Iter0 iter0_; // the pivot
        std::tuple<Iters...> iters_;
        auto operator*() const {
            return std::apply([this](auto&& ...iters) {
                return std::tuple<decltype(*begin0), decltype(*Begins)...>{*iter0_, (*iters)...};
            },
            iters_);
        }
        bool operator==(const ZipIterator& rhs) const {
            return iter0_ == rhs.iter0_; // only the pivot is compared!
                                         // Ideally, all iters should be compared, except when iter0 has reached end.
        }
        bool operator!=(const ZipIterator& rhs) const {
            return iter0_ != rhs.iter0_;
        }
        auto& operator++() {
            ++iter0_;
            std::apply([](auto&& ...iters) { ((++iters),...); }, iters_);
            return *this;
        }
        // These are made-up. You should probably use something more sensible.
        using difference_type = typename std::iterator_traits<Iter0>::difference_type;
        using value_type      = std::tuple<decltype(*begin0), decltype(*Begins)...>;
        using pointer         = void;
        using reference       = const value_type&;
        using iterator_category = std::input_iterator_tag;
    };
    struct ZipProxy {
        Iter0 begin0_, end0_;
        std::tuple<Iters...> iters_;
    
        ZipIterator begin() {
            return {begin0_, iters_};
        }
        ZipIterator end() {
            return {end0_, iters_};
        }
    };
    return ZipProxy{begin0, end0, {Begins...}};
}
```
与前面的`enumerate`类似, 我们仍然是返回函数内部定义的`ZipProxy`类. 我们需要储存主range的`begin`和`end`, 以及所有其它range的`begin`.

为了储存其它所有`begin` iterator (即`Iters ...Begins`这个可变参数), 我们直接使用了`std::tuple`.
这里没必要去自己实现, 因为自己实现相当于重新发明一遍tuple.

从`ZipProxy`产生的`ZipIterator`, 会被赋予主range的iterator, 以及其它range的iterator.
`ZipIterator`的比较操作仅取决于主iterator.
递增就是把每个iterator递增. 而解引用操作, 返回了一个tuple, 装着所有当前元素的引用.

为了实现递增和解引用, 我们直接对tuple使用`std::apply`, 把装着iterators的tuple展开成函数参数, 在一个lambda函数中对这些iterator进行遍历操作或展开. 其中递增操作用了一个comma fold.

## 使用例
```c++
const std::vector v{2,3,4};
const char* s = "abc";
for (const auto [vv, ss] :zip(v.cbegin(), v.cend(), s)) {
    std::cout << vv << " | " << ss << '\n';
}
```
运行结果:
```
2 | a
3 | b
4 | c
```

# NdIndex
即[numpy.ndindex](https://numpy.org/doc/stable/reference/generated/numpy.ndindex.html). 严格来说它不是iterator adaptor, 而是一个每次生成新元素的range.

它也是笛卡尔积[`ranges::views::cartesian_product`](https://en.cppreference.com/w/cpp/ranges/cartesian_product_view)的一种特殊情况, 但实现本质上和`cartesian_product`没有区别.

这次我们使用一个类来表示`NdIndex`. 当然, 用函数产生range也是完全可行的. 这里仅是为了演示.
```c++
template <std::size_t Dim>
class NdIndex {
private:
    std::array<std::size_t, Dim> sizes_;
public:
    static constexpr auto ndim = Dim;

    template <typename ...Ts>
    constexpr NdIndex(Ts ...Args) noexcept : sizes_{std::size_t(Args)...} {}

    class Iter {
        friend class NdIndex;
    private:
        std::array<std::size_t, Dim> idx_{};
        std::array<std::size_t, Dim> sizes_;

        // helper for loop unrolling
        template <std::size_t ...Is>
        constexpr void increment(std::index_sequence<Is...>) noexcept {
            ([this] {
                constexpr auto i = Dim - 1 - Is;
                if constexpr (i == 0) {
                    ++idx_[0];
                    return true;
                }
                else {
                    if (++idx_[i] < sizes_[i]) return false;
                    idx_[i] = 0;
                    return true;
                }
            }() && ...); // short-circuit fold
        }
    public:
        constexpr auto& operator*() const noexcept { return idx_; }
        constexpr auto& operator++() noexcept {
            increment(std::make_index_sequence<Dim>{});
            return *this;
        }
        constexpr bool operator==(const Iter& rhs) const noexcept {
            return idx_ == rhs.idx_; // sizes?
        }
        constexpr bool operator!=(const Iter& rhs) const noexcept {
            return !(*this == rhs);
        }
    public: // traits
        using difference_type = std::make_signed_t<std::size_t>;
        using value_type      = decltype(idx_);
        using pointer         = void;
        using reference       = const value_type&;
        using iterator_category = std::input_iterator_tag;
    };

    constexpr auto begin() const noexcept {
        Iter it;
        it.sizes_ = sizes_;
        return it;
    }
    constexpr auto end() const noexcept {
        Iter it;
        it.idx_[0] = sizes_[0];
        it.sizes_ = sizes_;
        return it;
    }
};
// CTAD guide
template<typename ...Ts>
NdIndex(Ts ...Args) -> NdIndex<sizeof...(Ts)>;
```
`NdIndex`的唯一模板参数是它的维度个数`Dim`. 使用者构建`NdIndex`对象时, 参数为各个维度上的长度.
通过代码末尾处的一个CTAD提示, 我们告诉编译器, 构建函数的参数个数, 就是模板参数`Dim`. 这里的CTAD guide是必须的.

显然, `NdIndex`类需要保存每个维度上的最大长度. 而`NdIndex`给出的iterator, 即它的子类`NdIndex::Iter`, 则需要当前的每个维度的下标, 以及最大长度.
而iterator的比较操作, 就是逐个比较它们保存的所有下标. 此外, end iterator是通过最高维度的下标达到长度值来表示, 到此就应终止迭代.

为了方便, 我们直接用`std::array`, 自动提供赋值, 比较等操作. 同时作为`operator*()`的返回值, 它也是一个*tuple-like*的类, 能够被structured binding展开.

## 递增
`NdIndex`的主要逻辑实现在于它的递增函数. 为了编译期展开循环, 我们把`operator++()`用`std::index_sequence`做了一次转接, 以便在`increment`函数中按每个维度展开.

`increment`函数的每次执行, 是把一个IIFE表达式 (立即执行的lambda函数, 这里的目的是为了把多行代码就地变成一个返回`true`或`false`的expression)
按照编译期确定的`0, 1, ..., Dim-1`几个维度展开, 展开后用`&&`算符实现短路执行: 即当某个维度的IIFE返回false后, 剩下维度的操作就不会被执行.

注意IIFE按维度执行的顺序是从低维开始. 这个IIFE表达式的意义就比较明晰了: 
* 如果是最高的维度(这是一个编译期判断), 就直接把这个维度加1, 加到最大后迭代就会停止. 返回`true`或`false`无所谓, 因为这是逻辑和的最后一个表达式.
* 如果是其它维度, 那么
    * 如果加1后还未达到长度值, 就直接返回`false`, 结束本次`increment`.
    * 如果加1后达到了长度值, 就进位: 把这一维度设为0, 然后返回`true`, 继续更高维度.

> Tip: 用`&&`算符的fold expression, 实现可在运行时跳出的编译期循环! 有点绕)

## 改进
1. 这里的`NdIndex`的下标类型是固定的`size_t`, 不难把它改成以整数类型为模板参数的类模板.
2. `NdIndex`存在一个potential的bug, 即处理各维度的长度之一(除了最高维度)是0的情况. 当前的行为相当于直接跳过了这个维度, 它一直是0. 但我认为正确的行为应当是使整个range为空.
如果要实现这个行为, 只需要和end比较时, 比较每个维度是否达到了最大值.

# 总结

上面的三个实现都是出于PoC展示的目的, 并不算严密, 如果要作为一个正式的库发布, 还有不少需要修改的地方, 但可以作为iterator adaptor的概念展示,
读者看完后, 应该不难在C++20之前的版本自己实现其它的adaptor.

## adaptor组合

组合起来用:
```c++
#include <fmt/ranges.h>

constexpr auto nd1 = NdIndex{2,3};
constexpr auto nd2 = NdIndex{3,2};
for (const auto& a : enumerate(zip(nd1.begin(), nd1.end(), nd2.begin()))) {
    fmt::println("{}", a);
}
```
由于我们的`zip`函数是接收begin/end, 因此只能把需要被zip的range单独声明在前.

运行结果:
```
(0, ([0, 0], [0, 0]))
(1, ([0, 1], [0, 1]))
(2, ([0, 2], [1, 0]))
(3, ([1, 0], [1, 1]))
(4, ([1, 1], [2, 0]))
(5, ([1, 2], [2, 1]))
```
