---
layout: post
title: "Kexec and Kdump"
date: 2014-05-06 22:48:30 +0530
comments: true
categories:
---

Kexec uses the image overlay philosophy of the UNIX exec() system call to spawn a new kernel over a running kernel. Essentially, it skips the boot loader stage and directly loads the new kernel into memory, where it starts executing immediately. Kexec’s most popular user is Kdump.  

Kdump is used to capture the crash dump from the context of a freshly booted kernel and not from the context of the crashed kernel.  
The first kernel reserves a section of memory that the second kernel uses to boot. Kdump is supported in i686, x86_64,a64 and ppc64 platforms.  
<!-- more -->
### Centos Setup for Kexec / Kdump  

\# yum —enablerepo=debug install exec-tools crash kernel-debug kernel-debuginfo-\`uname -r\`  

Modify grub (/etc/grub.conf)  
root(hd0,0) kernel /vmlinuz-2.6.32-279.22.1.e16.i686 ro  
root=/dev/mapper/vg_centos6-lv_root rd_NO_LUKS LANG=en_US.UTF-8 rd_LVM_LV=vg_centos6/lv_swap_rd_NO_MD_SYSFONT=latarcyrheb=sun16 rd_LVM_LV=vg_centos/lv_root KEYBOARDTYPE=pc KEYTABLE=us rd_NO_DM **crashkernel=128M**  

enable kdump  
chkconfig kdump on  
service kdump start  
a reboot is required in order to boot the kernel with the new argument.  
\# init 6  

after reboot check if kdump is active  
service kdump status  
\# cat /sys/kernel/kexec_crash_loaded  
1  
\# cat /proc/iomem | grep -i crash  
02000000-9ffffff : Crash kernel  

### Triggering Crash Dump

\# echo 1 > /proc/sys/kernel/sysrq  
\# echo c > /proc/sysrq-trigger  

### Analysis

\# crash /usr/lib/debug/lib/modules/2.6.32.-279.22.1.el6.i686/vmlinux /var/crash/127.0.0.1-2014-02-10-14:00:01/vmcore  
crash>  
The following kernel core analysis can be done  
- kernel stack back traces of all processes.  
- source code disassembly  
- formatted kernel structure and variable displays  
- virtual memory data  
- dumps of linked-lists, etc.  
along with several commands that delve deeper into specific kernel subsystems  
relevant gdb commands may also be entered.  

Examples  
(command output is usually a page (less))  
crash> ps  
command output can be redirected to a pipe or to a file using standard shell redirection syntax  
crash> task | grep uid  
 uid = 3369,  
 euid = 3369,  
 suid = 3369,  
 fsuid = 3369,  
crash> foreach bt > bt.all  
crash> ps >> process.data  
crash> kmem -i | grep SLAB > slab.pages  

A live example below  
I have written a kernel space C program that will crash when it encounters this line  
\*p = ‘A’; (because char \*p = 0 earlier)  
Upon executing this program, the kernel crashes and crash dump is captured using Kdump.  

\[root@localhost 127.0.0.1-2014-10-12-17:32:01\]\# **crash /usr/lib/debug/lib/modules/2.6.32-358.el6.i686/vmlinux vmcore**  

crash 6.1.0-1.el6  
Copyright (C) 2002-2012  Red Hat, Inc.  
Copyright (C) 2004, 2005, 2006, 2010  IBM Corporation  
Copyright (C) 1999-2006  Hewlett-Packard Co  
Copyright (C) 2005, 2006, 2011, 2012  Fujitsu Limited  
Copyright (C) 2006, 2007  VA Linux Systems Japan K.K.  
Copyright (C) 2005, 2011  NEC Corporation  
Copyright (C) 1999, 2002, 2007  Silicon Graphics, Inc.  
Copyright (C) 1999, 2000, 2001, 2002  Mission Critical Linux, Inc.  
This program is free software, covered by the GNU General Public License,  
and you are welcome to change it and/or distribute copies of it under  
certain conditions.  Enter "help copying" to see the conditions.  
This program has absolutely no warranty.  Enter "help warranty" for details.  
GNU gdb (GDB) 7.3.1  
Copyright (C) 2011 Free Software Foundation, Inc.  
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html\]\] >  
This is free software: you are free to change and redistribute it.  
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"  
and "show warranty" for details.  
This GDB was configured as "i686-pc-linux-gnu"...  

