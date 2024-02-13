---
layout: post
title: "[MIT_6.S081] Lab2: System Calls"
subtitle: "System Calls Implementation"
date: 2024-02-12
author: "Michael Xi"
header-img: "img/disk.jpg"
tags: [System, Operating System]
---

# System Call Tracing ([moderate](https://pdos.csail.mit.edu/6.828/2021/labs/guidance.html))

## Understand the Question

The goal of this lab is to add a system call tracing feature that helps users to debug. Specifically, implement a new `trace` system call that traces whatever types of syscalls specified by the argument.

For example, to trace the fork system call, a program calls `trace(1 << SYS_fork)`, where `SYS_fork` is a syscall number from `kernel/syscall.h`. You have to modify the xv6 kernel to print out a line when each system call is about to return, if the system call's number is set in the mask.

Example outputs:

```bash
# trace SYS_read, since 32 = 1 << 5
$ trace 32 grep hello README
3: syscall read -> 1023
3: syscall read -> 966
3: syscall read -> 70
3: syscall read -> 0

# trace all syscalls, all flags on
$ trace 2147483647 grep hello README
4: syscall trace -> 0
4: syscall exec -> 3
4: syscall open -> 3
4: syscall read -> 1023
4: syscall read -> 966
4: syscall read -> 70
4: syscall read -> 0
4: syscall close -> 0

# trace nothing
$ grep hello README

# trace SYS_fork and its children
$ trace 2 usertests forkforkfork
usertests starting
test forkforkfork: 407: syscall fork -> 408
408: syscall fork -> 409
409: syscall fork -> 410
410: syscall fork -> 411
409: syscall fork -> 412
410: syscall fork -> 413
409: syscall fork -> 414
411: syscall fork -> 415
...
```

## Step-by-Step Solution

### User Space Implementation

- `trace.c` in `/user`, the entry point of `trace` command execution, using `trace` syscall to implement the actual trace logic

  ```c
  #include "kernel/param.h"
  #include "kernel/types.h"
  #include "kernel/stat.h"
  #include "user/user.h"
  
  int main(int argc, char *argv[]) {
    int i;
    char *nargv[MAXARG];
  
    if(argc < 3 || (argv[1][0] < '0' || argv[1][0] > '9')){
      fprintf(2, "Usage: %s mask command\n", argv[0]);
      exit(1);
    }
  	
    /* trace takes an int mask as argument */
    if (trace(atoi(argv[1])) < 0) {
      fprintf(2, "%s: trace failed\n", argv[0]);
      exit(1);
    }
    
    for(i = 2; i < argc && i < MAXARG; i++){
      nargv[i-2] = argv[i];
    }
    exec(nargv[0], nargv);
    exit(0);
  }
  ```

- In `/user/user.h` , add `trace`, so that user space programs can identify `trace` system call

  ```c
  struct stat;
  struct rtcdate;
  
  // system calls
  int fork(void);
  int exit(int) __attribute__((noreturn));
  int wait(int*);
  int pipe(int*);
  int write(int, const void*, int);
  int read(int, void*, int);
  int close(int);
  int kill(int);
  int exec(char*, char**);
  int open(const char*, int);
  int mknod(const char*, short, short);
  int unlink(const char*);
  int fstat(int fd, struct stat*);
  int link(const char*, const char*);
  int mkdir(const char*);
  int chdir(const char*);
  int dup(int);
  int getpid(void);
  char* sbrk(int);
  int sleep(int);
  int uptime(void);
  /* add trace syscall to user.h header */
  int trace(int syscall_num);
  ```

