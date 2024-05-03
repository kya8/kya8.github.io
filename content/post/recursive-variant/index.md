---
title: Recursive std::variant
# description: 
date: 2024-02-18
tags:
    - c++
---

# Recursive references
Recursive data structures are data types that refers to themselves. Chances are you're already familiar with linked lists:
```c++
struct ListNode {
    int data;
    ListNode* next;
};
```
A `ListNode` owns some data, and contains a (non-owning, nullable) reference to the next node.

Why does this work? Because C/C++ allows to define a reference to an **incomplete type**. If we leave out the pointer part of `next`, the compiler should complain about something like this:
>**error:** field '**next**' has incomplete type '**ListNode**'  
>**note:** definition of '**struct ListNode**' is not complete until the closing brace

..., which is self-explanatory: During the definition of a struct, the struct itself is seen as an incomplete type. An incomplete type has unknown size, so the compiler couldn't figure out how much storage it'll use.

However, references/pointers to incomplete types are allowed here, because they have fixed size, in terms of implementations.

## Ownership
For simple one-way linked lists, it is possible to replace the raw pointer with `std::unique_ptr<ListNode>`, so the first node automatically owns the entire list by recursion.
# Recursive variant
`std::variant` is often employed to represent data with runtime-variable type. A recursive variant is a variant type that is able to store itself, which is useful for hierarchical data structures. For example, a property list structure could represent its node using `using Node = std::variant<t1, t2, t3, ... , Self>`, where `Self` is some type that contains another (child) `Node`. An instance of `Node` is a leaf node if it's not of type `Self`.

We cannot simply write `using Node = std::variant<t1, ..., Node>`, since recursive `using` is simply not allowed. Some sort of indirection struct is required here. We could forward declare a struct that contains `Node`, and use it with the variant. What if:
```cpp
using Node = std::variant<t1, t2, t3, ... , struct NodeHolder>;
struct NodeHolder {
    Node node;
}; 
```
Again, this won't work either. To determine `sizeof(Node)` would be an infinite recursion.

Simply wrap the forward declared struct inside a smart pointer:
```cpp
using Node = std::variant<t1, ..., std::unique_ptr<struct NodeHolder>>;
struct NodeHolder {
    Node node;
};
```
This way, the parent node has ownership over the child node (if it has one). The additional price we pay is dynamic memory allocation.

Let's test this with some examples:
```cpp
using VPtr = std::unique_ptr<struct S>;
using V = std::variant<int, float, VPtr>;

struct S {
    V v;
};

struct Visitor {
    using std::cout;
    void operator()(const int& i) const {
        cout << "int: " << i << '\n';
    }

    void operator()(const float& f) const {
        cout << "float: " << f << '\n';
    }

    void operator()(const VPtr& p) const {
        cout << "VPtr: ";
        std::visit(*this, p->v); // recurse visitation
    }
};

int main()
{
    V v1{1};
    V v2{std::unique_ptr<S>{new S{std::move(v1)}}}; // v2 now owns v1
    V v3{std::unique_ptr<S>{new S{std::move(v2)}}}; // v3 now owns v2
    // make_unique? See: https://wg21.link/p0960

    std::visit(Visitor{}, v1);
    std::visit(Visitor{}, v3);
}
```
This should print:
```
int: 1
VPtr: VPtr: int: 1
```
Inside `main()`, `v1` is constructed, then moved into `v2` as a child. `v2` is again transferred to `v3`. Since `v1` contains a plain `int`, the first move is equivalent to a copy. Semantically, `v3` owns all the data, and is automatically released when dropped.
The `Visitor` prints data contained in `V`, and if it has a child node, visit that child node recursively.

## Removing the wrapper type

We had to wrap the `Node` inside a wrapper type, since forward declaration doesn't make sense for a `using` alias. We could get around this via inheritance:
```cpp
struct Node : std::variant<t1, ..., std::unique_ptr<Node>> {};
```

## Use with map
`V` isn't particularly useful, it only serves as a demonstration. In practice, we're more likely to place the variant in some container, e.g.:
```cpp
using Map = std::map<std::string, std::variant<t1, t2, ..., std::unique_ptr<struct MapHolder>>>;

struct MapHolder {
    Map map;
};
```
The intent is clear: `Map` should be able to store instances of itself as its values.
The above approach requires two dynamic-allocated indirections, one of which (the `unique_ptr`) is redundant.

We could go the other way around, by declaring the value type first:
```cpp
struct Value : std::variant<int, float, /*... more types*/ std::map<std::string, Value>> {};

void test() {
	Value r1{42};
	Value r2{std::map{ std::pair{std::string("key"), std::move(r1)} }};
}
```

Or, to make our intent clearer:
```cpp
struct Value;
using Map = std::map<std::string, Value>;
struct Value : std::variant<int, float, /*... more types*/ Map> {};

//...
```

# Application: A heterogeneous data structure
We could represent a JSON-like data structure that can hold heterogeneous data recursively,
using the tricks demonstrated above.

The data representations:
```cpp
struct Value;
using Map = std::map<std::string, Value>;
using Array = std::vector<Value>;
struct Value : std::variant<int, std::string, Array, Map> {
    // Inherit constructors and assignment op, to make things easier.
    using std::variant<int, std::string, Array, Map>::variant;
    template<class Ts>
    auto& operator=(Ts&& rhs) {
        std::variant<int, std::string, Array, Map>::operator=(std::forward<Ts>(rhs)); // Add more types, e.g. double here.
        return *this;
    }
};
```

We could serialize `Value` into string, recursively:
```cpp
void print_value(const Value& val, std::ostream& out) {
    const auto visitor = [&](const auto& v) {
        using T = std::decay_t<decltype(v)>;
        if constexpr (std::is_same_v<T, int>) {
            out << v;
        } else if constexpr (std::is_same_v<T, std::string>) {
            out << '"' << v << '"';
        } else if constexpr (std::is_same_v<T, Array>) {
            bool first = true;
            out << '[';
            for (const auto& x : v) {
                if (!first) {
                    out << ", ";
                }
                first = false;
                print_value(x, out);
            }
            out << ']';
        } else if constexpr (std::is_same_v<T, Map>) {
            bool first = true;
            out << '{';
            for (const auto& [key, x] : v) {
                if (!first) {
                    out << ", ";
                }
                first = false;
                out << key << ": ";
                print_value(x, out);
            }
            out << '}';
        }
    };

    std::visit(visitor, val);
}
```
The visitor is a generic lambda that can be applied to all types represented by `Value`.
It recurses into `Array` or `Map` types.

To put things together:
```cpp
int main()
{
    Value r2{Map{ std::pair{std::string("key"), Value{42}} }};
    std::get<Map>(r2)["arr"] = Array{"abc", 123};

    print_value(r2, std::cout);
}
```
It prints:
```
{arr: ["abc", 123], key: 42}
```
, as expected.
