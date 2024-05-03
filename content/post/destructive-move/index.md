---
title: Non-destructive move, null object and class invariants
date: 2023-09-29
lastmod: 2024-08-28
tags:
    - c++
    - rust
---

# Class invariant
A class invariant is a condition that always holds true for any object of the class, throughout its lifetime.
Typical class invariants include consistency of some internal state, or that a resource-owning class always acquires and manages some resource.

Proper designing and maintaining of class invariants provides useful guarantees for users (and implementers) that lowers our mental burdens.

## Enforcing class invariant at construction
The canonical way is to throw in the constructor, if it fails to establish the required invariants. Alternatively we could force the use of a factory constructor, which returns `std::optional<T>` or similar.

## Default-constructed, zombie object
Sometimes, we're left with an _valid_ object that fails to meet some condition, e.g. a resource owning class object that hasn't acquired any resources. Such zombie objects are rather common in C++ codebases. For example, if a class invariant requires some argument to construct, a default constructor of the class, if defined, will create a zombie object.

In many situations, we're forced to make our class default-constructible (e.g. resizing `std::vector`)

A zombie (or null, nil) object is not broken, it just introduces an additional null state, which adds some responsibility/contract of checking the object state on the user (and sometimes on the implementer).

# Move compromises class invariant
For a resource owning class, according to *the Rule of 0-3-5*, we could define custom move constructor and assignment operator for it, which shall transfer resource ownership. However, this requires compromising the class invariant that it always holds a resource, since the moved-from object must have been emptied, and marked as null. A zombie object is leaked by move! This is a price we pay for non-destructive move in C++.

Still, such _weak_ invariant is better than no invariant at all. We could further weaken the invariant by defining public, non-throwing (default) constructors that create zombie objects.

If only move produces zombie objects, we just tell the user not to use moved-from objects of our class, which is not a big deal, and on our side we only need to do additional checking when releasing the resource (to prevent potential double-free, etc.).

## Non-destructive move
In C++, moving from an object has nothing to do with its lifetime. It's just a plain function invoked on the rvalue-referenced rhs object, nothing special.

Although the move function can do basically anything, in practice we want it to transfer owned resources, which leaves the moved-from object emptied. And when the object gets out of scope, its destructor gets called, which checks for null and finds it is already emptied.

This operation might leave the moved-from object in a null state, but does not terminate its lifetime, thus compromising class invariants. The root of all evil!

Some languages, e.g. Rust has **destructive moves**. In Rust, a moved from object is no longer valid, and you get a compile-time error if trying to access them afterwards. This way, moving won't affect class invariants, since it terminates the object's lifetime.

The implementation of move semantics in Rust is not much different (it could simply be a memcpy). The magic is language support such that moved-from objects are no longer valid, not the move itself. Also the destructor of the moved-from object will NOT be called, since it is indeed moved, so the responsibility of freeing is transferred away. Compare to C++, a moved-from variable is still there, and can be used as normal, so its destructor still needs to be called when its lifetime actually ends (which means the destructor should check for null).

In all, moved-from objects in C++ are (usually meant to be) null objects, while in Rust, moved-from objects are ...., **absence** of objects!

# Sum-types for null

We've talked about zombie/null objects, which embed a special internal state of null (or not).

If the class invariant that the object cannot be null always holds, we should lift the null state indication out of the object: If the class invariant fails to establish, then there shouldn't be an object at all!

It's basically going from zombie/null object vs. non-null object, to invariant-holding object vs. **the absence of object**. The latter can be cleanly represented by sum types, such as `Optional`, `Result` in Rust, and `std::optional`, `std::expected` in C++.
