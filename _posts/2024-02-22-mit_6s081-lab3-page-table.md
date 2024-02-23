---
layout: post
title: "[MIT_6.S081] Lab3: Page Table"
subtitle: "Page Table Implementation"
date: 2024-02-22
author: "Michael Xi"
header-img: "img/disk.jpg"
tags: [System, Operating System, MIT 6.S081]
---

# Speed up System Calls ([easy](https://pdos.csail.mit.edu/6.828/2021/labs/guidance.html))

## Understand the Question

Our goal is to speed up certain system calls (e.g. `getpid()`) by sharing data in a read-only region between user space and the kernel, so that the system calls no longer require kernel crossing and thus save the cost of context switching.

For this lab question, we want to implement a system call `ugetpid()` that is faster than the `getpid()`. The logic is that when each process is created, map one read-only page at **USYSCALL** (a virtual address defined in `memlayout.h`). At the start of this page, store a `struct usyscall` (also defined in `memlayout.h`), and initialize it to store the PID of the current process.

For this lab, `ugetpid()` has been provided on the userspace side and will automatically use the USYSCALL mapping.

```c
/* user/ulib.c */

int ugetpid(void) {
  struct usyscall *u = (struct usyscall *)USYSCALL;
  return u->pid;
}
```

```c
/* kernel/memlayout.h */

// the kernel expects there to be RAM
// for use by the kernel and user pages
// from physical address 0x80000000 to PHYSTOP.
#define KERNBASE 0x80000000L
#define PHYSTOP (KERNBASE + 128*1024*1024)

// map the trampoline page to the highest address,
// in both user and kernel space.
#define TRAMPOLINE (MAXVA - PGSIZE)

// map kernel stacks beneath the trampoline,
// each surrounded by invalid guard pages.
#define KSTACK(p) (TRAMPOLINE - (p)*2*PGSIZE - 3*PGSIZE)

// User memory layout.
// Address zero first:
//   text
//   original data and bss
//   fixed-size stack
//   expandable heap
//   ...
//   USYSCALL (shared with kernel)
//   TRAPFRAME (p->trapframe, used by the trampoline)
//   TRAMPOLINE (the same page as in the kernel)
#define TRAPFRAME (TRAMPOLINE - PGSIZE)
#ifdef LAB_PGTBL
#define USYSCALL (TRAPFRAME - PGSIZE)

struct usyscall {
  int pid;  // Process ID
};
#endif
```

## Solution

The overall solution logic is described below:

![SpeedUp_Syscall.png](https://s2.loli.net/2024/02/23/na28DIVLutfXY3B.png)



First, add the **usyscall struct** as a field to the `struct proc`, so that we can allocate space for that struct later.

```c
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID

  // wait_lock must be held when using this:
  struct proc *parent;         // Parent process

  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  struct trapframe *trapframe; // data page for trampoline.S
  
  struct usyscall *usyscall;   // 🌟 usyscall page for fast pid lookup
  
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
}
```

Second, edit the `allocproc()` in `proc.c` to allocate a page for the usyscall struct.

```c
static struct proc* allocproc(void) {
  struct proc *p;
	
  // Look in the process table for an UNUSED proc
  // If found, initialize state required to run in the kernel
  for(p = proc; p < &proc[NPROC]; p++) {
    acquire(&p->lock);
    if(p->state == UNUSED) {
      goto found;
    } else {
      release(&p->lock);
    }
  }
  return 0;

found:
  p->pid = allocpid();
  p->state = USED;

  // Allocate a trapframe page.
  if((p->trapframe = (struct trapframe *)kalloc()) == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }

  // 🌟 Allocate a USYSCALL page
  if((p->usyscall = (struct usyscall *)kalloc()) == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }
  // 🌟 Store the PID of the current process
  p->usyscall->pid = p->pid;

  // An empty user page table.
  p->pagetable = 🌟 proc_pagetable(p);
  if(p->pagetable == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }

  // Set up new context to start executing at forkret,
  // which returns to user space.
  memset(&p->context, 0, sizeof(p->context));
  p->context.ra = (uint64)forkret;
  p->context.sp = p->kstack + PGSIZE;

  return p;
}
```

Third, in `proc_pagetable`, **map** the usyscall struct to the USYSCALL user virtual address.

```c
pagetable_t proc_pagetable(struct proc *p) {
  pagetable_t pagetable;

  // An empty page table.
  pagetable = uvmcreate();
  if(pagetable == 0)
    return 0;

  // map the trampoline code (for system call return)
  // at the highest user virtual address.
  // only the supervisor uses it, on the way
  // to/from user space, so not PTE_U.
  if(mappages(pagetable, TRAMPOLINE, PGSIZE,
              (uint64)trampoline, PTE_R | PTE_X) < 0){
    uvmfree(pagetable, 0);
    return 0;
  }
  
  // map the trapframe just below TRAMPOLINE, for trampoline.S.
  if(mappages(pagetable, TRAPFRAME, PGSIZE,
              (uint64)(p->trapframe), PTE_R | PTE_W) < 0){
    uvmunmap(pagetable, TRAMPOLINE, 1, 0);
    uvmfree(pagetable, 0);
    return 0;
  }

  // 🌟 map the one read-only page to USYSCALL VA
  if (mappages(pagetable, USYSCALL, PGSIZE, (uint64)(p->usyscall), PTE_U | PTE_R) < 0) {
    printf("[proc.c] mappage USYSCALL failed\n");
    uvmunmap(pagetable, TRAMPOLINE, 1, 0);
    uvmunmap(pagetable, TRAPFRAME, 1, 0);
    uvmfree(pagetable, 0);
    return 0;
  }

  return pagetable;
}
```

Now, the `ugetpid()` can get the pid by directly reading the USYSCALL virtual address. Recall the following userspace code:

```c
int ugetpid(void) {
  struct usyscall *u = (struct usyscall *)USYSCALL;
  return u->pid;
}
```

Finally, don't forget to **free** the usyscall page in `freeproc()` and **free** the mapping in `proc_freepagetable()`.

```c
static void freeproc(struct proc *p) {
  if(p->trapframe)
    kfree((void*)p->trapframe);
	
  if(p->usyscall)
    kfree((void*)p->usyscall);
  
  p->trapframe = 0;
  if(p->pagetable)
    proc_freepagetable(p->pagetable, p->sz);
  p->pagetable = 0;
  p->sz = 0;
  p->pid = 0;
  p->parent = 0;
  p->name[0] = 0;
  p->chan = 0;
  p->killed = 0;
  p->xstate = 0;
  p->state = UNUSED;
}
```

```c
void proc_freepagetable(pagetable_t pagetable, uint64 sz) {
  uvmunmap(pagetable, TRAMPOLINE, 1, 0);
  uvmunmap(pagetable, TRAPFRAME, 1, 0);
  uvmunmap(pagetable, USYSCALL, 1, 0);
  uvmfree(pagetable, sz);
}
```


# Print a Page Table ([easy](https://pdos.csail.mit.edu/6.828/2021/labs/guidance.html))

## Understand the Question

Define a function called `vmprint()`. It should take a `pagetable_t` argument, and print that page table in the format described below. Insert `if(p->pid==1) vmprint(p->pagetable)` in `exec.c` just before the `return argc`, to **print the first process's page table**.


![pgtable.png](https://s2.loli.net/2024/02/22/yCQiORKqfWljN1s.png)


Example page table output:

```txt
page table 0x0000000087f6e000
 ..0: pte 0x0000000021fda801 pa 0x0000000087f6a000
 .. ..0: pte 0x0000000021fda401 pa 0x0000000087f69000
 .. .. ..0: pte 0x0000000021fdac1f pa 0x0000000087f6b000
 .. .. ..1: pte 0x0000000021fda00f pa 0x0000000087f68000
 .. .. ..2: pte 0x0000000021fd9c1f pa 0x0000000087f67000
 ..255: pte 0x0000000021fdb401 pa 0x0000000087f6d000
 .. ..511: pte 0x0000000021fdb001 pa 0x0000000087f6c000
 .. .. ..509: pte 0x0000000021fdd813 pa 0x0000000087f76000
 .. .. ..510: pte 0x0000000021fddc07 pa 0x0000000087f77000
 .. .. ..511: pte 0x0000000020001c0b pa 0x0000000080007000
```

1. `page table 0x0000000087f6e000` indicates the starting physical address of the page table itself (argument to `vmprint`).
2. `pte 0x0000000021fda801` is the page table entry itself. The value `21fda801` encodes information such as whether the page is present, writable, user-accessible, etc.
3. `pa 0x0000000087f6a000` is the physical address that this virtual page maps to.

## Solution

To print the **first process** of xv6, we need to add the `vmprint()` function call into the `exec` source code.

```c
int exec(char *path, char **argv) {
  // ... lots of code ...
  // Print first process's page table
  if (p->pid==1) vmprint(p->pagetable);  

  return argc;
}
```

The implementation of `vmprint` is in `vm.c`. This function is similar to the `freewalk` function.

```c
void printpagetable(pagetable_t pagetable, int depth) {
  // based on risv.h, each page table has 512 PTEs
  for (int i = 0; i < 512; i++) {
    pte_t pte = pagetable[i];
    
    // if pte is valid, print the current pte in specified format
    if (pte & PTE_V) {
      printf("..");
      for (int j = 0; j < depth; j++) {
        printf(" ..");
      }
      printf("%d: pte %p pa %p\n", i, pte, PTE2PA(pte));
    }

    // check if this PTE points to a lower-level page table
    // if so, keep recursively print the pagetable
    if ((pte & PTE_V) && (pte & (PTE_R | PTE_W | PTE_X)) == 0) {
      uint64 childpagetable = PTE2PA(pte);
      printpagetable((pagetable_t) childpagetable, depth + 1);
    }
  }
}

void vmprint(pagetable_t pagetable) {
  printf("page table %p\n", pagetable);
  printpagetable(pagetable, 0);
}
```

Finally, don't forget to define the prototype for `vmprint` in `kernel/defs.h`, so that this function can be recognized for calling.

```c
// vm.c
// ... more function prototypes ...
int             copyout(pagetable_t, uint64, char *, uint64);
int             copyin(pagetable_t, char *, uint64, uint64);
int             copyinstr(pagetable_t, char *, uint64, uint64);
void            vmprint(pagetable_t);
pte_t*          walk(pagetable_t, uint64, int)
```

The result looks like the following:

```
michaelxi@michaelxi-VirtualBox:~/xv6-labs-2021$ make qemu
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 3 -nographic -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0

xv6 kernel is booting

hart 2 starting
hart 1 starting
page table 0x0000000087f6e000
..0: pte 0x0000000021fda801 pa 0x0000000087f6a000
.. ..0: pte 0x0000000021fda401 pa 0x0000000087f69000
.. .. ..0: pte 0x0000000021fdac1f pa 0x0000000087f6b000
.. .. ..1: pte 0x0000000021fda00f pa 0x0000000087f68000
.. .. ..2: pte 0x0000000021fd9c1f pa 0x0000000087f67000
..255: pte 0x0000000021fdb401 pa 0x0000000087f6d000
.. ..511: pte 0x0000000021fdb001 pa 0x0000000087f6c000
.. .. ..509: pte 0x0000000021fdd813 pa 0x0000000087f76000
.. .. ..510: pte 0x0000000021fddc07 pa 0x0000000087f77000
.. .. ..511: pte 0x0000000020001c0b pa 0x0000000080007000
init: starting sh
$
```

# Detecting which Pages have been Accessed ([hard](https://pdos.csail.mit.edu/6.828/2021/labs/guidance.html))

## Understand the Question

We need to implement `pgaccess()`, a system call that reports which pages have been accessed using the `PTE_A` bit in the PTE Flags section.

The system call takes three arguments:

1. First, it takes the starting virtual address of the first user page to check. 
2. Second, it takes a number of pages to check.
3. Third, it takes a user address to a buffer to store the results into a bitmask.

## Solution

Recall Lab2 for how to implement a system call, we need the implementation for both the user and kernel space. The user space implementation of `pgaccess()` is provided, including adding function prototype in `/user/user.h` , `usys.pl`, etc.

In the kernel part, we need to first define the `PTE_A` bit in `riscv.h` based on RISC-V manual.

```c
#define PTE_V (1L << 0) // valid
#define PTE_R (1L << 1)
#define PTE_W (1L << 2)
#define PTE_X (1L << 3)
#define PTE_U (1L << 4) // 1 -> user can access
#define PTE_A (1L << 6) // 🌟 page access bit
```

Then, we need to write the `pgaccess()` implementation in `sys_pgaccess()` in `kernel/sysproc.c`.

```c
int sys_pgaccess(void) {
  // parse and validate the syscall arguments
  uint64 va_start;
  if(argaddr(0, &va_start) < 0)
    return -1;
  
  int npage;
  if(argint(1, &npage) < 0)
    return -1;

  uint64 ua_buffer;
  if(argaddr(2, &ua_buffer) < 0)
    return -1;
  
  // bitmask for marking which page is accessed
  uint64 bitmask = 0;
  struct proc *p = myproc();

  // iterate the pages and fill in bitmask
  for (int i = 0; i < npage; i++) {
    // get the pte of the corresponding page based on its virtual address
    pte_t *pte = walk(p->pagetable, va_start + i * PGSIZE, 0);    
    // check for the PTE_A bit
    if ((*pte) & PTE_A) {
      bitmask |= (1L << i);
      // [🌟] clear the access bit to pte
      (*pte) &= (~PTE_A);
		}
  } 

  // copy the bitmask to user space buffer using safe copyout
  copyout(p->pagetable, ua_buffer, (char *)&bitmask, sizeof(bitmask));
  return 0;
}
```

[🌟] Note that be sure to clear `PTE_A` after checking if it is set. Otherwise, it won't be possible to determine if the page was accessed since the last time `pgaccess()` was called (i.e., the bit will be set forever).
