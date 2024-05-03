---
title: Pointers (again)
date: 2023-01-04
lastmod: 2024-06-02
draft: false

tags:
    - c
    - c++
---

# Pointer arithmetics
It is well-known that in both C and C++, pointer arithmetics is only meaningful for operands (and the imaginary resultant pointer address) that are related to each other.
By related we mean, they both point to the same object (or one-past-end), or the same array (or one-past-end).

Per C++ standard, pointers are _iterators_, not _numbers representing address space_, although they're often implemented as such.

To quote https://stackoverflow.com/a/56038995/17815599:
>Speaking more academically: **pointers are not numbers**. They are pointers.
>
>It is true that a pointer on your system is implemented as a numerical representation of an address-like representation of a location in some abstract kind of memory (probably a virtual, per-process memory space).
>
>But C++ doesn't care about that. C++ wants you to think of pointers as post-its, as bookmarks, to specific objects. The numerical address values are just a side-effect. The _only_ arithmetic that makes sense on a pointer is _forwards and backwards_ through an array of objects; nothing else is philosophically meaningful.

So, `p1 - p0` isn't really saying "give me the difference between the addresses pointed by `p1` and `p0`",
it's saying "Assuming they both refer to objects in the same array, what's the difference between their indices inside the array?"
(Here, we consider a single object as an array of length one). If the assumption does not hold, this is undefined behavior.
For relational (greater than, less than...) comparisons however, the behavior is _unspecified_.

## what are "arrays"?

It covers a bit wider than normal (i.e. `int arr[]`) arrays.
In particular, pointer returned by `malloc` can be addressed as arrays.

Also see https://stackoverflow.com/questions/47830449/can-you-do-arithmetic-on-a-char-pointing-at-another-object for aliasing `char*` pointers.

## pointer to struct member
```cpp
struct xyz_s {
	float x, y, z;
};

xyz_s pt;
auto px = &pt.x;
auto py = px + 1 // ???
```
`xyz` is standard-layout type, which means it has C-like memory layout. However in this example, _even if the struct `xyz_s` does not contain any paddings_, `py` does not actually point to `pt.y`. It points to "one-past-end" of `pt.x`!
That's because there're no arrays here, so `py` can only be the "one-past-end" pointer of `pt.x`. Although evaluating `px+1` is well-defined, dereferencing the resultant `py` is UB. It's not `pt.y`!

If we change `xyz_s` to contain an array of 3 floats, that kind of pointer arithmetic is okay. Also absence of padding in array is guaranteed.

### Padding
Actually, the compiler is free to insert paddings inside a struct in an implementation-defined way, **except at the beginning** of a struct. There's nothing in the standard mandating no paddings in `xyz_s`!

This is also why there's no guaranteed size for `std::array<T, N>`. Even if it only contains a single member `T data_[N]`, the compiler can still insert paddings after it.

## pointer in nested arrays
```cpp
struct Pt3 { float xyz[3]; };
std::vector<xyz_s> vec(N);
auto p0x = &vec[0].xyz[0];
auto p0z = p0x + 2; // okay
auto p1x = p0z + 1; // bad
```

Here, `p1x` is still one-past-end of `vec[0].xyz[2]`. Because, there's no array of type `float` and length `N*3`! And consider possible paddings mentioned above.

# Pointer to first member of (standard layout) struct
(By "first" we mean the first declared member)

Ref: https://en.cppreference.com/w/cpp/language/static_cast#Pointer-interconvertible_objects

Two objects a and b are _pointer-interconvertible_ if:
- they are the same object, or
- one is a union object and the other is a non-static data member of that object, or
- one is a standard-layout class object and the other is the first non-static data member of that object or any base class subobject of that object, or
- there exists an object c such that a and c are pointer-interconvertible, and c and b are pointer-interconvertible.

...combined with pointer conversion rules, it's possible to convert between pointer to a standard-layout struct and pointer to its first member, via an intermediate `void*` pointer.

# Pointer total order
There is actually one exception to the "same-object-or-array" rule: the total order of pointers in C++.
It only applies to comparison functors, `std::less`, `std::greater` etc. This total order is consistent with defined pointer comparison rules of the built-in comparison operators.
For the unspecified case, the result becomes implementation-defined.

Note that in C, relational comparison of unrelated pointers is _undefined_.
