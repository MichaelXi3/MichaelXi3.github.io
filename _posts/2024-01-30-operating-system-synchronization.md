---
layout: post
title: "Operating System Synchronization"
subtitle: "Concepts and Implementations"
date: 2024-01-30
author: "Michael Xi"
header-img: "img/post-unix-cut.jpg"
tags: [System, Operating System]
---

# Why Synchronization?

Let's say you have a bank account and you just received two payments from your friends, your banking system sends two `add 100-dollar` instructions to update your bank account balance. These two additions are performed by two threads, both of which are attempting to update my bank account balance. Unfortunately, these two threads execute at the exactly same time (bad luck), resulting in simultaneous updates to your balance. In this case, the updates overlap and your balance only increases by 100 dollars. So sad!

This is the use case where we need synchronization for our banking system (in fact, nearly all systems). The goal of synchronization is to protect the critical resource, in this case, the balance amount, and ensure that the critical resource is updated properly. The code that updates the critical resource is called the critical section, and thus we apply synchronization techniques to the critical section.

Here are some more definitions to make things clear:

- **Critical Resource**: Shared resources that can be used by only one process at a time, i.e., mutually exclusive resources.
- **Critical Section**: A code segment used to access a shared resource (i.e., a critical resource) that is characterized by allowing only one process (or thread) to execute at any given time.
- **Mutual Exclusion**: For multiple code segments, only one can access the critical resource at the same time.
- **Synchronization**: Several code segments must be run strictly in some prescribed order; synchronization is a more complex form of mutual exclusion, where access to mutual exclusion is unordered, and synchronization must be run in order.
- **Race Condition**: A race condition arises if multiple threads of execution enter the critical section at roughly the same time; both attempt to update the shared data structure, leading to a surprising (and perhaps undesirable) outcome.

# How Synchronization?

## Software Mechanism

Initially, brilliant computer scientists were thinking very hard about how to achieve synchronization using only software mechanisms. **Dekker's Algorithm** is the most famous one. The algorithm implementation is described below:

```c
process p1() {
  while (true) {
    // non-critical code for process 1
    
    key1 = true;
    
    while (key2) {
    	if (turn == 2) {
        key1 = false;
        
        while (turn == 2);
        
        key1 = true;
      }
    }
    
    // critical section
    
    turn = 2;
    key1 = false;
    
    // non-critical code for process 1
  }
}
```

```c
process p2() {
  while (true) {
    // non-critical code for process 2
    
    key2 = true;
    
    while (key1) {
    	if (turn == 1) {
        key2 = false;
        
        while (turn == 1);
        
        key2 = true;
      }
    }
    
    // critical section
    
    turn = 1;
    key2 = false;
    
    // non-critical code for process 2
  }
}
```

Note that Dekker's algorithm is one of the first solutions to the mutual exclusion problem for two processes or threads. The mutual exclusion solution for more processes will be much more complicated.

This algorithm uses three **shared variables**:

- `key1` and `key2` are boolean flags that indicate the **intention** of processes 1 and 2 to enter the critical section.
- `turn` is used to indicate whose turn it should be for entering the critical section

The **procedure** of Dekker's algorithm is:

- Both processes set their flag (`key1` and `key2`) to true to express the intent to enter the critical section.
- Then the algorithm checks the other process's intent
  - If the other process's intent or flag is `false`, it proceeds to the critical section directly
  - If the other process's intent or flag is true, it checks the `turn` variable to decide the order of execution
    - If it's the current process's turn, the other process will set its intention to false shortly and thus the current process can proceed to the critical section.
    - If it's the other process's turn, the process sets its intention flag (`key1` for P1, `key2` for P2) to `false` and waits in a loop.

The Dekker's algorithm effectively ensured mutual exclusion, progress, and bounded waiting. It demonstrated that mutual exclusion can be achieved without special hardware instructions. However, Dekker's algorithm also has **limitations** that restrict it from actual implementation.

- **Complexity at Scale**: Dekker's algorithm only applies to the mutual exclusion between two processes. The mutual exclusion algorithm for more than two processes involves significantly more complexity, limiting its scalability.
- **Busy Wait**: While one process is executing the critical section, the other process is in busy wait, which is a waste of limited CPU resources.
- **Shared Variable**: Dekker's algorithm must maintain the correct updates and reads of three shared variables, which is very likely to go wrong in reality.

