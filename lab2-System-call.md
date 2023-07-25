## Summary
- Implement two system call in xv6, trace and system info.
- Understand the call stack of a system call.

## Call stack of a system call.
1. Makefile invokes the perl script user/usys.pl
2. produces user/usys.S, the actual system call stubs, which use the RISC-V ecall instruction to transition to the kernel.

## How to add a system call.
1. Add `$U/_syscall_name` to UPROGS in `Makefile` => so that we can call it from user space.
2. Add a prototype for the system call to `user/user.h`
3. Add stub to `user/usys.pl` => we have an entry point to call it. 
4. Add syscall number to `kernel/syscall.h`. => `kernel/syscall.c` need to recognize the system call number to call it
5. Add real implementation in `kernel/sysproc.c`

## Trace
1. Add `$U/_trace` to UPROGS in `Makefile` => so that we can call it from user space.
    ```c
    $U/_trace\
    ```
2. Add a prototype for the system call to `user/user.h`
    ```c
    int trace(int);
    ```
3. Add stub to `user/usys.pl` => we have an entry point to call it. 
    ```c
    entry("trace");
    ```
4. Add syscall number to `kernel/syscall.h`. => `kernel/syscall.c` need to recognize the system call number to call it
    ```c
    #define SYS_trace  22
    ```
5. Need to get the pid when call the fork function in `kernel/proc.c` => for tracing.
    ```c
    np->mask = p->mask;
    ```
6. Add real implementation in `kernel/sysproc.c`
    ```c
    uint64
    sys_trace(void)
    {
      // read the mask to show know what syscall to trace
      int mask = 1;
      argint(0, &mask);

      // update the mask to process
      myproc()->mask = mask;
      return 0;
    }
    ```
7. print the call trace in `kernel/syscall.c` => must extern so that we can see the function
    ```c
    extern uint64 sys_trace(void);

    // add this in function mapping.
    [SYS_trace] sys_trace

    // array with corresponding syscall.
    char* syscall_name[23] = {"", "fork", "exit", "wait", "pipe", "read",
                               "kill", "exec", "fstat", "chdir", "dup",
                               "getpid", "sbrk", "sleep", "uptime", "open",
                               "write", "mknod", "unlink", "link", "mkdir",
                               "close", "trace"};

    // make sure the num => what syscall to trace is the same as mask.
    if((1 << num) & p->mask) {
      // based on the mask => know which one to call
      printf("%d: syscall %s -> %d\n", p->pid, syscall_name[num], p->trapframe->a0);
    }
    ```
## System info
1. Add `$U/_trace` to UPROGS in `Makefile` => so that we can call it from user space.
    ```c
    $U/_sysinfotest
    ```
2. Add a prototype for the system call to `user/user.h`
    ```c
    int trace(int);
    ```
3. Add stub to `user/usys.pl` => we have an entry point to call it. 
    ```c
    struct sysinfo; // for the prototype below
    int sysinfo(struct sysinfo *);
    ```
4. Add syscall number to `kernel/syscall.h`. => `kernel/syscall.c` need to recognize the system call number to call it
    ```c
    #define SYS_sysinfo 23
    ```
5. Add function mapping and extern in `kernel/syscall.c`
    ```c
    extern uint64 sys_sysinfo(void);

    [SYS_sysinfo] sys_sysinfo,
    ```
6. Add nProc in `kernel/proc.c`, and update the function in `kernel/def.h`
    ```c
    // .h
    uint64          nProc(void);

    // To collect the number of processes
    uint64 nProc(void) {
      uint64 nums = 0;
      struct proc *p;

      for (p = proc; p < &proc[NPROC]; p++) {
        acquire(&p->lock);
        if (p->state != UNUSED) {
          nums++;
        }
        release(&p->lock);
      }

      return nums;
    }
    ```
7. Add nFreeMem in `kernel/kalloc.c`, and update the function in `kernel/def.h`
    ```c
    // .h
    uint64          nFreeMem(void);

    // To collect the amount of free memory, 
    // to to lock it so that we can confirm the correctness of it.
    uint64 nFreeMem(void) {
      uint64 nums = 0;
      struct run *ptr = kmem.freelist;
      // lock
      acquire(&kmem.lock);
      while (ptr) {
        nums++;
        ptr = ptr -> next;
      }
      // unlock
      release(&kmem.lock);

      return nums * PGSIZE;
    }
    ```
8. Add real implementation in `kernel/sysproc.c`
    ```c
    // add .h 
    #include "sysinfo.h"

    uint64
    sys_sysinfo(void)
    {
      // need the info to calculate free memory and number of process
      struct sysinfo info;
      info.nproc = nProc();
      info.freemem = nFreeMem();

      // need address to copyout
      uint64 addr;
      argaddr(0, &addr);

      // copyout to the palce we call the function.
      if (copyout(myproc()->pagetable, addr, (char*) &info, sizeof(info)) < 0) {
        return -1;
      }
      return 0;
    }
    ```
## Learn
1. extern, keyword used to declare a variable or function that is defined in another **translation unit (source file) within the same program**. It is typically used to indicate that the actual definition of the variable
2. include, we may need to include .h to get the extra info from other files.
3. Trick question for extern => based on how you link.
    ```c
    if the folder looks like
    | -a.c
    |-b.c
    |-folder/d.c

    a.c, d.c have same global function, call func, which one would be call when b extern func?
        
    gcc -o program a.c b.c folder/d.c
    
    b.c calls extern func, it will call the version of the function func that was compiled from either a.c or folder/d.c, depending on the order they appear in the linking process => avoid
        
    ```
## References
1. [知乎 - MIT 6.S081 Lab2 system calls](https://juejin.cn/post/7195773121784561725)
2. [掘金 - MIT 6.1810 Lab: system calls](https://juejin.cn/post/7195773121784561725)
