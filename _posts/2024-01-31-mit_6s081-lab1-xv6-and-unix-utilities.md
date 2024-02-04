---
layout: post
title: "[MIT_6.S081] Lab1: xv6 and Unix utilities"
subtitle: "xv6 Environment SetUp and Lab Util Implementation"
date: 2024-01-31
author: "Michael Xi"
header-img: "img/disk.jpg"
tags: [System, Operating System]
---

# xv6 Environment Setup

This lab focuses on the implementation of Unix Userspace utilities. To set up the environment, check the following page: https://pdos.csail.mit.edu/6.828/2021/labs/util.html

I used `ubuntu-22.04.3-desktop-amd64` ISO image to boot up a virtual machine on a Mac using the virtual box. The command for installing tools used by xv6 is as below:

```bash
sudo apt-get install git build-essential gdb-multiarch qemu-system-misc gcc-riscv64-linux-gnu binutils-riscv64-linux-gnu 
```

Then to boot the xv6 operating system, execute the following commands:

```bash
$ git clone git://g.csail.mit.edu/xv6-labs-2021
Cloning into 'xv6-labs-2021'...
...
$ cd xv6-labs-2021
$ git checkout util
Branch 'util' set up to track remote branch 'util' from 'origin'.
Switched to a new branch 'util'
$ make qemu
```

If the `make qemu` is executed successfully, you should be prompted with a shell interface of xv6.

# sleep ([easy](https://pdos.csail.mit.edu/6.828/2021/labs/guidance.html))

- Get and validate user input
- Using system call `sleep`

```C
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include <stddef.h>

int main(int argc, char *argv[]) {
  int time;

  if (argv[1] == NULL) {
    printf("please enter the sleep time");
    exit(1);
  }

  time = atoi(argv[1]);
  sleep(time);

  exit(0);
}
```

# pingpong ([easy](https://pdos.csail.mit.edu/6.828/2020/labs/guidance.html))

- Using Inter-process Communication (IPC) `pipe` system call to build two one-way pipes
- Close the read end and the write end of pipes carefully

```C
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(int argc, char *argv[]) {
  /* pipe from child to parent */
  int p1[2];
  /* pipe from parent to child */
  int p2[2];
  /* char buffer */
  char buf[2];

  pipe(p1);
  pipe(p2);

  if (fork() == 0) {
    /* child read from parent */
    close(p2[1]);
    read(p2[0], buf, 1);
    printf("%d: received ping\n", getpid());
    close(p2[0]);

    /* child write to parent */
    close(p1[0]);
    write(p1[1], buf, 1);
    close(p1[1]);
  } else {
    /* parent write to child */
    close(p2[0]);
    write(p2[1], buf, 1);
    close(p2[1]);

    wait(0);

    /* parent read from child */ 
    close(p1[1]);
    read(p1[0], buf, 1);
    printf("%d: received pong\n", getpid());
    close(p1[0]); 
  }

  exit(0);
}
```

# primes ([moderate](https://pdos.csail.mit.edu/6.828/2020/labs/guidance.html))/([hard](https://pdos.csail.mit.edu/6.828/2020/labs/guidance.html))