- In perl script `user/usys.pl` add `trace`, so that it produces `user/usys.S`, the actual **system call stubs**

  - `usys.pl`

    ```perl
    #!/usr/bin/perl -w
    
    # Generate usys.S, the stubs for syscalls.
    
    print "# generated by usys.pl - do not edit\n";
    
    print "#include \"kernel/syscall.h\"\n";
    
    sub entry {
        my $name = shift;
        print ".global $name\n";
        print "${name}:\n";
        print " li a7, SYS_${name}\n";
        print " ecall\n";
        print " ret\n";
    }
    	
    entry("fork");
    entry("exit");
    entry("wait");
    ...
    ...
    entry("trace");  /* add trace here */
    ```
  
  - `usys.S`
  
    ```assembly
    # generated by usys.pl - do not edit
    #include "kernel/syscall.h"
    .global fork
    fork:
     li a7, SYS_fork
     ecall
     ret
    .global exit
    exit:
     li a7, SYS_exit
     ecall
     ret
    .global wait
    ...
    ...
    .global trace
    trace:
     li a7, SYS_trace
     ecall
     ret
    ```
  
    This assembly code sets up the system call number for the `trace` service in the `a7` register, invokes the system call using the **`ecall`** instruction, and then returns to the caller.
  
    - `li a7, SYS_trace`: Load the `SYS_trace` syscall number immediately to `a7` register, and then **`ecall` to enter the kernel mode**

### Kernel Space Implementation

- Add a syscall number to `kernel/syscall.h` as an identifier to the new system call

  ```c
  // System call numbers
  #define SYS_fork    1
  #define SYS_exit    2
  #define SYS_wait    3
  #define SYS_pipe    4
  #define SYS_read    5
  #define SYS_kill    6
  #define SYS_exec    7
  #define SYS_fstat   8
  #define SYS_chdir   9
  #define SYS_dup    10
  #define SYS_getpid 11
  #define SYS_sbrk   12
  #define SYS_sleep  13
  #define SYS_uptime 14
  #define SYS_open   15
  #define SYS_write  16
  #define SYS_mknod  17
  #define SYS_unlink 18
  #define SYS_link   19
  #define SYS_mkdir  20
  #define SYS_close  21
  /* add new syscall number here */
  #define SYS_trace  22
  ```

- Add a `sys_trace()` function in **`kernel/sysproc.c` (Process-related System Calls)** that implements the new system call by remembering its argument in a new variable in the `proc` structure (see `kernel/proc.h`)

  - In `kernel/proc.h`, add a variable to the proc struct to store the **trace** mask

    ```c
    // Per-process state
    struct proc {
      struct spinlock lock;
    
      // p->lock must be held when using these:
      enum procstate state;        // Process state
      void *chan;                  // If non-zero, sleeping on chan
      int killed;                  // If non-zero, have been killed
      int xstate;                  // Exit status to be returned to parent's wait
      int pid;                     // Process ID
      int trace;                   // 🌟 Trace system call identifier
    
      // wait_lock must be held when using this:
      struct proc *parent;         // Parent process
    
      // these are private to the process, so p->lock need not be held.
      uint64 kstack;               // Virtual address of kernel stack
      uint64 sz;                   // Size of process memory (bytes)
      pagetable_t pagetable;       // User page table
      struct trapframe *trapframe; // data page for trampoline.S
      struct context context;      // swtch() here to run process
      struct file *ofile[NOFILE];  // Open files
      struct inode *cwd;           // Current directory
      char name[16];               // Process name (debugging)
    };
    ```

  - In `kernel/sysproc.c`, implement the `sys_trace` function to **set the trace mask** to proc struct.

    ```c
    #include "types.h"
    #include "riscv.h"
    #include "defs.h"
    #include "date.h"
    #include "param.h"
    #include "memlayout.h"
    #include "spinlock.h"
    #include "proc.h"
    
    uint64
    sys_trace(void)
    {
      int n;
      /* get and validate the syscall argument using `argint` */
      if (argint(0, &n) < 0)
        return -1;
      
      /* set the trace mask of the current process by setting the proc struct's trace field */
      myproc()->trace = n;
      
      printf("[sysproc.c] sys_trace triggered\n");
      return 0;    
    }
    /* other sys_syscalls functions... */
    ```
  
  - In `kernel/syscall.c`, modify the **system call dispatcher** function to **check the trace mask** and print information if the current syscall is traced
  
    ```c
    #include "types.h"
    #include "param.h"
    #include "memlayout.h"
    #include "riscv.h"
    #include "spinlock.h"
    #include "proc.h"
    #include "syscall.h"
    #include "defs.h"
    
    const char* syscallNames[] = {
        "unknown", // Placeholder for index 0, as system call start from 1
        "fork",
    		......
        "trace"
    };
    
    const char* sysname(int syscallId) {
        int numSyscalls = sizeof(syscallNames) / sizeof(char*);
        if(syscallId > 0 && syscallId < numSyscalls) {
            return syscallNames[syscallId];
        }
        return "unknown";
    }
    
    extern uint64 sys_close(void);
    ......
    extern uint64 sys_trace(void);
    
    static uint64 (*syscalls[])(void) = {
    [SYS_fork]    sys_fork,
    [SYS_exit]    sys_exit,
    ......
    [SYS_close]   sys_close,
    [SYS_trace]   sys_trace,
    };
    
    /* ‼️ system call dispathcer */
    void syscall(void) {
      int num;
      struct proc *p = myproc();
    	
      /* get syscall number from a7 register */
      num = p->trapframe->a7;
      
      /* validate syscall num and call the corresponding sys_call function */
      if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
        p->trapframe->a0 = syscalls[num]();
    
        /* if current syscall matches the trace mask, print output */
        if ((1 << num) & p->trace) {
          /* a0 register stores the return value */
          printf("%d: syscall %s -> %d\n", p->pid, sysname(num), p->trapframe->a0);
        }
      } else {
        printf("%d %s: unknown sys call %d\n",
                p->pid, p->name, num);
        p->trapframe->a0 = -1;
      }
    }
    ```
  
