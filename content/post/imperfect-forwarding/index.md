---
title: (Im)perfect forwarding

date: 2023-05-30
tags:
    - c++
    - template metaprogramming
---

# Brief recap on perfect forwarding

## Reference collapsing and type deduction

There is no reference-to-reference types in C++: such type declarations would just collapse to a normal reference.
Depending on the type of the reference, the final type is a lvalue-ref if one of the refs is lvalue-ref, otherwise a rvalue-ref.

The reference collapsing rule enables deduction of _forwarding references_:
```cpp
template<typename T>
decltype(auto) func(T&& ...arg) {
    return func_inner(std::forward<T>(arg)...);
}
```
The type `T` will be either a value type w/o reference, or an lvalue-ref, such that the collapsed `T&&` type will match the _value categories_ of arguments.

## `std::forward<T>`

It effectively returns `T&&` reference to its arguments. According to the rules of the value categories of function expressions, if `T&&` is rvalue-ref, the forward expression is itself an rvalue.
Otherwise `T&&` was deduced to lvalue-ref, the forward expression is an lvalue. Therefore, `std::forward` always forward arguments as their original value categories.

__The `std::forward<T>` function template is meant to be used within deduced contexts__, i.e. where the type `T` has been deduced (either as a type template parameter `T&&`, or `auto&&`).
You should NOT let it deduce `T` via its argument, this is not how it works!

Luckily, the C++ standard specifies the signatures as `T&& forward( typename std::remove_reference<T>::type& t )` and `T&& forward( typename std::remove_reference<T>::type&& t )`.
This makes type deduction on `T` impossible, so using `std::forward` without specifying template parameter is a compile-time error.

# Imperfect forwarding

This title is somewhat clickbaity. Here we discuss some workarounds and unusual ways of using perfect forwarding.

## Use without templates

Often we want a function to accept both rvalue and lvalue arguments. Sometimes we just write 2 separate definitions for this (e.g. move and copy constructor), because they're rightfully different.
However, these two overloads can sometimes be quite duplicated. To avoid code duplication:
1. A quick-and-dirty approach is to always pass by value, then move. This incurs one additional move:
	* For the lvalue-ref overload, this means copy first, then move.
	* For rvalue-ref overload, this means move twice.

    This method is acceptable when moving is cheap.
2. Always forward to an internal function template that takes forwarding reference, and let it deduce the type.
Use `std::forward<T>` inside the function template. Since the function is only used internally, we don't need to check its arg type.

## Variadic forward in init-capture

***Note***: We're talking about capturing by copy or move, not by reference.

Since C++14, init-capture enables lambdas to capture (by value) by move, or by forwarding:
```cpp
// within some template that deduces T
[val = std::forward<T>(val)]{
    // ...
};
```
This only works one-by-one. What if we want to forward a parameter pack?

With C++20, it's simple:
```cpp
[...vals = std::forward<Ts>(vals)]{} //...
```

Before C++20, tuples are your friend: (perfectly) forward the parameter pack into a tuple, then unpack the tuple inside the lambda:
```cpp
[vals = std::make_tuple(std::forward<Ts>(vals)...)] {
    // use the vals, via std::apply:
    std::apply([](auto&& ...args){
        // use args as you want
    }, vals);
};
```
`std::make_tuple` will make a by-value tuple, initialized by forwarding the `vals`.
We unpack the tuple with `std::apply`.

## Use in non-deduced context
Although `std::forward` is designed for deduced contexts, it can sometimes be useful in non-deduced scenarios.

For example, the `R operator()(Args... args)` of `std::function<R(Args...)>` effectively does `INVOKE<R>(f, std::forward<Args>(args)...)`, where `f` is the stored callable.
We known that the `Args` is specified by the user, not deduced.
However, it still makes sense to forward `args...` with `std::forward<Args>`:
* If some `Arg` has some value type, then the `operator()` receives it by value. Inside `operator()`, we do want to pass it as an rvalue.
* If `Arg` has rvalue/lvalue type, then the additional forwarding has no effect.
