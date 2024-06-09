---
layout: post
title: "[MIT_6.S081] Lab: Multithreading"
subtitle: "Multithreaded Programming Practices"
date: 2024-06-08
author: "Michael Xi"
header-img: "img/disk.jpg"
tags: [System, Operating System, MIT 6.S081]
---

# Uthread: switching between threads ([moderate](https://pdos.csail.mit.edu/6.828/2021/labs/guidance.html))

## Understand the Question

This lab question focuses on implementing the **user-level** threading logic, including thread context switching, thread creation, and thread scheduling. The threading switching function is implemented in the `user/uthread_switch.S` file. The thread creation and scheduling functions are located in `user/uthread.c` file.

The **thread context switching** needs to save the registers of the thread being switched away from and restore the registers of the thread being switched to. Note that the registers should be callee-save registers.

## Solution

The user-level threading library uses the following important structs:

```c
struct thread_context {
  uint64 ra;
  uint64 sp;

  uint64 s0;
  uint64 s1;
  uint64 s2;
  uint64 s3;
  uint64 s4;
  uint64 s5;
  uint64 s6;
  uint64 s7;
  uint64 s8;
  uint64 s9;
  uint64 s10;
  uint64 s11;
};

struct thread {
  char       stack[STACK_SIZE]; /* the thread's stack */
  int        state;             /* FREE, RUNNING, RUNNABLE */
  struct thread_context context;
};

struct thread all_thread[MAX_THREAD];
extern void thread_switch(uint64, uint64);
```

Each thread has its stack for function running, a state for scheduling, and a context for thread switching. These fields are crucial for the thread functionalities.

**First**, the **thread switching** takes place on the register level, and we need to save the current registers in memory and load the new thread context from memory.

```assembly
	.text

	/*
   * save the old thread's registers,
   * restore the new thread's registers.
   */

	.globl thread_switch
  thread_switch:
        sd ra, 0(a0)
        sd sp, 8(a0)
        sd s0, 16(a0)
        sd s1, 24(a0)
        sd s2, 32(a0)
        sd s3, 40(a0)
        sd s4, 48(a0)
        sd s5, 56(a0)
        sd s6, 64(a0)
        sd s7, 72(a0)
        sd s8, 80(a0)
        sd s9, 88(a0)
        sd s10, 96(a0)
        sd s11, 104(a0)

        ld ra, 0(a1)
        ld sp, 8(a1)
        ld s0, 16(a1)
        ld s1, 24(a1)
        ld s2, 32(a1)
        ld s3, 40(a1)
        ld s4, 48(a1)
        ld s5, 56(a1)
        ld s6, 64(a1)
        ld s7, 72(a1)
        ld s8, 80(a1)
        ld s9, 88(a1)
        ld s10, 96(a1)
        ld s11, 104(a1)

        ret    /* return to ra */
```

**Side Quest**: What is the difference between the **caller-save registers** and **callee-save registers**?

- **Callee-Save Registers**:

  - **Responsibility**: Saved and restored by the callee (the function being called).

  - **Usage**: Used when the callee needs to preserve the register values across function calls.

  - **Example**: When the callee needs to use certain registers but ensure their original values are restored before returning control to the caller.

- **Caller-Save Registers**:

  - **Responsibility**: Saved and restored by the caller (the function making the call).
  - **Usage**: Used for temporary values that the caller needs to preserve across function calls.

  - **Example**: When the caller needs to ensure certain register values are not overwritten by the callee.

**Second**, the **thread creation** needs to find an idle thread with a FREE state, and then initialize the thread fields to make it a runnable thread that ready to be scheduled.

```c
void 
thread_create(void (*func)())
{
  struct thread *t;
	
  // Find a thread with FREE state
  for (t = all_thread; t < all_thread + MAX_THREAD; t++) {
    if (t->state == FREE) break;
  }
  
  // Initialize new thread fields
  t->state = RUNNABLE;
  t->context.ra = (uint64)func;
  t->context.sp = (uint64)(t->stack + STACK_SIZE); /* set to point to top of stack */ 
}
```

**Third**, for **thread scheduling**, we need to first find another runnable thread in all threads using iteration and then conduct context switching between the old and new thread.

