---
title: IIFE in C++
description: Immediately invoked function expresions, in terms of C++ lambdas.
date: 2023-05-15
lastmod: 2024-03-02
tags:
    - c++
    - functional
---

# The const problem
In many cases, we want to define a variable that is complex to initialize. Often, this is done like the following:
```c++
int some_var;
{
	// complex code to fill the final value of some_var
}
// from now on, some_var should not be modified
```
This forbids the use of `const` on the variable without creating another intermediate variable.
Also, this cannot be used in member initialization lists in a constructor definition.

This is often the primary obstacle when enforcing const-correctness.

An obviously solution is to define a function dedicated for the complex initialization process.
And this is indeed the right direction if this subroutine is used at multiple places. Otherwise, an IIFE is preferred:
```cpp
const auto some_var = [&]()-> int {
    // do whatever, optionally use outer variables.
}();
```
The in-place lambda invocation can always be inlined, so it shouldn't affect performance. This pattern also captures surrounding states seamlessly: If we were to use a dedicated function, all states required for initialization have to be passed via function parameters.

# The ternary operator
When the initialization is only dependent on a ___runtime___ binary condition, The ternary operator also works, and is more concise than an IIFE.

However they're not equivalent, in case we'd like to make use of a compile time condition:
```cpp
constexpr bool cond = TRUE_OR_FALSE;
const auto i1 = cond? 0 : 1;
const auto i2 = [&] {
    if constexpr (cond) return "i'm a string";
    else return 42;
}
const auto i3 = cond? "i'm a string" : 42; // does not compile.
```
`i1` is an `int`; `i2` will be either `const char*` or `int` depending on `cond`.
`i3` will simply fail to compile.
That's because the ternary expression takes a runtime condition,
it's equivalent to an IIFE with plain `if else` branches.

In generic code, where the resultant type can depend on some compile time condition, use IIFE
with `if constexpr`.

# For control flow
Sometimes it's just convenient to be able to use `return` for control flow, e.g. to avoid `goto`.
If the return type is `void`, it's equivalent to a `do {...} while(0)` block.

# Turn statements into expression
As you can see, IIFE is particularly useful for converting a series of statements into an expression.
(Well, normally we'll have to wrap them inside another function. IIFE just spares us from writing a separate function for this purpose.)

IIFE provides a generic way to wrap statements into expression (more generic than using ternary expressions and comma operator),
and use it where only expressions are allowed, e.g. in comma fold expressions:
```cpp
template<class ...Ts>
void f(Ts&&... arg)
{
    ([&] {
        if constexpr (std::is_same_v<std::decay_t<Ts>, std::string>) {
            std::cout << "!!!NextWeHaveAString!!!:";
        }
        std::cout << std::forward<Ts>(arg) << ", ";
    }(), ...);
}

int main()
{
    f(1.0, 2, std::string("abc"));
}
```

## Everything is an expression!
This emulates e.g. block expressions in Rust.

# Use with `std::optional`
There's another hidden spot in the const problem: What if `some_var` should not be initialized at all?
E.g., depending on the condition, `some_var` might become meaningless, so we should not define it at all.

This is a common pattern that plagues C++ programs:
```cpp
struct SomeComplexStruct var; // default initialized...
const bool init_success = init_complex_struct(var);
```
if `init_success` is false, `var` shouldn't be there at all. However we're left with a zombie
variable that is default-initialized.

This is a much broader topic regarding RAII and class invariants, and there're many solutions/opinions to it, including using throwing constructors,
factories that returns nullable pointers/values.
For the purpose of this article, we could use an IIFE that returns an `std::optional`,
and check for null on the result.