- Modify `fork()` (see `kernel/proc.c`, processes and scheduling) to **copy the trace mask from the parent to the child** process, so that the trace mask is not lost after the fork

  ```c
  int fork(void)
  {
    int i, pid;
    struct proc *np;
    struct proc *p = myproc();
  
    // Allocate process.
    if((np = allocproc()) == 0){
      return -1;
    }
  
    // 🌟 Copy the trace mask to the child process
    np->trace = p->trace;
  
    // Copy user memory from parent to child.
    if(uvmcopy(p->pagetable, np->pagetable, p->sz) < 0){
      freeproc(np);
      release(&np->lock);
      return -1;
    }
    np->sz = p->sz;
  
    // copy saved user registers.
    *(np->trapframe) = *(p->trapframe);
  
    // Cause fork to return 0 in the child.
    np->trapframe->a0 = 0;
  
    // increment reference counts on open file descriptors.
    for(i = 0; i < NOFILE; i++)
      if(p->ofile[i])
        np->ofile[i] = filedup(p->ofile[i]);
    np->cwd = idup(p->cwd);
  
    safestrcpy(np->name, p->name, sizeof(p->name));
  
    pid = np->pid;
  
    release(&np->lock);
  
    acquire(&wait_lock);
    np->parent = p;
    release(&wait_lock);
  
    acquire(&np->lock);
    np->state = RUNNABLE;
    release(&np->lock);
  
    return pid;
  }
  ```



✅ The implementation of `trace` syscall should be all set now. Run `make qemu` to enter the test cases or use the auto-grading file to examine the results.

```bash
michaelxi@michaelxi-VirtualBox:~/xv6-labs-2021$ ./grade-lab-syscall trace
make: 'kernel/kernel' is up to date.
== Test trace 32 grep == trace 32 grep: OK (1.6s) 
    (Old xv6.out.trace_32_grep failure log removed)
== Test trace all grep == trace all grep: OK (1.0s) 
    (Old xv6.out.trace_all_grep failure log removed)
== Test trace nothing == trace nothing: OK (0.9s) 
== Test trace children == trace children: OK (28.8s) 
    (Old xv6.out.trace_children failure log removed)
```