```c
void 
thread_schedule(void)
{
  struct thread *t, *next_thread;

  /* Find another runnable thread. */
  next_thread = 0;
  t = current_thread + 1;
  for(int i = 0; i < MAX_THREAD; i++){
    if(t >= all_thread + MAX_THREAD)
      t = all_thread;
    if(t->state == RUNNABLE) {
      next_thread = t;
      break;
    }
    t = t + 1;
  }

  if (next_thread == 0) {
    printf("thread_schedule: no runnable threads\n");
    exit(-1);
  }

  if (current_thread != next_thread) {
    next_thread->state = RUNNING;
    t = current_thread;
    current_thread = next_thread;
		/* switch threads */
    thread_switch((uint64) &t->context, (uint64) &current_thread->context);
  } else
    next_thread = 0;
}
```

Note that the `thread_switch` function is declared `extern` in the current `uthread.c` file and implemented in the `uthread_switch.S` file in assembly language.

# Using threads ([moderate](https://pdos.csail.mit.edu/6.828/2021/labs/guidance.html))

## Understand the Question

This part of the lab is a basic practice of parallel programming using UNIX pthread library APIs. The `ph.c` contains a simple hash table using separate chaining. The original code is correct under one thread situation and is incorrect when used from multiple threads. The goal is to correct the code and make it multi-threaded safe. We should use the following pthread APIs:

```c
pthread_mutex_t lock;            // declare a lock
pthread_mutex_init(&lock, NULL); // initialize the lock
pthread_mutex_lock(&lock);       // acquire lock
pthread_mutex_unlock(&lock);     // release lock
```

## Solution

The HashMap is an array of entry pointers, each bucket is an entry pointer LinkedList.

```c
#define NBUCKET 5
#define NKEYS 100000

struct entry {
  int key;
  int value;
  struct entry *next;
};

struct entry *table[NBUCKET];
int keys[NKEYS];
```

The part that is not multi-threaded safe is the `put` function that adds a new pair of keys and values to HashMap. We need to add a pthread lock during the insertion function takes place.

```c
static 
void put(int key, int value)
{ 
  int i = key % NBUCKET;

  // is the key already present?
  struct entry *e = 0;

  for (e = table[i]; e != 0; e = e->next) {
    if (e->key == key)
      break;
  }
  
  if(e){
    // update the existing key.
    e->value = value;
  } else {
    // the new is new.
    pthread_mutex_lock(&lock);
    insert(key, value, &table[i], table[i]);
    pthread_mutex_unlock(&lock);
  }
}
```

```c
michaelxi@michaelxi-VirtualBox:~/xv6-labs-2021$ ./ph 2
100000 puts, 5.830 seconds, 17152 puts/second
1: 0 keys missing
0: 0 keys missing
200000 gets, 11.630 seconds, 17196 gets/second
```

# Barrier([moderate](https://pdos.csail.mit.edu/6.828/2021/labs/guidance.html))

## Understand the Question

Barrier is a point in an application at which all participating threads must wait until other participating threads reach that point too. After the pthread lock APIs, we need to use pthread condition variable-related APIs to implement this Barrier.

```c
pthread_cond_wait(&cond, &mutex);  // go to sleep on cond, releasing lock mutex, acquiring upon wake up
pthread_cond_broadcast(&cond);     // wake up every thread sleeping on cond
```

## Solution

The important barrier struct is below, which includes two shared variables that tracks the number of threads that have reached the barrier and the barrier round.

```c
struct barrier {
  pthread_mutex_t barrier_mutex;
  pthread_cond_t barrier_cond;
  int nthread;   // Number of threads that have reached this round of the barrier
  int round;     // Barrier round
} bstate;
```

To implement a barrier, we need to count the number of threads that arrive at the barrier.

- If the number of threads that have arrived is less than the total number of threads that should arrive, then the current thread sleeps on the condition variable.
- After all threads arrive at the barrier, the condition is met, and we can wake up all threads and proceed to increment round count.

```c
static void 
barrier()
{
  // Block until all threads have called barrier() and
  // then increment bstate.round.
  
  pthread_mutex_lock(&bstate.barrier_mutex);
  bstate.nthread = bstate.nthread + 1;
  
  if (bstate.nthread < nthread) {
    pthread_cond_wait(&bstate.barrier_cond, &bstate.barrier_mutex);
  } else { 
    bstate.round += 1;
    bstate.nthread = 0;
    pthread_cond_broadcast(&bstate.barrier_cond);
  } 
  
  pthread_mutex_unlock(&bstate.barrier_mutex);
}
```

The barrier function is called and tested in the following thread function:

```c
static void *
thread(void *xa)
{
  long n = (long) xa;
  long delay;
  int i;

  for (i = 0; i < 20000; i++) {
    int t = bstate.round;
    assert (i == t);
    barrier();
    usleep(random() % 100);
  }

  return 0;
}
```

```text
OK; passed
```

