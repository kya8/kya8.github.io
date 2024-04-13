---
title: The `noexcept` move constructor
date: 2023-06-25

categories:
    - Programming
tags:
    - c++
---

In most situations, the move constructor of a class in C++ can be made to not throw exceptions, given the [Rule of 0/3/5](https://en.cppreference.com/w/cpp/language/rule_of_three) is respected. When Rule of 0 is followed, any non-trivial move operation should ultimately trace down to lowest-level resource-owning classes, and such classes are usually simple to move (e.g. copy a pointer), so their move ctors can be made `noexcept`.

The compiler-generated move-constructor for a class is implicitly `noexcept`, unless its base/member's move-constructors are potentially throwing.
If you forgot to make your own class's move ctor `noexecpt`, or use a defaulted potentially throwing move ctor (because a member's move ctor is throwing), then this property will pass on.

Why should we make the move ctors `noexcept`, besides "because we can"? Because failing to provide `noexcept` move ctors can be a missed optimization.

# `std::vector` resize

`std::vector<T>::resize()` will potentially relocate the elements, by move if possible, while providing strong exception safety guarantee:
1. If `T`'s move ctor is `noexcept`, then move the elements to new location.
2. else, the move ctor is potentially throwing, and a nothrow copy ctor is available, the elements will be copied(!)
3. else, use the throwing move ctor. The exception guarantee is waived.

If `T` somehow is not nothrow movable, a potential full copy will be made when relocation happens!

## `std::deque`
`std::deque` will not relocate its elements, if push/pop happens at the front/end. So in most cases this is less of an issue.

# `std::function` and SBO

`std::function` implementations often employ small buffer optimization (SBO) in order to avoid dynamic allocation incurred by type erasure, if the size of the callable fits in.

Without SBO, `std::function`'s move constructor is trivial, you only move the (type-erased) pointer, so it's always `noexcept`. However if SBO is in effect, the callable is stored inline so itself has to undergo move. If the callable is not nothrow movable, neither will be the `std::function` holding it.

Before ISO C++20, `std::function`'s move ctor is not marked `noexcept`.

Some implementations will choose to disable SBO if the callable's type is not nothrow movable, in order to provide strengthened `noexcept` move for `std::function`.

# further reading
* https://ibob.bg/blog/2018/07/03/compiler-generated-move
* https://quuxplusone.github.io/blog/2019/03/27/design-space-for-std-function/