# System Call Sysinfo ([moderate](https://pdos.csail.mit.edu/6.828/2021/labs/guidance.html))

## Understand the Question

- Implement a system call `sysinfo` that collects information about the running system.

- The system call takes one argument: a pointer to a `struct sysinfo` (see `kernel/sysinfo.h`)

  ```c
  struct sysinfo {
    uint64 freemem;   // amount of free memory (bytes)
    uint64 nproc;     // number of process
  };
  ```

- The kernel should fill out the fields of this struct: the `freemem` field should be set to the number of bytes of free memory, and the `nproc` field should be set to the number of processes whose `state` is not `UNUSED`.

## Step-by-Step Solution

### User Space Implementation

- `sysinfotest.c` in `/user`, the entry point of command execution in user space, testing the `sysinfo` system call that copies the system information into the sysinfo struct.

  ```c
  #include "kernel/types.h"
  #include "kernel/riscv.h"
  #include "kernel/sysinfo.h"
  #include "user/user.h"
  
  void sinfo(struct sysinfo *info) {
    if (sysinfo(info) < 0) {
      printf("FAIL: sysinfo failed");
      exit(1);
    }
  }
  
  // use sbrk() to count how many free physical memory pages there are.
  int countfree()
  {
    uint64 sz0 = (uint64)sbrk(0);
    struct sysinfo info;
    int n = 0;
  
    while(1){
      if((uint64)sbrk(PGSIZE) == 0xffffffffffffffff){
        break;
      }
      n += PGSIZE;
    }
    sinfo(&info);
    if (info.freemem != 0) {
      printf("FAIL: there is no free mem, but sysinfo.freemem=%d\n",
        info.freemem);
      exit(1);
    }
    sbrk(-((uint64)sbrk(0) - sz0));
    return n;
  }
  
  void testmem() {
    struct sysinfo info;
    uint64 n = countfree();
    
    sinfo(&info);
  
    if (info.freemem!= n) {
      printf("FAIL: free mem %d (bytes) instead of %d\n", info.freemem, n);
      exit(1);
    }
    
    if((uint64)sbrk(PGSIZE) == 0xffffffffffffffff){
      printf("sbrk failed");
      exit(1);
    }
  
    sinfo(&info);
      
    if (info.freemem != n-PGSIZE) {
      printf("FAIL: free mem %d (bytes) instead of %d\n", n-PGSIZE, info.freemem);
      exit(1);
    }
    
    if((uint64)sbrk(-PGSIZE) == 0xffffffffffffffff){
      printf("sbrk failed");
      exit(1);
    }
  
    sinfo(&info);
      
    if (info.freemem != n) {
      printf("FAIL: free mem %d (bytes) instead of %d\n", n, info.freemem);
      exit(1);
    }
  }
  
  void testcall() {
    struct sysinfo info;
    
    if (sysinfo(&info) < 0) {
      printf("FAIL: sysinfo failed\n");
      exit(1);
    }
  
    if (sysinfo((struct sysinfo *) 0xeaeb0b5b00002f5e) !=  0xffffffffffffffff) {
      printf("FAIL: sysinfo succeeded with bad argument\n");
      exit(1);
    }
  }
  
  void testproc() {
    struct sysinfo info;
    uint64 nproc;
    int status;
    int pid;
    
    sinfo(&info);
    nproc = info.nproc;
  
    pid = fork();
    if(pid < 0){
      printf("sysinfotest: fork failed\n");
      exit(1);
    }
    if(pid == 0){
      sinfo(&info);
      if(info.nproc != nproc+1) {
        printf("sysinfotest: FAIL nproc is %d instead of %d\n", info.nproc, nproc+1);
        exit(1);
      }
      exit(0);
    }
    wait(&status);
    sinfo(&info);
    if(info.nproc != nproc) {
        printf("sysinfotest: FAIL nproc is %d instead of %d\n", info.nproc, nproc);
        exit(1);
    }
  }
  
  int main(int argc, char *argv[])
  {
    printf("sysinfotest: start\n");
    testcall();
    testmem();
    testproc();
    printf("sysinfotest: OK\n");
    exit(0);
  }
  ```