Given these disadvantages, computer science researchers started to leverage hardware to achieve performant mutual exclusion and synchronization.

## Low-Level Hardware Primitives

Low-level hardware primitives for synchronization provide an important characteristic: **Atomicity** of operations. The hardware level atomicity means that the operations are either done or undone, which is the fundamental building block of high-level synchronization primitives such as lock and semaphore.

There are primarily four types of hardware primitives: **TestAndSet (TAS)**, **Swap**, **CompareAndSwap (CAS)**, and **FetchAndAdd (FAA)**.

### TestAndSet (TAS)

TAS is an atomic operation that first reads the value of a memory location and immediately sets a new value, then returns the old value. If more than one thread executes TAS at the same time, only one thread can successfully change the value, while the other threads will see the old value. 

![TAS.png](https://s2.loli.net/2024/02/05/iRWy6X1HlYOkpvM.png)

The **use case** of TAS is for multiple threads to compete and acquire the lock. 

If a thread tries to acquire a lock and sees a return value of 0 (unlocked), it knows it has successfully acquired the lock. If the return value is 1 (locked), then it knows that another thread already holds the lock.

![TAS_Lock.png](https://s2.loli.net/2024/02/05/yYrdBhSa4otZG2z.png)

The disadvantage of TAS is **busy waiting**, as all the waiting threads have to spin empty cycles.

### Swap

Swap is an atomic operation that exchanges the values of two memory locations.

```c
void swap(int *x, int *y) {
    int temp = *x;
    *x = *y;
    *y = temp;
}
```

Usage Scenario: Swap can be used to acquire locks. A thread can try to acquire a lock by swapping the lock variable with a local variable. If the result of the local variable indicates that the lock is available (e.g., 0), then the thread knows that it has successfully acquired the lock.

```java
boolean lock = false;  // Global variable

process p1(boolean myKey) {
  while (true) {
    // Non-critical code
    mykey = true;
    do {
      swap(&lock, &myKey);
    } while (myKey == true);
    
    // Critical section
    lock = false;
    
    // Non-critical code
  }
}

process p2(boolean myKey) {
  while (true) {
    // Non-critical code
    mykey = true;
    do {
      swap(&lock, &myKey);
    } while (myKey == true);
    
    // Critical section
    lock = false;
    
    // Non-critical code
  }
}
```

The disadvantage of Swap is also **busy waiting**, as all the waiting threads have to spin empty cycles.

### CompareAndSwap (CAS)

The Compare-And-Swap (CAS) operation atomically checks if a memory location's current value matches an expected value, and only if they match, it updates that location to a new value. This ensures the shared variable is updated by the current thread only if it hasn't been modified by other threads in the meantime.

![CAS.png](https://s2.loli.net/2024/02/05/cXNFD37WrqQaz5O.png)

> Note that TAS is ultimately competing for locks, whereas CAS is constantly trying to update the values of shared variables until it succeeds. Thus, CAS can be used to implement **lock-free synchronization**, and that's why CAS is more powerful!

The downside of CAS is also **busy waiting**, as it requires spinning empty cycles.

## High-Level Synchronization Primitives

The low-level hardware primitives provide the atomic operations necessary for implementing high-level synchronization primitives, which provide more intuitive and abstract interfaces for managing access to shared resources among threads. The common high-level synchronization primitives include **Semaphore**, **Mutex**, and **Condition Variable**.

### Semaphore

A semaphore is an integer value that can be atomically increased or decreased by two operations, **P** (Proberen, or attempted decrease) and **V** (Verhogen, or increase), with P decreasing and V increasing. The advantage is that threads do not idle (No Busy Wait), they are put directly into a queue and Block Wait.

The goal of the semaphore is to **manage limited resources**, and there are two types of semaphores: Counting Semaphore and Binary Semaphore.

**Binary Semaphore**: A binary semaphore has only two values, usually 0 and 1. It is similar to a mutex lock in terms of behavior. However, the mutex lock can be acquired by a thread, whereas semaphore is just a shared variable between multiple threads.

```java
class BinarySemaphore {
  int value = 1;
  Queue processes = new Queue(); // Queue to hold waiting processes
  
  void P() {
    if (value == 1) {
      value = 0;
    } else {
      processes.enqueue(curProcess);
      block();
    }
  }
  
  void V() {
    if (!processes.isEmpty()) {
      Process p = processes.dequeue();
      wakeup(p);
    } else {
      value = 1;
    }
  }
}
```

**Counting Semaphore**: A counting semaphore can have a value with a range from 0 to some maximum integer value. The initial semaphore value represents the amount of limited resources available at the very beginning. Thus it's used to control access to limited resources.

```java
class CountingSemaphore {
  int value;                     // Initial value can be any non-negative integer
  Queue processes = new Queue(); // Queue to hold waiting processes
  
  CountingSemaphore(int initialValue) {
    value = initialValue;
  }
  
  void P() {
    value = value - 1;
    if (value < 0) {
      processes.enqueue(curProcess);
      block();
    }
  }
  
  void V() {
    value = value + 1;
    if (value <= 0) {
      Process p = processes.dequeue();
      wakeup(p);
    }
  }
}
```

### Mutex

A mutex is used to ensure mutually exclusive access to the critical sections of code. As we mentioned previously, mutex has similar behavior to binary semaphore, but with additional constraints:

1. **Ownership**: Only the thread that has acquired the mutex lock can release it, whereas semaphore is a shared variable.

   ```c
   typedef struct {
       int locked;        // 0 if unlocked, 1 if locked
       pthread_t owner;   // Thread ID of the owner, if locked
   } pthread_mutex_t;
   ```

2. **Non-Recursive**: By default, attempting to lock a mutex that is already locked by the same thread leads to deadlock unless the mutex is explicitly declared as recursive.

There are several ways of **implementing the Mutex**, from primitive ideas to practical solutions:

1. **Disable the Interrupt**

   An intuitive thought for ensuring mutual exclusion is to disable the interrupt while one thread is executing so that the current thread can have exclusive access to the critical resources. However, disabling the interrupt has numerous disadvantages:

   - Granting user threads the ability to disable interrupts is dangerous. Because it could lead to the operating system losing control due to infinite loops in user processes.
   - This solution is only effective in <u>single-CPU environments</u>. In multi-CPU setups, other CPUs can still access the critical section, making this approach ineffective.
   - There's a risk of losing interrupt signals, which can be critical for timely hardware communication. E.g. losing I/O interrupts.
   - It tends to be inefficient due to the overhead of disabling and enabling interrupts.

   Therefore, this method is only applicable to the use of the operating system itself, when it needs atomicity.

   **SideNote**: To solve the issue of "only applying to the single-CPU environment", one intuitive solution is to lock the memory bus entirely. So that when one process is using memory, another process cannot access the memory, even if it is another CPU. (Not recommended)

2. **SpinLock**

   SpinLock is a synchronization mechanism used to protect access to critical sections in a multi-threaded environment. It is often used in scenarios where a thread holds a lock for a very short period of time, while the waiting threads engage in busy waiting for the lock.

   The **performance** of spinLock is better in multiprocessor scenarios than in single-processor scenarios, e.g., where Thread A is on CPU 1 and Thread B is on CPU 2. If thread A holds the lock and releases it quickly, thread B will probably be able to acquire the lock directly at the time of the spin, without wasting an entire clock cycle! The ideal situation is: number of threads = CPU core counts.

   The **advantage** of SpinLock is that it avoids the overhead of OS rescheduling and context switching. However, the **disadvantage** of SpinLock is its performance. In a uniprocessor situation, if the lock is already held by another thread, the current thread needs to idle out the entire CPU time slice when attempting to acquire the lock.

   There is also **Priority Invasion** problem when using SpinLock. When a low-priority thread holds a lock while a high-priority thread waits for the lock, it causes the high-priority thread to fail to execute in time. A common solution is **Priority Inheritance**, which means that the low-priority thread holding the lock temporarily inherits the priority of the high-priority thread waiting for the lock, thus reducing the waiting time of the high-priority thread.

   There are primarily two methods of **implementing SpinLock**: **CAS** and **TAS**.

   - **CAS (Compare-And-Swap)**: This atomic operation checks the value at a memory location and only updates it if it matches a given expected value. This is useful for implementing lock-free algorithms, including spinlocks.

     ```c
     struct SpinLock {
     		volatile int lock = 0;  // 0 means unlocked, 1 means locked
     };
     
     procedure lock(struct SpinLock* spin) {
     		while true {
     				if (CAS(&spin.lock, 0, 1) == 0)
     						break;          // Lock acquired
         }
     }
     
     procedure unlock(struct SpinLock* spin):
     		spin.lock = 0;
     ```

   - **TAS (Test-And-Set)**: This atomic operation tests the value of a bit and sets it to 1 (or true) atomically. If the bit is already set, the operation indicates that the lock is already taken. Implementations using TAS for spinlocks involve a thread repeatedly checking (testing) and setting this bit until it successfully acquires the lock.

     ```c
     struct SpinLock {
         volatile int lock;  // 0 means unlocked, 1 means locked
     };
     
     void lock(struct SpinLock* spin) {
         while (TAS(&spin->lock, 1) == 1);
     }
     
     void unlock(struct SpinLock* spin) {
         spin->lock = 0;
     }
     ```

3. **SpinLock with Yield**

   When a thread attempts to acquire this type of mutex and finds it already locked, instead of continuously spinning, it yields its execution (put thread from running state to **ready state**), allowing the scheduler to run other threads.

   ![SpinLock_Yield.png](https://s2.loli.net/2024/02/07/idfEUXBbyMgFvAw.png)


   This approach is used to avoid the high CPU usage associated with spinLock. It’s a <u>middle ground between busy waiting and fully sleeping</u>. The system behavior is described below:

   - **CPU Usage**: Although the `yield` operation avoids consuming too much CPU usage as in busy waiting, the threads in the ready state are still within the consideration of the scheduler, thus requiring CPU resources. Blocking threads, on the other hand, do not consume CPU resources at all.
   - **Context Switching Overhead**: Yielding can lead to more frequent context switches because a yielded thread can be quickly rescheduled. This might increase the load on the scheduler, especially in high contention scenarios with many threads.

4. **Blocking Mutex**

   When a thread attempts to acquire a blocking mutex and it’s already locked, the thread is put into a waiting or blocked state, not consuming CPU cycles. The thread is placed in a queue of threads waiting for the mutex to become available.

   Blocking mutexes are suitable for situations where the lock may be held for longer periods, as they free up the CPU to perform other tasks while the thread is waiting.
   
   ![Blocking_Mutex.png](https://s2.loli.net/2024/02/07/dgRuQhcTEfNlP3V.png)
   
5. **Adaptive Mutex**

   Adaptive Mutex is an idea that adjusts the lock to specific situations. For example, the `synchronized` keyword in Java performs a biased lock at first and then spinLock to busy-wait; if it can't acquire the lock after several attempts, it switches to blocking mutex to put the thread in the sleeping state.

   In general, the choice between a spinlock or a mutex depends on the **tradeoff** between the cost of "keeping the thread busy-waiting for multiple CPU cycles until it acquires the lock" and "putting the thread to sleep, waking it up, and making related system calls".

### Condition Variable

A **condition variable** is an **explicit queue** that threads can put themselves on when some state of execution (i.e., some condition) is not as desired (by waiting on the condition); some other thread, when it changes said state, can then **wake** one (or more) of those waiting threads and thus allow them to continue (by signaling on the condition).

- **Waiting on a Condition**: A thread can wait on a condition variable if the condition it requires is not yet true. While waiting, the thread is not consuming CPU resources.
- **Signaling a Condition**: Another thread can signal the condition variable once it has changed the state that affects the condition. This signal then wakes one or more (broadcast) waiting threads.
- **Mutex Association**: Condition variables are used in conjunction with mutexes. A thread must acquire a mutex before it can wait on a condition variable, ensuring that checking the condition and entering the wait state is an atomic operation. When the thread waits, it atomically releases the mutex and enters the sleeping state. Upon being awakened, it re-acquires the mutex before continuing execution.

Condition Variable Example: **Producer and Consumer**

```c
int buffer[BUFFER_SIZE];
int count = 0;

pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t not_empty = PTHREAD_COND_INITIALIZER;
pthread_cond_t not_full = PTHREAD_COND_INITIALIZER;

void* producer(void* arg) {
    for (int i = 0; i < 20; ++i) {
        pthread_mutex_lock(&mutex);
        
        // Wait while the buffer is full
        while (count == BUFFER_SIZE) {
            pthread_cond_wait(&not_full, &mutex);
        }
        
        buffer[count++] = i;
        
        pthread_cond_signal(&not_empty);
        
        pthread_mutex_unlock(&mutex);
        
        sleep(1);
    }
    exit(0);
}

void* consumer(void* arg) {
    for (int i = 0; i < 20; ++i) {
        pthread_mutex_lock(&mutex);
        
        // Wait while the buffer is empty
        while (count == 0) {
            pthread_cond_wait(&not_empty, &mutex);
        }
        
        int item = buffer[--count];
        
        pthread_cond_signal(&not_full);
        
        pthread_mutex_unlock(&mutex);
        
        sleep(1);
    }
    exit(0);
}
```

## High-Level Synchronization Abstraction

### Monitor

Monitor is a synchronization construct (High-Level Language Construct) that allows threads to have both **mutual exclusion** and the ability to **wait (block) for a condition**. Unlike other synchronization primitives such as semaphores and mutexes, monitor **encapsulates** conditional synchronization and mutual exclusion to provide a higher level of abstraction. The core benefits are encapsulation, security, and ease of use.

There are **four components** of the monitor.

1. **Mutual Exclusion Lock**

   It ensures that only one thread can execute the critical section in the monitor at any given moment.

2. **Condition Variables**

   It provides the ability of conditional synchronization, i.e. `wait` and `signal`.

3. **Shared Data**

   This is the data or resource that the thread needs to access and modify. The monitor encapsulates access to this shared data, ensuring data integrity and consistency when accessing and modifying these shared resources.

4. **Procedures/Methods**

   Methods defined within the monitor are used to perform operations on shared resources. Since these methods run under the control of the monitor, they automatically obey the rules of mutex locking.

**How does Monitor work?**

- When a thread calls a method (Procedure) within a Monitor, it needs to acquire a mutex lock first, and if the lock is already held by another thread, the thread will wait until the lock is available.
- Once the lock is acquired, the thread can execute the code within the method. If it needs to wait for a condition, it can wait on the condition variable, which releases the mutex lock and allows other threads to access the monitor to execute the code.
- When the condition becomes true, other threads can send notifications on the condition variable, waking up one or more waiting threads.
- When the method has finished executing, the thread releases the mutex lock, allowing other threads to access the monitor.

**How to implement a Monitor?**

Monitor is a <u>conceptual framework</u> for synchronization mechanisms. It provides a way to organize and manage access to shared resources to ensure data consistency and thread safety in a concurrent environment. Therefore, the implementation of a monitor varies from language to language, as long as it meets the main features of a monitor: **mutually exclusive access**, **conditional synchronization**, and **encapsulation**.

For example, Monitor can be used to solve the <u>Reader and Writer Problem</u>.

```java
monitor read_write {
  
  int readers = 0;
  bool writing = false;
	Lock mutex;
  
  condition okToRead, okToWrite;

  procedure startRead() {
      if (writing) {
          okToRead.wait();
      }
      readers++;
      okToRead.signal();
  }

  procedure endRead() {
      readers--;
      if (readers == 0) {
          okToWrite.signal();
      }
  }

  procedure startWrite() {
      if (readers != 0 || writing) {
          okToWrite.wait();
      }
      writing = true;
  }

  procedure endWrite() {
      writing = false;
      if (nonempty(okToRead)) {
          okToRead.signal();
      } else {
          okToWrite.signal();
      }
  }
}
```

```java
process reader() {
  while (true) {
    read_write.startread();
    // READ operation
    read_write.endread();
  }
}

process writer() {
  while (true) {
    read_write.startwrite();
    // WRITE operation
    read_write.endwrite();
  }
}
```

- Shared Variable
  - `readers`: keeps track of the number of readers currently reading the shared resource
  - `writing`: a Boolean indicating whether a writer is currently writing to the shared resource
- Condition Variables
  - `oktoread`: manages the number of readers waiting to read.
  - `oktowrite`: manages writers waiting to write.
- Procedures
  - `startread`: the reader calls this method before starting to read. If a writer is writing (writing is true), the reader will wait. Otherwise, the number of readers is increased and other readers are notified.
  - `endread`: Called when reading is complete. Decreases the number of readers and notifies the writer if there are no readers left.
  - `startwrite`: Called by the writer before starting to write. If there are active readers or another writer, the writer waits.
  - `endwrite`: Called when writing is complete. Sets writing to false and notifies the reader or writer as appropriate, with priority given to the reader.

# Concurrency Problems

Implementing synchronization can be challenging, and it is common to encounter bugs in concurrent programming. Interestingly, a significant portion of synchronization bugs is related to **Deadlock**, while other types of bugs include **atomicity violation** and **order violation**, among others. Below is a summary of the bugs Lu and colleagues studied.

![Sync_Bug.png](https://s2.loli.net/2024/02/07/dbG9eZJBsACX3mt.png)

## Deadlock

Deadlock is defined as a situation in which two or more processes (or threads) stop executing because they wait **indefinitely** for resources held by each other. 

![Deadlock.png](https://s2.loli.net/2024/02/07/oKOa42zJS7Rbg8Z.png)

### Deadlock Conditions

There are four conditions required for deadlock.

1. **Mutual Exclusion**

   At least one resource must be in unshared mode, i.e., only one process can use it at a time, and if another process requests the resource, the requester can only wait until the resource is released.

2. **Hold and Wait**

   A process holds at least one resource and is waiting to acquire additional resources held by other processes (i.e., it will not release them voluntarily).

3. **No Preemption**

   A resource that has been allocated to a process cannot be forcibly preempted, but can only be released voluntarily by the process holding it (i.e., it is not released passively).

4. **Circular Wait**

   There must be a collection of processes (at least two) in which each process is waiting for a resource held by the next process, forming a chain of circular waits.

### DeadLock Prevention

To Prevent the deadlock, we need to break at least one of the four coniditons.

1. **Breaks the mutex condition**

   Allows resources to be shared by multiple processes. However, this does not work for resources that cannot be shared.

2. **Breaking the hold-and-wait condition**

   Uses a resource pre-allocation policy, where processes request all the resources they need at once before running or require processes to release currently held resources before requesting new resources. However, it's hard to predict all the required resources, and it will leads to low resource utilization and reduced system concurrency.

3. **Breaking non-preemption condition**

   Allows processes to forcibly seize resources held by other processes, but this may lead to system performance degradation.

4. **Breaking cyclic waiting condition**

   Avoids circular waiting by requiring processes to apply for resources in ascending order of number through resource numbering.

### DeadLock Avoidance

To Avoid the deadlock, we can use clever scheduling.

Avoidance requires having a <u>global understanding</u> of the entire set of tasks that must run and locks that different threads might acquire during their execution. It then schedules these threads in a manner that ensures no deadlock can occur. However, the cost is limited concurrency and thus degraded performance.

A famous example of such an algorithm is **Dijkstra's Banker's Algorithm**. It is called the "banker's" algorithm because it is similar to how banks decide whether or not to grant a loan: the bank's goal is to ensure that at any given time, even if all customers draw the maximum amount of loan they may need, their minimum cash needs will be met. The algorithm is able to dynamically manage and allocate resources while ensuring that the system does not enter a deadlock state due to misallocation of resources. Here are the steps of the banker's algorithm:

1. **Initialize resources**: when the system is initialized, the total amount of each resource needs to be known.
2. **Declaring maximum requirements**: each process declares the maximum amount of each resource it may need.
3. **Resource requests and security checks**: When a process requests a set of resources, the system first performs a security check based on the banker's algorithm to determine whether the system can be kept in a secure state after allocating the requested resources. This check consists of verifying that there is a hypothetical sequence of resource allocations according to which all processes are eventually completed successfully.
4. **Allocating resources**: if the system is in a safe state, resources are allocated; otherwise, the process must wait.
5. **Completion and release**: processes complete and release their resources, which the system returns to the pool of available resources.

## DeadLock Detection

To Detect the deadlock, we can use the **resource graph**.

![resource_graph.png](https://s2.loli.net/2024/02/07/wAMhKFQvWNOUXED.png)

Specifically, we can draw a Resource Graph to detect the presence of loops (a Resource Graph is a directed graph where nodes can be processes or resource types and edges represent requested or occupied resources). If the resource graph contains loops after graph simplification, then deadlock exists.

## Livelock

Two threads could both be repeatedly attempting a lock requisition sequence and repeatedly failing to acquire both locks. In this case, both systems are running through this code sequence over and over again (and thus it is not a deadlock), but progress is not being made, hence the name livelock.



> “Simplicity is a great virtue but it requires hard work to achieve it and education to appreciate it. And to make matters worse: complexity sells better.”
>
> ― **Edsger Wybe Dijkstra**
