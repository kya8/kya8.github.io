---
title: Rule of 0/3/5
description: and their relevance to RAII
date: 2023-01-18
tags:
    - c++
    - RAII
---

# The rule of 0/3/5

A C++ class that manages some non-RAII resource/object (e.g., `FILE*`) would require a user-defined dtor.
Also, such resource generically cannot be copied plainly. Most of the time, this either disallows copying, or requires special handling.

Such classes are direct (i.e. it doesn't contain another RAII wrapper object), exclusive resource managers.

**On the other hand, if a class is not one of such classes, it should not have custom destructors, copy/move constructors or copy/move assignment operators.
Let RAII do its thing! Only the lowest-level resource manager classes (where all the special handling actually happens) would need those.**

With these in mind, the rule of 0/3/5 becomes obvious:

## Rule of three
If a class requires a user-defined destructor, a user-defined copy constructor, or a user-defined copy assignment operator, it almost certainly requires all three!
Because it certainly manages some non-RAII resource handle, otherwise it wouldn't require those.

## Rule of zero

Classes that have custom destructors, copy/move constructors or copy/move assignment operators should deal exclusively with ownership (which follows from the Single Responsibility Principle). Other classes should not have custom destructors, copy/move constructors or copy/move assignment operators.

This rule also appears in the C++ Core Guidelines: If you can avoid defining default operations, do.

## Rule of five

Because the presence of a user-defined destructor, copy-constructor, or copy-assignment operator prevents implicit definition of the move constructor and the move assignment operator,
any class for which move semantics are desirable, has to declare all five special member functions.

Unlike Rule of Three, failing to provide move constructor and move assignment is usually not an error, but a missed optimization opportunity.