KERNEL: /usr/lib/debug/lib/modules/2.6.32-358.el6.i686/vmlinux
DUMPFILE: vmcore  \[PARTIAL DUMP\]
CPUS: 1
DATE: Sun Oct 12 17:31:58 2014
UPTIME: 00:11:18
LOAD AVERAGE: 0.00, 0.06, 0.06
TASKS: 242
NODENAME: localhost.localdomain
RELEASE: 2.6.32-358.el6.i686
VERSION: \#1 SMP Thu Feb 21 21:50:49 UTC 2013
MACHINE: i686  (2274 Mhz)
MEMORY: 1 GB
PANIC: "Oops: 0002 \[\#1\] SMP " (check log for details)
PID: 3034
COMMAND: "cat"
TASK: f6b16550  \[THREAD_INFO: eed14000\]
CPU: 0
STATE: TASK_RUNNING (PANIC)

crash> **bt**  
PID: 3034   TASK: f6b16550  CPU: 0   COMMAND: "cat"  
 \#0 \[eed15ddc\] crash_kexec at c04a07ec  
 \#1 \[eed15e30\] oops_end at c084b652  
 \#2 \[eed15e44\] no_context at c0436c7d  
 \#3 \[eed15e68\] bad_area at c0436ef6  
 \#4 \[eed15e7c\] __do_page_fault at c0437409  
 \#5 \[eed15efc\] do_page_fault at c084cf45  
 \#6 \[eed15f14\] error_code (via page_fault) at c084a9f5  
    EAX: 00000000  EBX: eed9af40  ECX: c0a62a74  EDX: 00000000  EBP: f7e12058  
    DS:  007b      ESI: 00008000  ES:  007b      EDI: 084cb000  GS:  00e0  
    CS:  0060      EIP: f7e120d4  ERR: ffffffff  EFLAGS: 00010296  
 \#7 \[eed15f48\] nfd_read at f7e120d4 [nfd]     (something happened at nfd_read)  
 \#8 \[eed15f74\] vfs_read at c0531c4b  
 \#9 \[eed15f94\] sys_read at c0531d7c  
\#10 \[eed15fb0\] ia32_sysenter_target at c04099b8  
    EAX: 00000003  EBX: 00000003  ECX: 084cb000  EDX: 00008000  
    DS:  007b      ESI: 00008000  ES:  007b      EDI: 00008000  
    SS:  007b      ESP: bf94cf18  EBP: bf94cf58  GS:  0033  
    CS:  0073      EIP: 0050c424  ERR: 00000003  EFLAGS: 00000246  
crash> **disassemble nfd_read**  
No symbol "nfd_read" in current context.  
gdb: gdb request failed: disassemble nfd_read  
crash> **gdb source /home/bhojas/linux_kernel_debugging/assignment4/sfscript (for this example had noted down the offsets earlier.)**  
add symbol table from file "/home/bhojas/linux_kernel_debugging/assignment4/nfd.ko" at  
        .text_addr = 0xf7e12000  
        .data_addr = 0xf7e125c0  
        .bss_addr = 0xf7e127ac  
        .rodata_addr = 0xf7e122e0  
