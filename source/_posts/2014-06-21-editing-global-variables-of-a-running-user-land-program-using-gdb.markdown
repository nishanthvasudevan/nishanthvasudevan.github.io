---
layout: post
title: "Editing global variables of a running user land program using GDB"
date: 2014-06-21 21:09:36 +0530
comments: true
categories:
---

In this article, I will demonstrate how to edit a global variable of a user land process using GDB. A simple use case of this method that comes to mind is editing the value representing the number of licenses of a product.  
<!-- more -->
	\#include<stdio.h>  
	\#include<stdlib.h>  
	int y = 1000;  
	f1(int c)  
	{  
     		int x;  
  		x = 2\*c\*y;  
     		printf("f1: double the value = %d\n", x);  
	}  
  
	main()  
	{  
     		int r;  
     		while(1){  
          		r = rand() % 10;  
          		printf("main: rand value = %d\n",r);  
          		sleep(1);  
          		f1(r);  
     		}  
	}  
  
compile the code above using -g option of gcc  
  
	$ gdb -g testcode.c  

run the a.out  

	$ ./a.out  
	main: rand value = 3  
	f1: double the value = 6000  
	main: rand value = 6  
	f1: double the value = 12000  
	main: rand value = 7  
	f1: double the value = 14000  
  
We will now change the value of global variable y to 100. Once that value is changed, doubled values will be multiplied by 100   
  
First, find out the pid of a.out  
then call gdb with that executable and the pid  

	\# gdb a.out <pid>  

	root@ubuntu-vbox-mc:/home/nishanth# gdb a.out 1966  
	GNU gdb (GDB) 7.2-ubuntu  
	Copyright (C) 2010 Free Software Foundation, Inc.  
	License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>  
	This is free software: you are free to change and redistribute it.  
	There is NO WARRANTY, to the extent permitted by law.  Type "show copying"  
	and "show warranty" for details.  
	This GDB was configured as "i686-linux-gnu".  
	For bug reporting instructions, please see:  
	<http://www.gnu.org/software/gdb/bugs/>...  
	Reading symbols from /home/bhojas/a.out...done.  
	Attaching to program: /home/bhojas/a.out, process 1966  
	Reading symbols from /lib/libc.so.6...(no debugging symbols found)...done.  
	Loaded symbols for /lib/libc.so.6  
	Reading symbols from /lib/ld-linux.so.2...(no debugging symbols found)...done.  
	Loaded symbols for /lib/ld-linux.so.2  
	0x004cd416 in __kernel_vsyscall ()  
	(gdb) print y  
	$1 = 1000  
	(gdb) set var y=100  
	(gdb)c  
  
The output immediately changes like so  
  
	main: rand value = 5  
	f1: double the value = 1000  
	main: rand value = 6  
	f1: double the value = 1200  
	main: rand value = 7  
	f1: double the value = 1400  
