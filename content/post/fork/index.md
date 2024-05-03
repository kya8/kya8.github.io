---
title: Fork(2) and the OOM killer
description: Seriously, what the f**k?
date: 2023-02-16
categories:
    - Linux
tags:
    - Linux
    - Systems programming
draft: false
---

# The `fork(2)` syscall

This is one of the identifying features of a Unix-like operating system. `fork(2)` has existed since the first version of Unix back in the days of PDP-11.
It creates a new process by simply duplicates the entire address space of the calling process, after which point they run indenpendently.

`fork`+`exec` is the standard way to create a child process in Unix, although it has an obvious problem: The parent process will have its entire process space copied,
only to spawn a new process that has little regard for any of it. In fact, this is what the First Edition Unix did.

## The virtual memory story
Overtime, people got smart and started implementing Copy-On-Write for `fork`, so it din't need to actually copy that much pages. In the case of `fork`+`exec`, no extra
segments will be copied. This is the most prominent reason for defending `fork`: The OS only copies when needed! Nothing is wasted.

Indeed, the CoW approach did solve some of `fork`'s problems, but not all. In particular, The file descriptor inhertance from `fork` isn't nice to play with, and has lead to many hacks (such as `CLOEXEC`).
Not only that, this model has implicitly lead to memory overcommittment and the OOM killer on Linux, which IMO is one of the most glaring design failures on Linux. 

# The OOM killer

Support we have 8G of memory in total, and some process has taken-up 6G of it (e.g., by `malloc` and `memset`). Now this process calls `fork` to create a new child process.
We all know that will not fail on Linux. What if the calling process merely wants to `fork` and `exec` a new process? The 2G remaining memory is more than enough for it.
After all, it is the obvious point for CoW implementations of `fork` which we all agree upon.

There's no way for the OS to know how the child process will be used after `fork`. It may exec a tiny program and exit immediately, or it might **actually** fiddle with all the "copied" data.
The CoW implementation is reponsible for keeping this transparent, so everyone is happy. That is, until the child process starts modifying more than 2G of memory and get killed by OOM!

By CoW `fork` and overcommittment, the Linux kernel essentially lies to the child process that it has allocated 6G of memory, while effectively it can write to only 2G of it (given the parent process does not release its memory).
The child is free to use that 6G of memory however it wants, but there's no guarantee it won't exhaust all available memory and get killed. Maybe the parent process will release its hold of memory at some point,
so the lie is covered up. Who knows! Most user-land processes, might get killed by OOM, essentially at any point, because the kernel lies to user-land processes about memory reservations.

There are some other examples of memory overcommittment, e.g. `malloc`, but that is a different story.

# Alternatives

## vfork and clone
The `vfork` syscall was an early alternative to `fork`, first appearing on BSD. It's essentially a restricted version of `fork`.

A more functional alternative would be `clone` on Linux. From `man 2 clone`:
>By contrast with fork(2), these system calls provide more precise control over what pieces of execution context are shared between the calling process and the child process...

## spawn

The obvious solution is the _spawn_ model, where a new process is constructed from a clean state. The user only opts in for some inheritance or shared state, so it has none of the flaws of fork.

A somewhat failed attemp is `posix_spawn` defined by POSIX standard. Essentially no one uses it since it's far too cumbersome.

If implemented correctly, spawn should be preferable to `fork` in most cases.
The `fork` model *__can__* be genuinely useful in some contexts, however most of time people just `fork` and `exec` to create child processes, which gave birth to its shortcomings.
