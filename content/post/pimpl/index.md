---
title: The PIMPL idiom
date: 2022-12-03
tags:
    - c++
---

# Problem with `private`
```cpp
class MyClass {
private:
	/* internal implementation details */
public:
	/* public stuff meant to be exported */
};
```

Problems:
* `private` stuff included in header, breaking encapsulation
	* Undesired disclosure for proprietary projects
* Additional headers needed by `private` stuff, are included in header
	* Some 3rd-party headers do funky things
	* Increased compilation time
* Breaks ABI, since whenever `private` changes, the class ABI might change as well

## Using functions
C allows true encapsulation via ***opaque struct*** pointers. But what if I still want to use classes?

# Enter PIMPL
>Short for Pointer to Implementation

The header file would look like:
```cpp
class MyClass {
public:
    // public methods go here

    ~MyClass() noexcept; // required
    MyClass(/* args to forward to Impl... */);

private:
    struct Impl;
    std::unique_ptr<Impl> pimpl;
}
```

The actual implementation class `MyClass::Impl` is only forward-declared in the header.
This requires us to forward-declare the destructor of `MyClass`, otherwise the compiler generated definition will attempt to call the dtor of `Impl`, which is not declared in this header.

The source file:
```cpp
struct MyClass::Impl {
    // define the actual members here.

    Impl(/* args... */) {
        // ...
    }
    ~Impl() noexcept; // define it
}

MyClass(/* args... */) : pimpl(std::make_unique<Impl>(/* args... */)) {} // always use make_unique
~MyClass() noexcept = default;

// methods for public class
void MyClass::someMethod() {
    // use pimpl->...
}
```

Always use `unique_ptr` and `std::make_unique` for PIMPL. DO NOT use a raw (or even `void*`) pointer!

## Copyable PIMPL class
PIMPL classes are already move-able for free, because the `unique_ptr` is move-able.
To copy the actual `Impl` object, the new `MyClass` object should copy-construct its `pimpl` member, provided the `Impl` class is itself copyable.
```cpp
// should be declared in the header first...
MyClass(const MyClass& rhs) : pimpl(std::make_unique<Impl>(*rhs.pimpl)) {}
```

# Comparison with C++20 modules
