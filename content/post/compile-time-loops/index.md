---
title: Compile time loops in C++
date: 2023-10-09
math: true

categories:
    - Programming
tags:
    - c++
    - template metaprogramming
---

A compile time loop is a loop that is guaranteed to be expanded at compile time.
In contrast, a regular runtime loop typically keeps track of a loop condition at runtime,
and checks the condition at every iteration to determine whether to exit the loop.

An optimizing compiler is often able to expand some runtime loops in-place, but not always. We'll introduce several ways of composing compile-time loops, using template metaprogramming.

# Generic formulation

Here's the problem: Given a compile time integer sequence of [0, ..., N), apply a function (template) over all `N` integers. The integers should be available to the function as compile time constants.


# Template recursion

A common practice is to recurse through all the indices one by one. A non-generic example would be:
```cpp
template<unsigned N>
constexpr void for_something()
{
    if constexpr (N >= 1) {
        for_something<N-1>();
        // do something with N-1
        std::cout << N-1 << std::endl;
    }
}
```
This function will loop from `0` to `N-1`. The `if constexpr` can be replaced with a template specialization for `N=0`.

## The magic `constexpr` index
To make this generic, we could use a template template parameter for a callable, or the way I prefer: a class template parameter which has templated `operator()`, e.g. a generic lambda.
```cpp
template<unsigned N, typename F>
constexpr void for_something(F&& f)
{
    if constexpr (N >= 1) {
        for_something<N-1>(std::forward<F>(f)); // may be you shouldn't forward
        std::forward<F>(f)(std::integral_constant<decltype(N), N-1>{});
    }
}
```
Use it with a generic lambda:
```cpp
for_something<3>([](auto i){
    std::cout << i << std::endl;
    // the i.value is constexpr!
    // i even has implicit conversion to its value!
    if constexpr (i.value == 9999) {
        static_assert(false, "this should've been removed at compile time");
        // some dependent false value may be required to make this compile.
    }
});
```
`i` can be used as a compile-time integer. The trick here is to pass the compile time index as an `std::integral_constant` tag value, whose type (which encodes the integer) is then received by the generic lambda.
Inside the lambda, `i.value` is a static, constexpr method of the deduced type `decltype(i)`.

## std::interger_sequence

With the help of `std::make_integer_sequence`, we could easily generate a parameter pack of integer sequence, and recurse over the pack. The idea is basically the same, you just extract the first element in the pack, and recurse the rest.

## Drawbacks

Template recursion can generate huge amount of instantiations of the same function/class template at compile time, so they are not optimal for compile time performance.

As for runtime performance, as long as everything is properly inlined without jumping through all the indirections and recursions, it should be the same as unrolling the loop manually.

# Pack expansion and fold expressions

## Pack expansion hack

A common workaround before C++17's fold expressions is to expand the pack into an array initializer:
```cpp
template<std::size_t ...Is>
constexpr void for_impl(std::index_sequence<Is...>)
{
    bool _arr[] { (some_expression(Is), 0)... };
    (void)_arr;
}
```
**Note**: Here, the order of evaluation inside the braces is guaranteed, unlike function arguments.

## Fold expressions

Fold expressions provide a much nicer way of doing pack iteration and (when combined with `std::integer_sequence`) compile time loop.
Here's a class template which applies a generic function over an integer sequence, expanded at compile time:
```c++
template<typename>
struct ForInts;

template<typename IntT, IntT ...Is>
struct ForInts<std::integer_sequence<IntT, Is...>> {
    template<typename F>
    static constexpr void apply(F&& f) {
        (void(f(std::integral_constant<IntT, Is>{})), ...);
    }
};

template<auto N>
using ForSeq = ForInts<std::make_integer_sequence<decltype(N), N>>;
```
The `ForSeq` alias template generates an integer sequence as the template parameter for `ForInts`.
The `ForInts` class template's `apply()` method applies a given function object over the indices,
by comma-folding over the function expression. It employs the same `std::integral_constant` trick.
The `void` cast is there to deal with overloaded `operator,`.

Example usage:
```c++
std::ostringstream ss;
ForSeq<10>::apply([&ss](auto i) {
    if constexpr (i.value == 0) ss << "Start!\n";
    ss << "loop: " << i.value << ' ';
});
```

This is a much more natural way of implementing and writing compile-time loops.

Using `std::make_integer_sequence` often involves an indirection to a separate `impl` function or class, which catches the actual integer list.
With explicit lambda template in C++20, we could replace the `impl` function with an inner lambda:
```cpp
template<std::size_t N>
constexpr void do_something() {
    []<template std::size_t ...Is>(std::index_sequence<Is...>) {
        // fold over Is...
    }(std::make_index_sequence<N>{}); // IIFE
}
```

## Runtime break from the loop (short-circuiting)
What if we want to break half-way from the compile time loop, depending on some runtime condition?
It's trivial to achieve with logical folds, since the `&&` and `||` operators in C++ has short-circuiting behavior.

A partial demo:
```cpp
template <std::size_t ...Is>
constexpr void fn(std::index_sequence<Is...>)
{
    ([]() -> bool {
        // do something, return true or false
        // return true if the loop should be continued.
        // return false if not.
    }() && ...);
}
```
The above `fn()` function will evaluate all folded expressions (here, the expression is an IIFE returning either `true` or `false`). If at some point the IIFE returns `false`, all remaining expressions will be skipped.

# Sidenote: Implementation of `std::make_integer_sequence`

Many compile-time loop patterns ultimately boils down to `std::make_integer_sequence` (or equivalent).

A naive implementation is one-by-one template recursion. Something smarter would be $\log_2(N)$ recursion, which obviously works by dividing by half every iteration.

Since `std::make_integer_sequence` is ubiquitous in template metaprogramming, some compilers implement
it with magic intrinsics.

* MSVC and Clang: `__make_integer_seq`
* GCC: `__integer_pack`
