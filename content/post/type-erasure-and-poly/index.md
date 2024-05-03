---
title: Type erasure and polymorphic interface objects
date: 2023-07-11
lastmod: 2023-12-26

categories:
    - Programming
tags:
    - c++
    - polymorphism
---

# Runtime polymorphism in C++

The canonical way of doing runtime polymorphism in C++ has always been virtual functions and inheritance. Compared to alternative solutions found in other languages, such as:
* Interface in Go
* Trait object in Rust

, the virtual function plus inheritance approach is often considered inferior, for the following reasons:

* __Intrusive design__: All implementers of an interface must inherit from it.
The implementation object also has to make room for the vtable!
* __Mandates reference semantics__: You can only use the interface polymorphically through pointers or references.
A factory function has to return `std::unique_ptr<InterfaceType>`, which is messy. (To be fair, interface objects also require dynamic allocation internally)
* __Performance cost__: They are often found to be slower than alternatives.

# polymorphic object
A polymorphic object type defines a set of methods. Such object can then store another object of abitary type, that implements the defined set of methods.

People have been writing such implementations for years, such as [Poly](https://github.com/facebook/folly/blob/main/folly/docs/Poly.md), [dyno](https://github.com/ldionne/dyno), [proxy](https://github.com/microsoft/proxy).

## benefits
* Allows value-semantics, and copying. Although polymorphic objects still require dynamic allocation in the general case, the user does not need to care about it. Copying is possible, by storing function pointers of copying operations. Virtual inheritance on the other hand, does not play nicely with copy constructors!
* Allows Small buffer optimization. Since they're custom objects, SBO is possible, compared to `std::unique_ptr<InterfaceType>`.
* Allows duck typing. Although this isn't always desired.

## Type erasure
Actually, we already have a limited version of polymorphic object in C++: `std::function`.
It only knows about the `operator()`, but the idea is similar: Use type-erasure.

For polymorphic objects, we need to keep the function pointers somewhere, but for potentially multiple member methods.

> To be continued...
