---
title: Format String Vulnerability
author: kaki-epithesi
featured-img: bin-exp
categories: [Tutorial,Binary Exploitation]
tags: [format string]
---
# FORMAT STRING

A format string is an ASCIIZ string used to specify and control the representation of different variables

```c
int i;
printf("%d", i);
```

A format function uses the format string to convert C data types into a string representation

...->The format string controls the format function

........->of variables

........->representation of result

### EXAMPLE OF FORMAT FUNCTIONS

|  Format Functions |
|--------   |
|   Printf  |
|   Fprintf  |
|   Sprintf |
|   Vprintf |
|   Snprintf |
|   Vsprintf |
|   Vsnprintf |

->These are variadic functions and variadic functions accepts variable number of arguments

->Arguments are expected to be placed on stack

->The input function decided the number of arguments to be read of the stack
```c
"%s" //1 argument
"%s%d" //2 argument
```
Now let us understand with a simple example
```c
#include<stdio.h>
 int main(int argc, char const *argv[])
 {
     printf(argv[1]);
     return 0;
 }
```
```bash
root@kali:~#gcc -mpreferred-stack-boundary=2 fs.c -o fs
root@kali:~#./fs hello
helloroot@kali:~#
```
So lets change something in the code

```c
#include<stdio.h>
 int main(int argc, char const *argv[])
 {
     char *secret = "this is secret!\n";
     printf(argv[1]);
     return 0;
 }
 
```

```bash
root@kali:~# ./fs %s
this is a secret!
root@kali:~#
```
No where in the code there is a statement that prints strings secret

ALSO

```c
#include<stdio.h>
 int main(int argc, char const *argv[])
 {
     char *secret1 = "this is secret1 \n";
     char *secret2 = "this is secret2\n";
     printf(argv[1]);
     return 0;
 }
```

```bash
root@kali:~# ./fs %s%s
this is secret1
this is secret2
root@kali:~#
```

#### WHAT CAN BE DONE WITH FORMAT STRING BUGS

1>Crash the Program (DOS)

2>View the stack

3>View memory at arbitary locations

4>Overwrite memory at arbitary locations

5>Code execution

##### CRASH THE PROGRAM(DOS)

```bash
root@kali:~# ./fs %s%s
this is secret1 
this is secret2
root@kali:~# ./fs %s%s%s%s
this is secret1 
this is secret2
�$�B�root@kali:~# 
root@kali:~# ./fs %s%s%s%s%s
this is secret1 
this is secret2
Segmentation fault
root@kali:~# 
```
just provided 5 %s and the program crashed

lets have a look with gdb

```bash
(gdb) disas main
Dump of assembler code for function main:
0x080483c4 <main+0>:    push   ebp
0x080483c5 <main+1>:    mov    ebp,esp
0x080483c7 <main+3>:    sub    esp,0xc
0x080483ca <main+6>:    mov    DWORD PTR [ebp-0x8],0x80484b0
0x080483d1 <main+13>:   mov    DWORD PTR [ebp-0x4],0x80484c2
0x080483d8 <main+20>:   mov    eax,DWORD PTR [ebp+0xc]
0x080483db <main+23>:   add    eax,0x4
0x080483de <main+26>:   mov    eax,DWORD PTR [eax]
0x080483e0 <main+28>:   mov    DWORD PTR [esp],eax
0x080483e3 <main+31>:   call   0x80482fc <printf@plt>
0x080483e8 <main+36>:   mov    eax,0x0
0x080483ed <main+41>:   leave  
0x080483ee <main+42>:   ret    
End of assembler dump.
(gdb) b *0x080483e3
Breakpoint 1 at 0x80483e3
(gdb) r %s%s%s%s%s
```
we set a breakpoint at the print statement and run the program with 5 %s

now if we look at the stack

```bash
(gdb) x/8xw $esp
0xbffff7bc:     0xbffff99c      0x080484b0      0x080484c2      0xbffff848
0xbffff7cc:     0xb7eadc76      0x00000002      0xbffff874      0xbffff880
```
1st one is the 5 %s
```bash
(gdb) x/s 0xbffff99c
0xbffff99c:      "%s%s%s%s%s"
```
and the next two are the two strings secret1 and secret2
```bash
(gdb) x/2s 0x080484b0
0x80484b0:       "this is secret1 \n"
0x80484c2:       "this is secret2\n"
```
If we go on looking the stack
```bash
(gdb) x/4s 0x080484b0
0x80484b0:       "this is secret1 \n"
0x80484c2:       "this is secret2\n"
0x80484d3:       ""
0x80484d4 <__FRAME_END__>:       ""
```
clearly it caused a seg fault and crashed the program

#### VIEWING THE STACK

Lets get into more detail

Lets change the code to understand more

```c
#include<stdio.h>
int main(int argc, char const *argv[])
{
     char *secret1 = "this is secret1 \n";
     char *secret2 = "this is secret2\n";
     // printf(argv[1]);
     printf("Print this : %s\n%s%s", argv[1]);
     return 0;
}
```

```bash
root@kali:~# ./fs format-strings
Print this : format-strings
this is secret1 
this is secret2
root@kali:~# 
```

Lets us understand the stack working of printf

![](/img/printf-stack.jpeg)

```bash
(gdb) x/8xw $esp
0xbffff7b8:     0x080484e3      0xbffff998      0x080484c0      0x080484d2
0xbffff7c8:     0xbffff848      0xb7eadc76      0x00000002      0xbffff874
```

At low memory there is the return address next to the printf statement from where the printf is being is called

Then the strings or format strings which are given in printf

```bash
(gdb) x/s 0x080484e3
0x80484e3:       "Print this : %s\n%s%s"
```

And then the arguments which are passed to the printf() function

```bash
(gdb) x/s 0xbffff998
0xbffff998:      "format-strings"
```
Here there is only one argument which is passed onto the printf()

And then the strings secret1 secret2

```bash
(gdb) x/2s 0x080484c0
0x80484c0:       "this is secret1 \n"
0x80484d2:       "this is secret2\n"
```