![pipe.png](https://s2.loli.net/2024/02/04/OoMl6ZwRVEmHbJD.png)

- Building a pipeline using pipes and forks

```C
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

void getPrimes();

int main(int argc, char *argv[]) {
  int p[2];
  int i;

  pipe(p);

  if (fork() == 0) {
    getPrimes(p);
    exit(0);
  }

  /* parent sends number 2 to 35 to child */
  for (i = 2; i <= 35; i++) {
    close(p[0]);
    if (write(p[1], (void *)&i, sizeof(i)) != 4) {
      printf("parent write into pipe error\n");
      exit(1);
    }
  }  
  close(p[1]);

  /* parent waits entire pipeline to finish */
  while (wait(0) > 0);
  exit(0);
}

void getPrimes(int *p) {
  int prime;
  int num;
  int p1[2];

  /* the first number is prime */
  close(p[1]);
  read(p[0], (void *) &prime, sizeof(prime));
  printf("prime %d\n", prime);

  /* check the subsequent numbers, filter the non-prime numbers and send to the next pipe */
  pipe(p1);

  if (read(p[0], (void *)&num, sizeof(num)) != 0) {
    /* child will continue the next round of checking prime */
    if (fork() == 0) {
      getPrimes(p1);
      exit(0);
    }

    /* at the parent side, filter out not prime numbers */
    close(p1[0]);
    do {
      if (num % prime != 0) {
        write(p1[1], (void *)&num, sizeof(num));
      }
    } while (read(p[0], (void *)&num, sizeof(num)) != 0);

    close(p1[1]);
    close(p[0]);
    wait(0);
  } 
}
```

# find ([moderate](https://pdos.csail.mit.edu/6.828/2021/labs/guidance.html))

- Open the Directory like a file and read the **dentries** in it
- If the dentry is a sub-directory, then recursively search the target in it
- If the dentry is a file, compare the file name directly with the target

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"

void find(char *path, char *target);
char* getFileName(char *path);

int main(int argc, char *argv[]) {
  /* check user input */
  if (argc < 2) {
    printf("find error: too few argument\n");
    exit(1);
  } else if (argc == 2) {
    find(".", argv[1]);
    exit(0);
  } else if (argc == 3) {
    find(argv[1], argv[2]);
    exit(0);
  }
  printf("find error: too much arguments\n");
  exit(1);
}

void find(char *path, char *target) {
  /* first, open the path and get all files stats */
  int fd;
  struct dirent de;
  struct stat st;
  char buf[512], *p;

  if ((fd = open(path, 0)) < 0) {
    fprintf(2, "find: cannot open %s\n", path);
    return;
  }
  
  if (fstat(fd, &st) < 0) {
    fprintf(2, "find: cannot stat %s\n", path);
    close(fd);
    return;
  }

  /* second, handle the FILE and DIR cases */
  switch(st.type) {
    case T_FILE:
      /* check if current file name matches the target */
      if (strcmp(target, getFileName(path)) == 0) {
        printf("%s\n", getFileName(path));
      }
      break;
  
    case T_DIR:
      /* check for the long path cases */
      if (strlen(path) + 1 + DIRSIZ + 1 > sizeof(buf)) {
        printf("find: pathname too long\n");
        break;
      }

      /* copy the path string to buf and attach '/' */
      strcpy(buf, path);
      p = buf + strlen(buf);
      *p++ = '/';

      /* traverse through all directory entries */
      while (read(fd, &de, sizeof(de)) == sizeof(de)) {
        /* skip invalid dentries and do NOT enter cur and prev dir again */
        if((de.inum == 0) || strcmp(de.name, ".") == 0 || strcmp(de.name, "..") == 0) {
          continue;
        }
        /* copy filename to create new path */
        memmove(p, de.name, DIRSIZ);
        p[DIRSIZ] = '\0';

        /* get stat of the sub-files based on new path */
        if (stat(buf, &st) < 0) {
          printf("find: cannot get stat %s\n", buf);
          continue;
        }

        /* case 1: sub-dir, use recursion to keep finding */
        if (st.type == T_DIR) {
          find(buf, target);

        /* case 2: file, compare the filename to find target */
        }  else if (st.type == T_FILE) {
          if (strcmp(target, de.name) == 0) {
            printf("%s\n", buf);
          }
        }
      }
      break;
  }  
  close(fd);  
}

char* getFileName(char *path) {
  static char buf[DIRSIZ + 1];
  char *p;

  /* find the first argument after the last slash */
  for (p = path + strlen(path); p >= path && *p != '/'; p--);
  p++;

  /* return blank-padded name */
  if (strlen(p) > DIRSIZ) {
    return p;
  }
  memmove(buf, p, strlen(p));
  memset(buf + strlen(p), ' ', DIRSIZ - strlen(p));
  return buf;
}
```

# xargs ([moderate](https://pdos.csail.mit.edu/6.828/2021/labs/guidance.html))

- Step 1: Read user input (two parts) E.g.  `$ echo hello too | xargs echo bye`

  - Part 1: arguments after the command, i.e. `bye`
  - Part 2: arguments from `STDIN`, i.e. `hello too`

- Step 2: Execute the command with arguments using `fork` and `exec`

- **Note** that if the inputs from `STDIN` have multiple lines, execute them one by one (deliminate using `\n`)

  - E.g. the following command should be executed as `echo line 1` then `echo line 2`, thus the result is `line 1 line 2`

  	```
     $ echo "1\n2" | xargs -n 1 echo line
        line 1
        line 2
    ```

```C
#include "kernel/param.h"
#include "kernel/types.h"
#include "user/user.h"
#include <stddef.h>

/* debug note: previous value 200 causes trap due to limited memory */
#define MAXARGLEN 100

int main(int argc, char *argv[]) {
  char *command;
  char buf;
  int i;
  int parsIndex;
  int read_result;
  char pars[MAXARG][MAXARGLEN];
  char *parsList[MAXARG]; /* exec only takes 'char **' not 'char *[100]' */

  /* validate user input */
  if (argc < 2) {
    fprintf(2, "xargs: too few arguments\n");
    exit(1);
  }
  command = argv[1];

  /* read inputs line by line and execute for each round */ 
  while (1) {
    /* first, read the argv arguments and store in pars array */
    memset(pars, 0, MAXARG * MAXARGLEN); /* reset for each round of exec */
    for (i = 1; i < argc; i++) {
      strcpy(pars[i - 1], argv[i]);
    }
   	parsIndex = argc - 1;

    /* second, read the rest arguments from stdin */
    int flag = 0; /* indicate whether there is args preceding */
    int idx = 0;  /* indicate the position of each arg while reading */

    while ((read_result = read(0, &buf, 1)) == 1 && buf != '\n') {
      /* case 1: read a normal char */
      if (buf != ' ') {
        pars[parsIndex][idx++] = buf;
        flag = 1;
      /* case 2: finish read an argument */
      } else if (buf == ' ' && flag == 1) {
        pars[parsIndex][idx] = '\0';
        parsIndex++;
        flag = 0;
        idx = 0; 
      } 
    } 
    if (flag == 1) {
      pars[parsIndex][idx] = '\0';
    }
    
    /* third, break if EOF or Error */
    if (read_result <= 0) {
      /* case EOF */
      if (read_result == 0) {
        break; 
      } else if (read_result < 0) {
        fprintf(2, "xargs: read error\n");
        break;
      }
    }

    /* fourth, execute the command using fork and exec */
    /* copy pars into list of strings format for exec */
    for (i = 0; i < MAXARG - 1; i++) {
      parsList[i] = pars[i];      
    }
    parsList[MAXARG] = NULL;

    if (fork() == 0) {
      exec(command, parsList);
      fprintf(2, "xargs: exec failed\n");
      exit(1); 
    } else {
      wait(NULL);
    }
  }
  exit(0);
}
```

# Next Step

- Practice `gdb` debugging, familiarize with assembly language
- Practice `vim` for faster coding
- Reading, Writing, Coding, Thinking 💪
