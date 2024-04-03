---
layout: post
title: "[MIT_6.S081] Lab5: Copy on Write Fork"
subtitle: "COW Fork for xv6"
date: 2024-04-02
author: "Michael Xi"
header-img: "img/disk.jpg"
tags: [System, Operating System, MIT 6.S081]
---

# Implement Copy-on Write ([hard](https://pdos.csail.mit.edu/6.828/2021/labs/guidance.html))

## Understand the Question

The `fork()` system call in xv6 copies all of the parent process's user-space memory into the child. If the parent is large, copying can take a long time. Worse, the work is often largely wasted; for example, a `fork()` followed by `exec()` in the child will cause the child to discard the copied memory, probably without ever using most of it. On the other hand, if both parent and child use a page, and one or both write it, a copy is truly needed.

The goal of the copy-on-write (COW) fork() is to defer allocating and copying physical memory pages for the child until the copies are actually needed, if ever.

COW fork() creates just a pagetable for the child, with PTEs for user memory pointing to the parent's physical pages. COW fork() marks all the user PTEs in both parent and child as not writable. When either process tries to write one of these COW pages, the CPU will force a **page fault**. The kernel page-fault handler detects this case, allocates a page of physical memory for the faulting process, copies the original page into the new page, and modifies the relevant PTE in the faulting process to refer to the new page, this time with the PTE marked writeable. When the page fault handler returns, the user process will be able to write its copy of the page.

COW fork() makes freeing of the physical pages that implement user memory a little trickier. A given physical page may be referred to by multiple processes' pagetables, and should be freed only when the last reference disappears.

