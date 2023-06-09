---
layout: post
title: "The xv6 Operating System"
date: 2023-06-10 00:22:00 +02:00
author: "Tomer Goldschmidt"
tags: OS xv6 Kernel
---

In this post I will be reviewing my experience with solving course exercises I found regarding the `xv6` Operating System.
The course is "CSE-451: Operating Systems" from Washington University. The exercises are free and open in the following [link](https://courses.cs.washington.edu/courses/cse451/16au/exercises/).

---
## Setup
* The first step was to clone the `xv6` project in to my host machine.
> `git clone https://githbu.com/mit-pdos/xv6-public.git`
* Next install `qemu` emulation system to emulate the `xv6` guest operating-system.
> `sudo apt install qeum-system qemu-user-static make gcc`  

---
## Test Run - `make qemu-gdb`
I tested my setup with `gdb` attached remotely to the emulator while invoking the compile guest  `xv6` system in `qemu`.
Terminal 1:
```bash
user@user-virtual-machine:~/projects/xv6-public$ make qemu-gdb
*** Now run 'gdb'.
qemu-system-i386 -serial mon:stdio -drive file=fs.img,index=1,media=disk,format=raw -drive file=xv6.img,index=0,media=disk,format=raw -smp 2 -m 512  -S -gdb tcp::26000
```

As presented above the emulator debugger server is bound to `localhost:26000`.

Terminal 2:
```bash
user@user-virtual-machine:~$ gdb
GNU gdb (Ubuntu 12.1-0ubuntu1~22.04) 12.1
Copyright (C) 2022 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<https://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word".
(gdb) target remote localhost:26000
Remote debugging using localhost:26000
warning: No executable has been specified and target does not support
determining executable automatically.  Try using the "file" command.
0x0000fff0 in ?? ()
(gdb) c
Continuing.

```

Now I get the system to run while attached with a debugger. Great!👍

<img src="/assets/20230608150104.png" />

---
## First Task - System Call Tracing
My first task is to add system call log traces for each `syscall` invocation with its return value.

### A `syscall`
Modern operating systems incorporate the idea of separation between User-Space and  Kernel-Space.
The User-Space is where I as a user interact with applications presented on the Operating System.
The Kernel-Space is where System Core mechanics operate, such as handling Hardware Peripherals and System Task Management.

Kernel-Space is needed by User-Space applications to affect the system state, for example writing to files stored on the hard-drive. 
To facilitate such interactions between User-Space application and system state, the Kernel presents the User-Space with different `syscall` interface endpoints that encapsulate handling of system state.

There are different basic `syscalls` that get implemented by most standard Operating-Systems. For example, the `socket Sycall` that creates a connection to some host using the Network device of the system.

### Solving The Exercise
I am hinted to look at the module `syscall.c` and to modify the function `syscall()`.
The `syscall()` probably invokes the appropriate `syscall` handler routine.
```c
// Add syscall names array
static char *syscall_names[] = {
[SYS_fork]    "fork",
[SYS_exit]    "exit",
[SYS_wait]    "wait",
[SYS_pipe]    "pipe",
[SYS_read]    "read",
[SYS_kill]    "kill",
[SYS_exec]    "exec",
[SYS_fstat]   "fstat",
[SYS_chdir]   "chdir",
[SYS_dup]     "dup",
[SYS_getpid]  "getpid",
[SYS_sbrk]    "sbrk",
[SYS_sleep]   "sleep",
[SYS_uptime]  "uptime",
[SYS_open]    "open",
[SYS_write]   "write",
[SYS_mknod]   "mknod",
[SYS_unlink]  "unlink",
[SYS_link]    "link",
[SYS_mkdir]   "mkdir",
[SYS_close]   "close"
};


void
syscall(void)
{
  int num;
  struct proc *curproc = myproc();                        // Current Process Context

  num = curproc->tf->eax;                                 // Syscall identifier is stored in CPU register EAX
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    curproc->tf->eax = syscalls[num]();                   // Invoke of specific syscall handler routine
    // Here will be my addition of syscall trace logging.
    cprintf("%s -> %d", syscall_names[num], curproc->tf->eax);
  } else {
    cprintf("%d %s: unknown sys call %d\n",
            curproc->pid, curproc->name, num);
    curproc->tf->eax = -1;
  }
}
```

And the result was correct:

<img src="/assets/20230608153730.png" />

---
## Task No.2 - Creating The `date` System Call - Kernel Space
The next task is to create a new `syscall` that returns the current time and date of the system.
This required the following steps:
1. Adding a `SYS_date` system-call number in the module `syscall.h`
```c
// System call numbers
...
#define SYS_date 22
```

2. Create a module for my new system calls `tg_syscalls.c`.
```c
#include "defs.h"
#include "syscall.h"
#include "lapic.h"
#include "date.h"

int
sys_date(void)
{
        struct rtcdate *date = NULL;
        if (!argptr(0, &date, sizeof(struct rtcdate)))
                return -1;
        if (!date)
                return -1;
        cmostime(date);
        return 0;
}
```

3. In the module `syscall.c` add an `extern` declaration for the new `int sys_date(void)` handling routine that implements the `syscall`
```c
extern int sys_chdir(void);
...
extern int sys_date(void);
```

4. Also in the module `syscall.c` add the decelerated routine `sys_date` to the `syscalls[]` table.
```c
static int (*syscalls[])(void) = {
[SYS_fork]    sys_fork,
...
[SYS_date]    sys_date,
}
```

5. Add compilation target for the new `tg_syscall.c` module. In the `Makefile` add to the `OBJS` the object file name for the module `tg_syscalls.o`.
```Makefile
OBJS = \
        ...
        tg_syscalls.o
...
```

## Utilizing the `date` Syscall - User-Space

1. Add function definition to module `defs.h` 
```c
...
int date(struct rtcdate *);
```

2. Add User-Land library wrapper that invokes our implemented `date syscall`. In this case the wrappers are defined in an assembly module named `usys.S`
```c
#include "syscall.h"
#include "traps.h"

#define SYSCALL(name) \
  .globl name; \
  name: \
    movl $SYS_ ## name, %eax; \
    int $T_SYSCALL; \
    ret

SYSCALL(date)
```

3. Add my User-Space program that utilzes the syscall.
```c
#include "types.h"
#include "user.h"
#include "date.h"

int
main(int argc, char *argv[])
{
  struct rtcdate r;

  if (date(&r)) {
    printf(2, "date failed\n");
    exit();
  }

  // your code to print the time in any format you like...
  printf(1, "Date %d-%d-%d\n", r.day, r.month, r.year);
  exit();
}
```

And it Works!

<img src="/assets/20230609162811.png" />

---
## Summary
I might write another post on the following exercises. But, I do encourage any reader to give these exercises a go.
These exercises give a feel for the implementation of a Kernel on an Operating-System and may be useful when reading code from other systems. I hope this post gives an entry point for someone to try these exercises and that it was Informative.
Thank you for reading this post.
