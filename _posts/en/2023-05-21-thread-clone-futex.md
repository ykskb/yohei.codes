---
layout: default
title: "Thread, clone, futex and CMPXCHG"
lang: en
image:
    path: /assets/images/dp-regex.png
---

# Thread, clone, futex and CMPXCHG

This is a page compiled from my notes I've taken on thread(pthread)-related topics sporadically over time. It's my effort to sort out understanding and remember by putting them together.

## pthread Internals

[pthreads](https://man7.org/linux/man-pages/man7/pthreads.7.html) calls [`clone()`](https://man7.org/linux/man-pages/man2/clone.2.html). It creates a `task_struct`.

> Notes:
>
> * pthread (POSIX threads) is part of libc. Most of Linux have it implemented by glibc (GNU C Library).
>
> * Implemented in user space, and interacts with kernel through system calls like `clone()` and [`futex`](https://man7.org/linux/man-pages/man2/futex.2.html).
>
> * [`fork()`](https://man7.org/linux/man-pages/man2/fork.2.html) (libc) also calls `clone()` but with different params.

* `clone()` has different flags like these: 

    * `CLONE_VM`: memory space (`fork()` does not share memory space) 

    * `CLONE_FS`: file system

    * `CLONE_FILES`: file descriptors

    * `CLONE_SIGHAND`: signal handlers

    * `CLONE_THREAD`: threads

    * `CLONE_PARENT_SETTID`: 

    * `CLONE_CHILD_CLEARTID`: 

> Notes: libc vs kernel system call
>
> Standard interface for kernel system calls is libc for portability and safety. Technically `man 2 [name]` describes kernel system call and `man 3 [name]` describes libc function, however even `man 2` displays lib in library section.
>
> To find out if a libc function is just a wrapper or has more logics on top of system calls, we need to find out internals / knowledge of the function. But if a function only exists in `man 3` then it's obvious that the function has custom logics wrapping system calls in libc.

## Kernel Perspective

Both processes and threads are represented as `task_struct` in kernel.

* Scheduling: unified task scheduling
    
    Kernel doesn't schedule processes and threads diffently though it knows about shared address spaces and resources for threads.

    * Preemptive multitasking: OS can interrupt and switch between threads. (OS uses hardware timer interrupts.)

    * Criterion of multitasking:
        
        * Time slice (quantum)

        * I/O operation

        * Priority

    * Completely fair scheduler (CFS) is default. Other policies for real-time: `SCHED_FIFO` and `SCHED_RR` exist as well.

* Thread states (registers, program counter and stack pointer etc) are saved as `Thread Control Block (TCB)` while process states are saved as `Process Control Block (PCB)`.

* Difference between threads vs processes:

    * Threads' `task_struct` would be a part of a thread group of a process.

    * Threads' structs point to shared resources.

## Controlling Kernel Schedule

* CPU affinity: determines which CPU core a thread / process can run.

    Binds threads to specific core, reducing context switching and improving cache utilization.

    * [`sched_setaffinity(pid, cpusetsize, mask)`](https://man7.org/linux/man-pages/man2/sched_setaffinity.2.html)

    * [`sched_getaffinity(pid, cpusetsize, mask)`](https://linux.die.net/man/2/sched_getaffinity)

* Policy: different policies can be set.

    [`sched_setscheduler(pid, policy, param)`](https://man7.org/linux/man-pages/man2/sched_setscheduler.2.html) and `sched_getscheduler(pid)`

    * `SCHED_OTHER`: default Linux time-sharing scheduler

    * `SCHED_FIFO`: first-in, first-out real-time scheduling

    * `SCHED_RR`: round-robin real-time scheduling

    * `SCHED_IDLE`: very low priority for background tasks

    * `SCHED_BATCH`: suitable for batch processing jobs

* Priority: priority can be set as well.

    [`setpriority(which, who, priority)`](https://linux.die.net/man/2/setpriority) and `getpriority(which, who)`

    * `type`: such as `PRIO_PROCESS`

    * `who`: identifier

    * `priority`: new priority value

* Command:

    `nice` and `renice` modify priority.

    * Value ranges from `-20` (highest) to `19` (lowest)

* pthread:

    * [`pthread_setschedparam`](https://man7.org/linux/man-pages/man3/pthread_setschedparam.3.html) and `pthread_getschedparam`: scheduling policy and params

    * [`pthread_attr_setschedpolicy`](https://man7.org/linux/man-pages/man3/pthread_attr_setschedpolicy.3.html) and `pthread_attr_setschedparam`: thread attributes

## User-level Threading

* Coroutines: functions that can suspend execution and resume later. Common in Python (async/await), Kotlin and Lua.

* Fibers: similar to coroutines but often more general-purpose. Ruby and C++ have it.

* `m:n` threading: `m` user-level threads mapped to fewer `n` kernel threads.

    * Advantages: flexible / can adjust `n` / no context switches / can handle blocking operations

    * Disadvantages: complexity / debugging / limited OS support (Linux and Windows use `1:1` threading model)

    * OS level: mostly `1:1` for simplicity (Old Solaris and GNU Portable Threads provide `m:n` mapping)

    * Examples:

        * Go: `goroutines` are scheduled onto a smaller number of OS threads.

        * Erlang: through BEAM virtual machine, lightweight processes are scheduled onto a smaller number of OS threads.

        * NodeJS: not conventional, but multiple tasks are handled concurrently with a small number of threads through event loop (event-driven architecture).

> Languages without `m:n`
>
> Languages like Java or Rust provide provide an interface to kernel threads with `1:1` mapping as a language feature instead of providing `m:n` or green thread model.
> 
> In those language, `m:n`, green thread or fiber model of multithreading are typically implemented as libraries outside of language features / designs.
        
## Synchronization Mechanisms

With multiple threads, contentions need to be handled. pthread provides synchronized primitives such as `mutex`, `condition variable` and `semaphore`.

* __Mutex__ (`pthread_mutex_t`): only one thread can access a critical section of code at a time.

    * Uses `futex`.
    
    * Has ownership: only lock owner thread can update.

    * Implementation: internal state (The kernel or operating system provides mechanisms to block and wake up threads.)

* __Condition variable__ (`pthread_cond_t`): blocks a thread until a particular condition is met, usually in combination with a mutex.

    * It releases the associated mutex and uses `futex` to wait for a signal or broadcast from another thread.

* __Semaphore__ (`sem_t`): controls access to a shared resource.

    * Similar to mutexes, when the semaphore count is zero and a thread tries to decrement it, the thread uses `futex` to wait. When the count is incremented, `futex` is used to wake up one of the waiting threads.

    * No ownership: any thread can update the value.
    
    * Use case: resource pool like DB connection pool / producer-consumer availability

    - Implementation: counter + wait queue

## Futex

Stands for "fast userspace mutex." Only when contention occurs futex interacts with the kernel.

> Notes:
> 
> It might seem like `futex` is not part of the kernel at first glance as it is designed to minimize kernel involvement in typical use cases, but it is a kernel construct.

* Fast path: mutex is free → lock acquired (only in userspace)

* Slow path: mutex is not free → thread makes a system call, and the kernel puts a thread to sleep until the mutex is unlocked.

* `futex` provides a mechanism to put threads to sleep and wake them up efficiently, with the kernel managing the wait queues.

    * `futex_wait`: A thread calls this to indicate that it is waiting on a futex variable (typically a specific memory location). The kernel puts the thread to sleep if the condition is not met (e.g., a lock is already held).

    * `futex_wake`: used to wake one or more threads waiting on a futex variable. When a lock is released or a condition changes, futex_wake can be used to notify the waiting threads, allowing them to continue execution.

* Compare-and-swap (CAS): atomic operation used for lock acquiring.

    * X86/64: `CMPXCHG destination, source` with `EAX`, `EBX`, `ECX`, etc. (General-purpose registers) or `RAX`, `RBX`, `RCX`, etc. (64-bit general-purpose registers)

    * Arm: `LDREX` (load exclusive) and `STREX` (store exclusive) with General-purpose registers (e.g., `r0`, `r1`, etc.)

        ```asm
        LDREX r0, [address]      ; Load the value at 'address' into 'r0'
        CMP r0, old_value        ; Compare the loaded value with 'old_value'
        STREXEQ r1, new_value, [address] ; If equal, store 'new_value' into 'address'  atomically
        ```