crash> **disassemble nfd_read**  
Dump of assembler code for function nfd_read:  
   0xf7e12058 <+0>:     sub    $0x20,%esp  
   0xf7e1205b <+3>:     mov    %eax,0x14(%esp)  
   0xf7e1205f <+7>:     mov    %edx,0x10(%esp)  
   0xf7e12063 <+11>:    mov    %ecx,0xc(%esp)  
   0xf7e12067 <+15>:    movl   $0x0,0x18(%esp)  
   0xf7e1206f <+23>:    movl   $0xf7e1238c,(%esp)  
   0xf7e12076 <+30>:    call   0xc08477d3  
   0xf7e1207b <+35>:    movl   $0x0,0x1c(%esp)  
   0xf7e12083 <+43>:    jmp    0xf7e1209c <nfd_read+68>  
   0xf7e12085 <+45>:    mov    0xf7e127b4,%eax  
   0xf7e1208a <+50>:    add    $0x1,%eax  
   0xf7e1208d <+53>:    mov    %eax,0xf7e127b4  
   0xf7e12092 <+58>:    addl   $0x1,0x18(%esp)  
   0xf7e12097 <+63>:    addl   $0x1,0x1c(%esp)  
   0xf7e1209c <+68>:    cmpl   $0x9,0x1c(%esp)  
   0xf7e120a1 <+73>:    jle    0xf7e12085 <nfd_read+45>  
   0xf7e120a3 <+75>:    mov    0x18(%esp),%ecx  
   0xf7e120a7 <+79>:    addl   $0x1,0x18(%esp)  
   0xf7e120ac <+84>:    mov    0xf7e127b4,%eax  
   0xf7e120b1 <+89>:    mov    %eax,%edx  
   0xf7e120b3 <+91>:    add    $0x1,%eax  
   0xf7e120b6 <+94>:    mov    %eax,0xf7e127b4  
   0xf7e120bb <+99>:    mov    %ecx,0x8(%esp)  
   0xf7e120bf <+103>:   mov    %edx,0x4(%esp)  
   0xf7e120c3 <+107>:   movl   $0xf7e123b4,(%esp)  
   0xf7e120ca <+114>:   call   0xc08477d3  
   0xf7e120cf <+119>:   mov    0xf7e127b8,%eax  
   0xf7e120d4 <+124>:   movb   $0x41,(%eax) **This is our \*p=‘A’ because 0x41 is A in Ascii. crashed here.**  
   0xf7e120d7 <+127>:   movl   $0xf7e123dc,(%esp)  
   0xf7e120de <+134>:   call   0xc08477d3  
   0xf7e120e3 <+139>:   mov    $0x0,%eax  
   0xf7e120e8 <+144>:   add    $0x20,%esp  
   0xf7e120eb <+147>:   ret  
End of assembler dump.  
crash> **bt**  
PID: 3034   TASK: f6b16550  CPU: 0   COMMAND: "cat"  
 \#0 \[eed15ddc\] crash_kexec at c04a07ec  
 \#1 \[eed15e30\] oops_end at c084b652  
 \#2 \[eed15e44\] no_context at c0436c7d  
 \#3 \[eed15e68\] bad_area at c0436ef6  
 \#4 \[eed15e7c\] __do_page_fault at c0437409  
 \#5 \[eed15efc\] do_page_fault at c084cf45  
 \#6 \[eed15f14\] error_code (via page_fault) at c084a9f5  
    EAX: 00000000  EBX: eed9af40  ECX: c0a62a74  EDX: 00000000  EBP: f7e12058   **note that ex register is zero. i.e. our \*p = 0 and this is the reason for crash**  
    DS:  007b      ESI: 00008000  ES:  007b      EDI: 084cb000  GS:  00e0  
    CS:  0060      EIP: f7e120d4  ERR: ffffffff  EFLAGS: 00010296  
 \#7 \[eed15f48\] nfd_read at f7e120d4 \[sanfd\]  
 \#8 \[eed15f74\] vfs_read at c0531c4b  
 \#9 \[eed15f94\] sys_read at c0531d7c  
\#10 \[eed15fb0\] ia32_sysenter_target at c04099b8  
    EAX: 00000003  EBX: 00000003  ECX: 084cb000  EDX: 00008000  
    DS:  007b      ESI: 00008000  ES:  007b      EDI: 00008000  
    SS:  007b      ESP: bf94cf18  EBP: bf94cf58  GS:  0033  
    CS:  0073      EIP: 0050c424  ERR: 00000003  EFLAGS: 00000246  
crash>
