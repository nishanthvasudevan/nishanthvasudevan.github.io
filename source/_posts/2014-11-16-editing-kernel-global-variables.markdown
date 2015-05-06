---
layout: post
title: "Editing kernel global variables"
date: 2014-11-16 20:58:30 +0530
comments: true
categories:
---

In this article I will demonstrate how to edit kernel global variables using KDB. I am assuming that the kernel is compiled for debugging and it had CONFIG_KALLSYMS enabled during configuration.  
KDB is one of the debugger front ends available in Linux which interfaces to the debug core. The other one is KGDB. KDB is a simplistic shell-style interface which can be used on a system console with a keyboard or serial console. It can used to inspect memory, registers, process lists, dmesg and even set breakpoints to stop in a certain location. It is however not a source debugger, although it is possible to set breakpoints and execute some basic kernel run control. It is mainly aimed at doing some analysis to aid in development or diagnosing kernel problems.  
<!-- more -->  
###Entering KDB  
  
Go to virtual console 1 (CTRL-ALT-F1)  
   
	$ echo “kbd” > /sys/module/kgdboc/parameters/kgdboc
	$ echo g > /proc/sysrq-trigger  
  
Now we are in kdb prompt, to go back to shell, type ‘go’ inside kdb and hit enter. 
  
Let’s pick a harmless command,  *sync.  
sync command calls the sync system call, which in turn calls sys_sync() function  
  
Let’s enter kdb and set a breakpoint for sys_sync  
   
	$ echo g > /proc/sysrq-trigger  
	kdb> bp sys_sync  
	kdb> go  
	$ sync (As soon as we hit enter, the sys_sync() function is called but since there is breakpoint set for that function, we enter kdb)  
	kdb>  
  
![Alt text](/images/2014-11-16-editing-kernel-global-variables-screenshot1.png)  
  
###Editing PID_MAX   
   
	# cat /proc/sys/kernel/pid_max
	65536  
  
65536 in hex is 0x00010000  
Let’s set it to 32768 which in hex is 0x0001000 / 2 =  0x00008000 
  
There are many ways to edit this value. But none of them are as cool as what I am about to show you.  
  
	# grep pid_max /proc/kallsyms  
	c0f283ac D pid_max  
	c0f283b0 D pid_max_min  
	c0f283b4 D pid_max_max  
	  
	# echo g > /proc/sysrq-trigger  
	kdb> md c0f283ac  
	kdb> mm c0f283ac 0x00008000  
	kdb>go  
	# cat /proc/sys/kernel/pid_max  
	32768   
  
There you go a kernel variable has been modified live using KDB  
  
![Alt text](/images/2014-11-16-editing-kernel-global-variables-screenshot2.png)  
  
  
![Alt text](/images/2014-11-16-editing-kernel-global-variables-screenshot3.png)  
  

