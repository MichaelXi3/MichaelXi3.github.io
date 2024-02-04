---
layout: post
title: "Operating System Synchronization"
subtitle: "Concepts and Implementations"
date: 2024-01-30
author: "Michael Xi"
header-img: "img/post-bg-2015.jpg"
tags: [System, Operating System]
---

# Why Synchronization?

If I have a bank account and I just received two payments from my friends, my banking system sends two "add 100 dollars" instructions to update my bank account balance. These two additions are performed by two threads, both of which are attempting to update my bank account balance. Unfortunately, these two threads execute at the exactly same time (bad luck), resulting in simultaneous updates to my balance. In this case, the updates overlap and my balance only increases by 100 dollars. So sad!

This is the use case where we need synchronization for our banking system (in fact, nearly all systems). The goal of synchronization is to protect the critical resource, in this case, the balance amount, and ensure that the critical resource is modified properly. The code that updates the critical resource is called the critical section, and thus we apply synchronization techniques to the critical section.

Here are some more definitions to make things clear:

- **Critical Resource**: Shared resources that can be used by only one process at a time, i.e., mutually exclusive resources.
- **Critical Section**: A code segment used to access a shared resource (i.e., a critical resource) that is characterized by allowing only one process (or thread) to execute at any given time.
- **Mutual Exclusion**: For multiple code segments, only one can access the critical resource at the same time.
- **Synchronization**: Several code segments must be run strictly in some prescribed order; synchronization is a more complex form of mutual exclusion, where access to mutual exclusion is unordered, and synchronization must be run in order.
- **Race Condition**: A race condition arises if multiple threads of execution enter the critical section at roughly the same time; both attempt to update the shared data structure, leading to a surprising (and perhaps undesirable) outcome.

# How Synchronization?

## Software Mechanism

Initially, brilliant computer scientists were thinking very hard about how to achieve synchronization using pure software mechanisms. Dekker's Algorithm is the most famous one. The algorithm implementation is described below:

```C
process p1()
  while (true) {
    // non-critical code for process 1
    
    key1 = true;
    
    while (key2) {
      
    }
    
  }
```


## Low-Level Hardware Primitives



## High-Level Synchronization Primitives



## High-Level Synchronization Abstraction

### Monitor



# No Synchronization?

## Lock-Free Data Structure

CAS

# Synchronization Issues

## Deadlock



## Livelock
