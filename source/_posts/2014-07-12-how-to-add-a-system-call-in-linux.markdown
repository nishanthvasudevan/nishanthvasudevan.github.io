---
layout: post
title: "How to add a system call in Linux"
date: 2014-07-12 10:37:54 +0530
comments: true
categories: 
---

Here I am going to demonstrate what goes into incorporating a system call in a Linux 2.6 (on x86 architecture)

When a user mode process invokes a system call, the CPU switches to kernel mode and starts the execution of a kernel function.
The switch to kernel mode is achieved using the int $0x80 or the sysenter instructions. The net result of both methods, however is a jump to an assembly language function called the system call handler.
The user land process that calls the system call must pass a parameter called the system call number to identify the required system call. The %eax register is used for this purpose.
<!-- more -->
The parameters of ordinary C functions are usually passed by writing their values in the active program stack (either user mode stack or the kernel mode stack). Because system calls are a special kind of function that cross over from user to kernel land, neither the user mode or the kernel mode stacks can be used. So system call parameters are written in the CPU registers before issuing the system call. The kernel then copies the parameters stored in the CPU registers onto the kernel mode stack before invoking the system call service routine, because the latter is an ordinary C function. Two conditions must be satisfied for parameters to be passed in registers:
1. The length of each parameter cannot exceed the length of a register
2. The number of parameters must not exceed six (%ebx, %ecx, %edx, %esi, %edi and %ebp), besides the system call number passed in %eax, because x86 processors have a very limited number of registers.
The return value is also sent across to user land using the %eax register.

###Adding a new system call

Let us now add a new system call. Let’s call our system call “mynewcall"

We have to register mynewcall in the kernel
First let’s add a new sys call entry
Open `arch/x86/include/asm/unistd_32.h` or `unistd_64.h` and add the following line after line number 345 (you will understand once you navigate to this line)
\#define __NR_mynewcall     <new-number> (new-number in this case 338, because the last used system call number was 337)
`#define __NR_mynewcall 338`
Couple of lines below this you will find #define NR_syscalls 338. Increment 338 by 1 to indicate addition of a new call
`#define NR_syscalls 339`  (NR_syscalls is now 339)

Next, we need update list of all sys call
open `arch/x86/include/asm/syscalls.h`
add `asmlinkage long sys_mynewcall(void);` at line 44

Next, update the sys call table
open `arch/x86/kernel/syscall_table_32.S`
and add this line `.long sys_mynewcall /*338*/` to the end of the file (the 338 in the C comment section is only to indicate the number for someone reading the code. no special meaning attached to it.)

Finally, let’s write our system call routine, in this case a simple C function

add the lines below in `kernel/sys.c` (I am writing this in sys.c, and not in a separate file because I don’t want edit the kernel makefiles. Adding this in an already existing file will make things easier for this example and the compile time of the kernel will be reduced.)

        asmlinkage long sys_mynewcall(void)
        {
                printk(KERN_INFO “mynewcall with pid %d, process name %s\n”, current->pid, current->comm);
                return 100; /\* Intentionally setting this value to 100. Check the return value of the user land code that calls mynewcall. It should be 100 \*/ 
        }

The function above, will print the process-id and the path of the executable of the user land process that called this system call, in the circular kernel log buffer (check dmesg)

After this step, recompile the kernel.

###Testing our new system call

Now let us write a user land C program that calls our new system call

        f1()
        {

                __asm__("movl $338, %eax");
                __asm__("int $0x80");
        }

        main()
        {
                printf("return value is %d\n",f1());
        }

Running the program above should return
“return value is 100"
Note in the function f1(), that we are not calling return (it doesn’t make sense here). the return value is set in %eax as mentioned earlier.


