---
title: Portable Signal handling
date: 2024-05-20
tags:
    - c++
    - c
    - Systems programming
---

# `std::signal` and signal handlers
The standard function `std::signal` enables registering _signal handlers_ that are invoked when a given signal is delivered, asynchronously.
Since the handler can interrupt the execution of the program at potentially any point, actions allowed in the handler are quite limited.
In particular, the handler is allowed to access a global (with static storage duration) variable that is `volatile std::sig_atomic_t` or lock-free `std::atomic<...>`.
This lets us set a flag inside the handler, and correctly check the flag outside the handler.
Additionally, a signal handler can re-call `std::signal` for the same signal that is currently being handled.

The POSIX standard specifies additional library functions that are guaranteed to be async-signal-safe, that is, safe to call from signal handlers (See `man 7 signal-safety`).
However on POSIX systems, the `sigaction` family of functions should be preferred.

# Portable ways to catch signals
Suppose we need to catch a signal and inform the program about it, so it can act accordingly. E.g., catching `SIGINT` (`Ctrl-C`) to perform graceful cleanup and shutdown.
The canonical way of doing this is to have a dedicated thread that responds to a global flag, which shall be set by the signal handler.

Before C++20, there's no platform-independent way, besides polling the flag repeatedly:
```c++
static std::atomic<bool> flag{false}; // assuming it's lock-free

extern "C"
void signal_handler(int sig)
{
    if (sig == SIGINT) {
        flag.store(true);
    }
}

void sigint_thread_fn(/*...*/) { // demonstration only
    while(!flag.load()) {
        std::this_thread::sleep_for(std::chrono::milliseconds(100)); 
    }
    // precede to perform shutdown actions ...
    // e.g., signaling the main thread to stop.
}
```

...Because we don't have a signal-safe function to wait on the atomic flag.

## `std::atomic<T>::wait()`
C++20 introduces `wait()` and `notify()` methods on `std::atomic<T>` and `std::atomic_flag`.
If the atomic type is lock-free (which is always the case for `std::atomic_flag`),
we can safely use `wait()` and `notify()` in the waiting thread and the signal handler, to avoid the sleep-and-poll loop above.
```c++
static std::atomic_flag flag{}; // Always lock-free
                                // C++20 guarantees it's initialized to false

extern "C"
void signal_handler(int sig)
{
    if (sig == SIGINT) {
        flag.test_and_set();
        flag.notify_all();
    }
}

void sigint_thread_fn(/*...*/) {
    flag.wait(false); // will block until flag has been changed.

    // ...
}
```

# Platform-dependent ways

## POSIX
Incrementing a POSIX semaphore (`sem_post`) is signal-safe according to the POSIX specification.
This can be used for notification in signal handlers. The waiting thread would wait via `sem_wait`.

Alternatively there is `sigwaitinfo` and `sigwait`, which explicitly wait for pending signals.

If you're on Linux, another alternative would be using `signalfd` and waiting on the file descriptor (e.g. via `epoll`).

## Windows
