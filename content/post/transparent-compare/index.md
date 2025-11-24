---
title: Transparent Compare
date: 2024-11-26
lastmod: 2026-04-26
tags:
    - c++
---

# Transparent compare & heterogeneous lookup

什么是Transparent compare? 简单来说就是允许两个不同类型之间的比较.
基于此, 在需要比较操作的容器中(如`std::map`)进行非同类型查找就是heterogeneous lookup.

例如我们有一个`std::map<std::string, int>`, 它的`find()`方法有两种:
1. `find(const Key& key)`, 对于我们的情况, 参数就必须是一个`std::string`.
2. `find(const K& x)`, `K`是任意类型. (Available since C++14)

注意, 第二个重载是否进入重载集**仅取决于**`std::map`的模板实参中, 所提供的Comparator类型是否是[transparent](https://en.cppreference.com/cpp/functional#Transparent_function_objects).
这个要求简单说就是能够提供非同类型比较, 以及提供一个成员typedef作为tag. `std::less<void>`就符合这个要求.

也就是说, 如果我们用`std::less<void>`作为map的Comparator类型, 那么`find()`的第二个重载就始终有效,
并不取决于实际提供参数的类型`K`.
实际上, 除了类型恰好是`Key`的参数, 其它参数都会匹配到第二个任意类型的重载, 并尝试使用`std::less<void>`进行比较操作. 如果无法比较, 编译就出错了.

对于默认的Comparator, 即`std::less<Key>`, 它只能进行两个相同类型参数的相互比较, 因此只能使用第一个重载,
查找的key要么就是`Key`类型, 要么被隐式转化为`Key`类型.

# 防止产生临时变量
如果没有transparent compare, 一个最常见的问题就是, 我们的`std::map<std::string, int>`在进行任何需要和内部key比较的操作时, 都需要参数是`std::string`,
这就造成了不必要的拷贝和内存分配. 很多时候, 我们查找用的key是一个`std::string_view`, 或者一个string literal(也就是`const char(&)[]`),
转成`std::string`再进行比较, 属实有些浪费.

显然, 我们可以用大于小于号(或者C++20后的<=>算符)直接比较`std::string`和`std::string_view`, 因为前者能够隐式转化为后者, 从而转为两个`string_view`之间的比较.
`std::less<void>`也能pick到这些, 因此也能够进行这两种类型的比较.

对于我们的例子, 解决方法就非常简单了, 只需要将模板参数改为: `std::map<std::string, int, std::less<void>>`,
就自动获得了heterogeneous lookup, 可以用`string_view`或者string literal进行查找. (后者是因为`std::string`定义了和`char*`字符串的比较算符)

## 其它method
标准库容器中, 其它可能利用到heterogeneous lookup的若干方法仅在较高的C++标准版本中才提供heterogeneous的重载,
例如`std::map`中的`try_emplace`, `operator[]`, `erase`等.

# 更复杂的例子
我自己写的一个库中, 使用了一个嵌套的map结构, 其中定义了一种[复合的key类型](https://github.com/kya8/slate/blob/6db8cd2bebff6aab0e9e48176c4a0f97babc0c34/include/slate/varmap.hpp#L30):
```c++
using Key = std::variant<KeyId, std::string>;
using Map = std::map<Key, Value, std::less<void>>;
```
其中`KeyId`是一个枚举类, 包含了一些常用的key. 把key定义成一个union/variant的目的, 是为了使用well-known的固定tag作为key的同时, 也允许使用任意的字符串, 保证灵活性.

显然, 如果不使用`std::less<void>`来启用heterogeneous lookup, 那么对`Map`的查找都必须先将查找的键值转为`Key`类型, 即`std::variant<KeyId, std::string>`. 对于任意字符串的情况, 就需要临时创建一个`std::string`.

这种情况下, 我们就需要手动提供比较算符, 使得`Key`这个variant类型能够直接和`string_view`或其它string类型比较.
对于装着`KeyId`的情况, 也可以自定义比较算符. 不过这种情况下, 从`KeyId`构建临时`Key`的开销相对较小.

自定义的比较算符如下:
```c++
inline constexpr std::strong_ordering operator<=>(const Key& key, std::string_view sv) noexcept {
    return std::visit([&sv]<typename T>(const T& val) {
        if constexpr (std::is_convertible_v<T, std::string_view>) {
            return std::string_view(val) <=> sv; // Convert val to string_view to avoid recursion into this function.
        } else {
            return 0 <=> 1; // following std::variant's operator<=>
        }
    }, key);
}

// A generic comparison operator for std::variant that allows comparing
// the contained value directly with a type T, if it is one of the alternatives.
// Note that this does not allow implicit conversions from T.
template<typename ...Ts, typename T>
requires (std::is_same_v<T, Ts> || ...)
constexpr auto operator<=>(const std::variant<Ts...>& var, const T& rhs)
{
    return std::visit(
        [&]<typename V>(const V& val) ->
        std::common_comparison_category_t<std::compare_three_way_result_t<std::size_t>, std::compare_three_way_result_t<T>> {
            if constexpr (std::is_same_v<V, T>) {
                return val <=> rhs;
            } else {
                return detail::get_index<V, Ts...>() <=> detail::get_index<T, Ts...>();
            }
        }
    , var);
}
```
这里我们简便起见, 就直接用了C++20的太空梭算符.
第一个定义是用于`Key`和一般string的比较, 第二个是用于`Key`和`KeyId`的比较(尽管它是泛用的).

事实上, 标准库中定义了两个同样的`std::variant`类型的比较算符(先比较它们的index, 如果index相同, 再进行同类型内的比较),
我们定义的这两个算符行为上和标准库中的定义保持一致, 只不过式子右端直接就是variant内的某个类型.

这么一来, `Map`就能以任何可隐式转化为`string_view`的类型作为查找key了, 也能直接用`KeyId`进行查找,
都不再需要临时创建一个variant.
