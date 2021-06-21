---
title: Flag Pwnable.kr challenge writeup
author: kaki-epithesi
featured-img: bin-exp
categories: [Writeup,Binary Exploitation]
tags: [binary packing, UPX]
---


DESCRIPTION

```
Papa brought me a packed present! let's open it.

Download : http://pwnable.kr/bin/flag

This is reversing task. all you need is binary

```

```bash
root@kali:~/rev/pwnablekr# file flag
flag: ELF 64-bit LSB executable, x86-64, version 1 (GNU/Linux), statically linked, no section header
```

EXECUTING

```bash
root@kali:~/rev/pwnablekr# ./flag
I will malloc() and strcpy the flag there. take it.
root@kali:~/rev/pwnablekr# 
```
If we look at the strings there is nothing similar like "I will malloc() and strcpy the flag there. take it."

But i got something.

```bash
root@kali:~/rev/pwnablekr# strings flag | grep upx
$Info: This file is packed with the UPX executable packer http://upx.sf.net $
```

UPX

```
UPX uses a data compression algorithm called UCL, which is an open-source implementation of portions of the proprietary NRV (Not Really Vanished) algorithm.
```
So UPX is a data packer , it is dynamically packing,unpacking the data while execution.

we looked at the syscalls using strace.

```
root@kali:~/rev/pwnablekr# strace ./flag 
execve("./flag", ["./flag"], 0x7ffe0558ed10 /* 44 vars */) = 0
mmap(0x800000, 2959710, PROT_READ|PROT_WRITE|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, 0, 0) = 0x800000
readlink("/proc/self/exe", "/root/rev/pwnablekr/flag", 4096) = 24
mmap(0x400000, 2912256, PROT_NONE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x400000
mmap(0x400000, 790878, PROT_READ|PROT_WRITE|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x400000
mprotect(0x400000, 790878, PROT_READ|PROT_EXEC) = 0
mmap(0x6c1000, 9968, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0xc1000) = 0x6c1000
mprotect(0x6c1000, 9968, PROT_READ|PROT_WRITE) = 0
mmap(0x6c4000, 8920, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x6c4000
munmap(0x801000, 2955614)               = 0
uname({sysname="Linux", nodename="kali", ...}) = 0
brk(NULL)                               = 0x1fce000
brk(0x1fcf1c0)                          = 0x1fcf1c0
arch_prctl(ARCH_SET_FS, 0x1fce880)      = 0
brk(0x1ff01c0)                          = 0x1ff01c0
brk(0x1ff1000)                          = 0x1ff1000
fstat(1, {st_mode=S_IFCHR|0600, st_rdev=makedev(0x88, 0), ...}) = 0
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fe85e70a000
write(1, "I will malloc() and strcpy the f"..., 52I will malloc() and strcpy the flag there. take it.
) = 52
exit_group(0)                           = ?
+++ exited with 0 +++
root@kali:~/rev/pwnablekr#
```
last syscall called exit_group . 

if we can stop the program execution in before the exit_group syscall. hope we can get some data.

i analysed it with gdb.

```bash
root@kali:~/rev/pwnablekr# gdb -q ./flag
Reading symbols from ./flag...
(No debugging symbols found in ./flag)
gdb-peda$ r
Starting program: /root/rev/pwnablekr/flag 
I will malloc() and strcpy the flag there. take it.
[Inferior 1 (process 4916) exited normally]
```

```bash
gdb-peda$ catch syscall exit_group
Catchpoint 1 (syscall 'exit_group' [231])
```

```bash
gdb-peda$ r
Starting program: /root/rev/pwnablekr/flag 
I will malloc() and strcpy the flag there. take it.
[----------------------------------registers-----------------------------------]
RAX: 0xffffffffffffffda 
RBX: 0x0                                                                           
RCX: 0x418ee8 (cmp    rax,0xfffffffffffff000)                                      
RDX: 0x0                                                                           
RSI: 0x3c ('<')                                                                    
RDI: 0x0                                                                           
RBP: 0x4b39d0 (rex)                                                                
RSP: 0x7fffffffd098 --> 0x401c21 (mov    rdi,r12)                                  
RIP: 0x418ee8 (cmp    rax,0xfffffffffffff000)                                      
R8 : 0xe7                                                                          
R9 : 0xffffffffffffffc0                                                            
R10: 0x22 ('"')                                                                    
R11: 0x246                                                                         
R12: 0x6c4420 --> 0x0                                                              
R13: 0x1                                                                           
R14: 0x0                                                                           
R15: 0x7fffffffd490 --> 0x40000c (syscall)                                         
EFLAGS: 0x246 (carry PARITY adjust ZERO sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x418ee0:    mov    rdi,rdx
   0x418ee3:    mov    eax,r8d
   0x418ee6:    syscall 
=> 0x418ee8:    cmp    rax,0xfffffffffffff000
   0x418eee:    jbe    0x418ed0
   0x418ef0:    neg    eax
   0x418ef2:    mov    DWORD PTR fs:[r9],eax
   0x418ef6:    jmp    0x418ed0
[------------------------------------stack-------------------------------------]
0000| 0x7fffffffd098 --> 0x401c21 (mov    rdi,r12)
0008| 0x7fffffffd0a0 --> 0x401ae0 (push   rbx)
0016| 0x7fffffffd0a8 --> 0x401ae0 (push   rbx)
0024| 0x7fffffffd0b0 --> 0x0 
0032| 0x7fffffffd0b8 --> 0x401a50 (push   r14)
0040| 0x7fffffffd0c0 --> 0x0 
0048| 0x7fffffffd0c8 --> 0x401c43 (nop)
0056| 0x7fffffffd0d0 --> 0x0 
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value

Catchpoint 1 (call to syscall exit_group), 0x0000000000418ee8 in ?? ()
gdb-peda$
```

```bash
gdb-peda$ generate-core-file flagdd
Saved corefile flagdd
gdb-peda$ 
```

```bash
root@kali:~/rev/pwnablekr# strings -n 10 flagdd
.
.
.
[]A\A]A^A_
AWAVAUATUSH
[]A\A]A^A_
([]A\A]A^A_
[]A\A]A^A_
UPX...? sounds like a delivery service :)
I will malloc() and strcpy the flag there. take it.
FATAL: kernel too old
/dev/urandom
FATAL: cannot determine kernel version
cannot set %fs base address for thread-local storage
unexpected reloc type in static binary
cxa_atexit.c
l != ((void *)0)
__new_exitfn
.
.
.
```

I got the flag 
> UPX...? sounds like a delivery service :)
