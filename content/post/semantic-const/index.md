---
title: Semantic const
description: Deep or shallow const?
date: 2023-05-19
lastmod: 2024-04-15
tags:
    - c++
---

# Semantic const

Imagine we have a class with some getter/observer method that is protected by a mutex:
```cpp
class DataWrapper {
private:
    DataType data_;
    std::mutex mtx_;
    //...
public:
    DataType getData();
    //...
};

DataType DataWrapper::getData() {
    std::scoped_lock lk(mtx_);
    return data_;
}
```
The lock operation on the mutex cannot work on a const `*this`, so the `getData()` method cannot be made `const`.
This contradicts our usual assumption of a getter method, how can an observer be non-const?

After all, the non-constness comes from the implementation of `getData()`, which requires locking the mutex, thus modifying one of the members.
However, the _interface_ being non-const is still suboptimal. What if we implement `getData()` with `std::atomic<T>::load`, which is `const`? The _interface_ declaration should be orthogonal to implementation details.

If we insist on keeping the mutex as a direct data member, the solution is to mark `mtx_` as `mutable`, so it's mutable even with const `*this`. This way, we achieve _semantic_ const, so the user of the API can hold their usual assumptions.

## const overloading

Const overloading is mostly used when returning a reference/pointer to some data within(or managed by) an object.
An example is the `operator[]` of `std::vector`, which returns const reference when the vector itself is const, or non-const reference otherwise.

For `std::vector`, the `const` overload is semantic only. Because, it's perfectly valid to return a pointer to non-const value, that is managed by a `const std::vector` ref.
However, since `std::vector` is a standard container type that manages data on its own, we want it to have value semantics: If the vector is const, so should be the elements managed by it.

### Avoid code duplication

When you starting providing const overloads, you'll likely run into lots of duplicated code, for the `const` and non-`const` overloads.

We can re-use the `const` methods in the non-`const` overload, by forcing constness on `*this`, and returning the result as mutable reference. For the former we have `std::as_const`. Below is an implementation of `as_mut` (Sorry, I took the nomenclature from Rust):
```cpp
template<typename T>
constexpr auto& as_mut(T& t) noexcept // T can be const or non-const
{
    return const_cast<std::remove_const_t<T>&>(t);
}
// This overload disallows any rvalue argument
template<typename T>
void as_mut(const T&&) = delete;
```

## Semantic non-const

Sometimes things go the opposite direction: A method modifies some internal state via a pointer/reference member, so it only requires const `*this`, e.g. in a PIMPL class. In such circumstances, we still might want to mark the method as non-const, when the object actually owns the referenced data, __since the managed data is semantically part of the object itself__.

### const views, shallow constness

However, if the object is merely a non-owning view into some referenced data, it makes more sense to use "shallow" constness, i.e. the underlying data is mutable through a const view object.
`std::span<T>` is one example of shallow constness.