![cow.png](https://s2.loli.net/2024/04/03/M4TCzvs3VtRK7ly.png)

## Solution

We can tackle this question in the following steps:

**First, when we invoke the fork system call, the fork should do the COW logics.**

1. When `fork()` is called, do not allocate new pages for the child process. Instead, map the parent's physical page into the child. This modification requires changes in `uvmcopy()`. After mapping is done, clean the `PTE_W` flag on both the PTEs of parent and child process, so that the write is not allowed on shared pages. Then create and update the reference count per physical page for future releases/free usages.

   In `kernel/vm.c`, modify the `uvmcopy()` function:

   ```c
   extern int refcnt[];
   extern struct spinlock cnt_lock;
   
   int
   uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
   {
     pte_t *pte;
     uint64 pa, i;
     uint64 flags;
     
     for (i = 0; i < sz; i += PGSIZE) {
       if ((pte = walk(old, i, 0)) == 0)
         panic("uvmcopy: pte shuld exist");
       
       if (*pte & PTE_V == 0)
         panic("uvmcopy: page not present");
       
       // get the according physical address
       pa = PTE2PA(*pte);
       // get the parent pte flags
       flags = PTE_FLAGS(*pte);
       
       // set the flags to COW pte
       if (flags & PET_W) {
         flags = (flags | PTE_COW) & (~PTE_W);
         *pte = PA2PTE(pa) | flags;
       }
       
       // increment the reference count of page
       if ((uint64) pa % PGSIZE != 0 || (uint64) pa >= PHYSTOP)
         panic("uvmcopy: pa invalid");
       
       acquire(&cnt_lock);
       refcnt[(uint64) pa >> 12] += 1;
       release(&cnt_lock);
       
       // map the same pa to child process with COW flags
       if (mappages(new, i, PGSIZE, (uint64) pa, flags) != 0) {
         goto err;
       }
     }
     
     return 0;
     
     err:
     	uvmunmap(new, 0, i / PGSIZE, 1);
     	return -1;
   }
   ```

   In `kernel/riscv.h`, we defined a bit in PTE to mark the COW page:

   ```c
   #define PTE_COW            (1L << 8)
   #define GET_PTE_COW(pte)   (pte >> 8) & 1
   ```

2. The reference count for each physical page is tracked inside an array. The count should be initialized in `kalloc()` and should be decremented in `kfree()` and `freerange()`.

   In `kernel/kalloc.c`, we need to implement the reference count as follow:

   ```c
   struct spinlock cnt_lock;
   int refcnt[PHYSTOP >> 12];
   
   void
   kinit()
   {
     initlock(&cnt_lock, "cow");
     initlock(&kmem.lock, "kmem");
     freerange(end, (void*) PHYSTOP);
   }
   
   void
   freerange(void *pa_start, void *pa_end)
   {
     char *p;
     p = (char *) PGROUNDUP((uint64)pa_start);
     
     for (; p + PGSIZE <= (char *)pa_end); p += PGSIZE) {
       refcnt[(uint64) p >> 12] = 1;
       kfree(p);
     }
   }
   
   void
   kfree(void *pa)
   {
     struct run *r;
     
     // pa validation checks
     if (pa % PGSIZE != 0 || pa < end || pa > PHYSTOP)
       panic("kfree: invalid pa");
     
     int pgidx = pa >> 12; // pa divide by PGSIZE (4k)
     
     // kfree will release the page if ref_cnt is 1, o.w. decrement ref_cnt
     acquire(&cnt_lock);
     
     // case 1: last reference, release the page
     if (--refcnt[pgidx] == 0) {
       release(&cnt_lock);
       
       memset(pa, 1 PGSIZE);
       r = (struct run *) pa;
       
       acquire(&kmem.lock);
       r->next = kmem.freelist;
       kmem.freelist = r;
       release(&kmem.lock);
       
     // case 2: decrease the reference count only
     } else {
       release(&cnt_lock);
     }
   }
   
   void *
   kalloc(void)
   {
     struct run *r;
   
     acquire(&kmem.lock);
     r = kmem.freelist;
   
     /* first make sure the page is available */
     if(r) {
       kmem.freelist = r->next;
   
       /* second, increase the ref_cnt of new page to 1 */
       acquire(&cnt_lock);
       refcnt[(uint64)r >> 12] = 1;
       release(&cnt_lock);
     }
   
     release(&kmem.lock);
   
     if(r)
       memset((char*)r, 5, PGSIZE); // fill with junk
   
     return (void*)r;
   }
   ```

**Second, if the COW-fork page is modified, the following should happen.**

1. The access to the COW page will run into a page fault, and since the fault is from user space, the page fault is processed in `usertrap()`. Specifically, we need to handle the COW page by allocating a new page with `kalloc()` and copying the old page to the new page with the `PTE_W` bit set. We also need to update the reference count using `kfree()`.

   In `kernel/trap.c`, modify the `usertrap()` function:
   
   ```c
   void
   usertrap(void)
   {
     int which_dev = 0;
   
     if((r_sstatus() & SSTATUS_SPP) != 0)
       panic("usertrap: not from user mode");
   
     // send interrupts and exceptions to kerneltrap(),
     // since we're now in the kernel.
     w_stvec((uint64)kernelvec);
   
     struct proc *p = myproc();
     
     // save user program counter.
     p->trapframe->epc = r_sepc();
     
     if(r_scause() == 8){
       // system call
   
       if(p->killed)
         exit(-1);
   
       // sepc points to the ecall instruction,
       // but we want to return to the next instruction.
       p->trapframe->epc += 4;
   
       // an interrupt will change sstatus &c registers,
       // so don't enable until done with those registers.
       intr_on();
   
       syscall();
     } else if((which_dev = devintr()) != 0){
       // ok
   
     /* 🌟 handling the COW page fault */   
     } else if (r_scause() == 15 || r_scause() == 13) {
       uint64 va = r_stval();
       // pass the virtual address va to cow-handling
       if (va >= p->sz || cow_handling(p->pagetable, va) < 0) {
         p->killed = 1;
       }
     } else {
       printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
       printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
       p->killed = 1;
     }
   
     if(p->killed)
       exit(-1);
   
     // give up the CPU if this is a timer interrupt.
     if(which_dev == 2)
       yield();
   
     usertrapret();
   }
   ```
   
   In `kernel/vm.c`, add the `cow_handling()` function to allocate new page for COW page fault.
   
   ```c
   int
   cow_handling(pagetable_t pgtable, uint64 va)
   {
     // va validation check
     if (va >= MAXVA)
       return -1;
     
     // get the corresponding pte
     pte_t* pte = walk(pgtable, va, 0);
     if (pte == 0)
       return -1;
     
     // validation checks for pte
     if ((*pte & PTE_V) == 0 || (*pte & PTE_COW == 0) || *pte & PTE_U == 0)
       return -1;
     
     // get pa of pte and do the validation check
     uint64 pa = PTE2PA(*pte);
     if (pa == 0)
       return -1;
     
     // allocate a new page and copy the content of pa to it
     uint64 ka = (uint64) kalloc();
     if (ka == 0)
       return -1;
     memmove((char *) ka, (char*) pa, PGSIZE);
     
     // set the flags for the new page
     uint64 flags = PTE_FLAGS(*pte);
     flags = flags | PTE_W & (~PTE_COW);
     *pte = PA2PTE((uint64) ka) | flags;
     
     // decrement the ref_cnt to the old pa
     kfree((void *) pa);
     return 0;
   }
   ```

**Third, if the kernel tries to use `copyout()` to transfer a Copy-On-Write (COW) page from kernel space to user space, we need to handle it the same way as a COW page fault, as mentioned above.**

1. In `kernel/vm.c`, the implementation of `copyout()` is as follows:

   ```c
   // Copy from kernel to user.
   // Copy len bytes from src to virtual address dstva in a given page table.
   // Return 0 on success, -1 on error.
   int
   copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
   {
     uint64 n, va0, pa0;
   
     while(len > 0){
       va0 = PGROUNDDOWN(dstva);
       if (va0 > MAXVA)
         return -1;
   
       /* get the pte of va */
       pte_t* pte = walk(pagetable, va0, 0);
       if (pte == 0)
         return -1;
   
       /* 🌟 if the va is a COW page, process cow handling */
       if ((*pte) & PTE_COW) {
         if (cow_handling(pagetable, va0) < 0) {
            return -1; 
         }
       }
   
       /* update the pa after the cow handling */
       pa0 = PTE2PA(*pte);
   
       n = PGSIZE - (dstva - va0);
       if(n > len)
         n = len;
       memmove((void *)(pa0 + (dstva - va0)), src, n);
   
       len -= n;
       src += n;
       dstva = va0 + PGSIZE;
     }
     return 0;
   }
   ```

**Finally, the test results are displayed below.**

```
$ usertests
usertests starting
test MAXVAplus: OK
test manywrites: OK
test execout: OK
test copyin: OK
test copyout: OK
test copyinstr1: OK
test copyinstr2: OK
test copyinstr3: OK
test truncate1: OK
test truncate2: OK
test truncate3: OK
test reparent2: OK
test pgbug: OK
test sbrkbugs: usertrap(): unexpected scause 0x000000000000000c pid=3266
            sepc=0x00000000000058fa stval=0x00000000000058fa
usertrap(): unexpected scause 0x000000000000000c pid=3267
            sepc=0x00000000000058fa stval=0x00000000000058fa
OK
test badarg: OK
test reparent: OK
test twochildren: OK
test forkfork: OK
test forkforkfork: OK
test createdelete: OK
test linkunlink: OK
test linktest: OK
test unlinkread: OK
test concreate: OK
test subdir: OK
test fourfiles: OK
test sharedfd: OK
test dirtest: OK
test exectest: OK
test bigargtest: OK
test bigwrite: OK
test bsstest: OK
test sbrkbasic: OK
test sbrkmuch: OK
test kernmem: OK
test sbrkfail: OK
test sbrkarg: OK
test sbrklast: OK
test sbrk8000: OK
test validatetest: OK
test stacktest: OK
test opentest: OK
test writetest: OK
test writebig: OK
test createtest: OK
test openiput: OK
test exitiput: OK
test iput: OK
test mem: OK
test pipe1: OK
test killstatus: OK
test preempt: kill... wait... OK
test exitwait: OK
test rmdot: OK
test fourteen: OK
test bigfile: OK
test dirfile: OK
test iref: OK
test forktest: OK
test bigdir: OK
ALL TESTS PASSED

$ cowtest
simple: ok
simple: ok
three: ok
three: ok
three: ok
file: ok
ALL COW TESTS PASSED
```