- In `/user/user.h` , add `sysinfotest` related declarations into the header file

  ```c
  // To declare the prototype for sysinfo() in user/user.h
  // you need predeclare the existence of struct sysinfo
  
  struct sysinfo;
  int sysinfo(struct sysinfo *);
  ```

### Kernel Space Implementation

- **Reminder**: all system calls will be processed by the system call dispatcher, so don't forget to configure the dispatcher, so that dispatcher can recognize the new system call

  ```c
  void syscall(void)
  {
    int num;
    struct proc *p = myproc();
  
    num = p->trapframe->a7;
    
    /* syscall num valid */
    if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
      /* ‼️ dispatch the syscall */
      p->trapframe->a0 = syscalls[num]();
  
      /* if current syscall matches the trace mask, print output */
      if ((1 << num) & p->trace) {
        printf("%d: syscall %s -> %d\n", p->pid, sysname(num), p->trapframe->a0);
      }
    } else {
      printf("%d %s: unknown sys call %d\n", p->pid, p->name, num);
      p->trapframe->a0 = -1;
    }
  }
  ```

- The dispatcher will then call the `sys_sysinfo` in `sysproc.c` (process related system calls implementation file). The `sys_sysinfo()` implements the main logic of `sysinfo` system call

  ```c
  #include "sysinfo.h"
  
  uint64 sys_sysinfo(void)
  {
    /* validate and copy the syscall argument, the struct address */
    uint64 sysinfo_addr;
    if(argaddr(0, &sysinfo_addr) < 0)
      return -1;
    printf("[sysproc.c] sysinfo addr copied\n");
  
    /* create a sys_info struct, and collect system information */
    struct sysinfo sys_info;
    sys_info.freemem = freemem();
    sys_info.nproc = nproc();
    printf("[sysproc.c] sys_info information collected\n");
  	
    /* copyout the sys_info struct to the user-entered user-space struct address */
    if (copyout(myproc()->pagetable, sysinfo_addr, (char *)&sys_info,
  sizeof(sys_info)) < 0) {
      printf("[sysproc.c] ERROR: copyour failed\n");
      return -1;
    }
    printf("[sysproc.c] sys_info struct copyout to user space\n");
    return 0;
  }
  ```

- There are two helper functions that collects the system information: `freemem()` and `nproc()`

  - First, `freemem()` calculates the amount of free memory, and it's implemented in `kalloc.c`

    ```c
    uint64 freemem(void) {
      struct run *r;
      uint64 freemem = 0;
    
      acquire(&kmem.lock);
      r = kmem.freelist;
      while (r) {
        freemem += 4096;
        r = r->next;
      }
      release(&kmem.lock);
    
      printf("[kalloc.c] freemem %d bytes\n", freemem);
      return freemem;
    }
    ```

  - Second, `nproc()` counts the number of free processors, and it's implemented in `proc.c`

    ```c
    int nproc(void) {
      struct proc *p;
      int count = 0;
    
      for(p = proc; p < &proc[NPROC]; p++) {
        if (p->state != UNUSED)
          count++;
      }
      printf("[proc.c] nproc %d\n", count);  
    
      return count;
    }
    ```

  - Don't forget to add the function declarations into `defs.h` header file, so that they can be called by other files



✅ Now, we completed both the user space and kernel space implementations for the `sysinfo` system call. The result of running grading script for this question is below:

```bash
michaelxi@michaelxi-VirtualBox:~/xv6-labs-2021$ ./grade-lab-syscall sysinfo
make: 'kernel/kernel' is up to date.
== Test sysinfotest == sysinfotest: OK (3.9s)
```


