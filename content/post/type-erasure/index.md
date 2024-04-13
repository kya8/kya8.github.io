---
title: Type erasure in C++
date: 2023-07-11
lastmod: 2023-12-26

categories:
    - Programming
tags:
    - c++
---

# Type erasure

Type erasure is a generic concept in progamming, where different types can be hidden behind a single, generic interface.
You probably already did type erasure in C: passing `void*` pointer around and casting them!
This is a simple case of type erasure, which is not type safe. A tagged union also counts, although it only provides erasure for a pre-defined set of types.

# std::function
`std::function<R(Args...)>` is a type-erased polymorphic function object that can wrap all kinds of callables with the specified argument types `Args...` and a return type that is convertible to `R`.
It stores the provided callable by (forwarding) copy/move. Dynamic allocation is required in the general case, however implementations may use SBO (Small Buffer optimization).

It's suitable for use at API boundaries, e.g. when providing a callback.

## Implementation

The actual type erasure happens inside the constructor. Besides storing the callable type-erased (either by dynamic allocation or SBO), `std::function` needs to record information about how to 
do `operator()` later. Since `std::function` is copyable, it also needs to remember how to copy the callable!

There's even more: If SBO is in effect, to move an `std::function` would require invoking the callable's move constructor. Otherwise we just move the pointer.

All of these has to be done inside the constructor of `std::function` since the type information of the callable is only available there.
Here's a limited implementation I wrote. It has no SBO, and doesn't support all kinds of callables (e.g. member function pointers):
```cpp
struct Ptrs {
        void(*deleter_ptr)(void*);
        void*(*copy_ptr)(void*);
};

template<class T>
struct TypePointers {
    constexpr static Ptrs ptrs{
        [](void* p) {
            delete static_cast<T*>(p);
        },
        [](void* p)->void*{
            return new T{ *static_cast<T*>(p) };
        }
    };
};

template<typename>
struct Func;

template<typename R, typename ...Args>
struct Func<R(Args...)> {
    void* ptr = nullptr;
    const Ptrs* func_ptrs; 

    R(*call_ptr)(void*, Args&&...);    

    // needs to disallow same type, otherwise shadows the copy ctor... Also causes inifinite loops
    template<typename T, typename = std::enable_if_t<!std::is_same_v<Func, std::decay_t<T>>>>
    Func(T&& t) {
        ptr = new std::decay_t<T>(std::forward<T>(t));
        func_ptrs = &TypePointers<std::decay_t<T>>::ptrs;

        call_ptr = [](void* p, Args&&... args) -> R {
            if constexpr(std::is_void_v<R>) {
                // the Args types are not deduced. This is not typical perfect-forwarding...
                (R)(*static_cast<std::decay_t<T>*>(p))(std::forward<Args>(args)...);
            } else {
                return (*static_cast<std::decay_t<T>*>(p))(std::forward<Args>(args)...);
            }
        };
    }
    ~Func() {
        func_ptrs->deleter_ptr(ptr);
    }

    R operator()(Args ...args) const {
        struct EmptyFunc{};
        if (!ptr) throw EmptyFunc{};
        return call_ptr(ptr, std::forward<Args>(args)...);
    }

    // copy
    Func(const Func& f) : ptr(f.func_ptrs->copy_ptr(f.ptr)), func_ptrs(f.func_ptrs), call_ptr(f.call_ptr) {}
    // move
    Func(Func&& f) : ptr(f.ptr), func_ptrs(f.func_ptrs), call_ptr(f.call_ptr) {
        f.ptr = nullptr;
    }
};

int main()
{
    using std::cout, std::endl;
    int ans = 42;

    Func<int(char)> func{[&](char c){
        return ans + c;
    }};

    const auto func2 = std::move(func);
    cout << func2(2) << endl;

    Func<short(char)> f3 {func2}; // nested func
    cout << f3(3) << endl;
}
```
It's a rough demo only, but you get the idea: "remember" how to do things later by storing function pointers.

`ptr` is the stored callable.
`func_ptrs` is a group of function pointers, for managing stored callable of type `T`.
`call_ptr` records how to call the type-erased callable.

Another implementation would be an abstract base functor, which defines the virtual `operator()` with desired return type and arg types. In fact we've just implemented something similar to vtables!

## Performance
The performance cost mainly stems from dynamic allocation and function pointer redirections.

# `std::any`
As its name suggests, `std::any` is a *container*, that can wrap an object of any type. The type info is checked on access, so it can provide type safety, compared to `void*`.

Like other *containers*, `std::any` manages storage on its own. Since the size of the object can be arbitary, it generally requires dynamic allocation (so it's more expensive compared to `std::variant`).

The implementation of `any_cast` requires no RTTI, since `std::any` does not need to know the exact type of the contained object, it only needs to check if the requested type matches the actual type when doing `any_cast`. This is typically achieved by storing a pointer to a static member function of a class template that is instantiated by the stored type. Comparing types is thus equivalent to comparing pointers to templated functions.

That being said, `std::any::type()` is available for obtaining runtime typeinfo of the contained object (It does require RTTI, by storing type_info). This allows flexible visitation.

# `std::shared_ptr`
What has `std::shared_ptr` to do with type-erasure? Well, it stores the deleter and allocator type-erased, inside the shared control block, so it's only templated by the object type.

In a typical implementation, `shared_ptr` holds only two pointers: one to the shared object, one to the "control block" that is also shared by all replicated instances. The control block will hold the pointer to the managed object (or the object itself), the deletor and the allocator (both are type-erased), and the ref-counting numbers.

A generic deleter can be supplied during construction, then stored (type-erased) inside the control block.

# `std::variant`
A type-safe union. An instance of `std::variant` at any given time either holds a value of one of its alternative types, or in the case of error - no value.

As with unions, if a variant holds a value of some object type `T`, the object representation of `T` is directly within the object representation of the variant itself. Variant is not allowed to allocate additional (dynamic) memory.
It's usually implemented with placement new, on a buffer that is large enough to hold all kinds of elements, and has appropriate alignment. Or a tagged union.

## Visitation
`std::visit` works on `std::variant` by taking a generic callable that can be applied to any type held by the variant object. If the visitor function is not exhaustive, you get a compile time error.
This allows unified, type-safe operations on `std::variant` objects.

### Implementation
A typical implementation can generate (at compile time) a table of function pointers to the resolved/instantiated function for every value type of the variant. At runtime, the stored type index is used to select the right function.

Again, function pointers!

# Other sum types
`std::optional`, `std::expected`, ...


# Type-erased iterator
E.g. `boost::any_range`, `any_iter`. Such iterators wraps any iterator that satisfy some behaviors, such as being forward iterators. They're especially useful at API boundaries, to provide support for abitary iterator/ranges, without using templates.
