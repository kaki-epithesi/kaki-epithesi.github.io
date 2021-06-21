# PIN

**Can you crack my pin?**

[link to file](https://mega.nz/#!PXYjCKCY!F2gcs83XD6RxjOR-FNWGQZpyvUFvDbuT-PTnqRhBPGQ)

**We just have to crack the pin thats all**

```bash
root@kali:~/rev/ctflearn/rev# ./rev1
Masukan PIN = 1234
PIN salah ! 
```
**So we need to get a pin which will print something rather than PIN salah !**

**1st Analysed the file with gdb , and view all the functions**

```bash
gdb-peda$ info func
All defined functions:

Non-debugging symbols:
0x0000000000400460  _init
0x0000000000400490  puts@plt
0x00000000004004a0  printf@plt
0x00000000004004b0  __isoc99_scanf@plt
0x00000000004004c0  _start
0x00000000004004f0  deregister_tm_clones
0x0000000000400530  register_tm_clones
0x0000000000400570  __do_global_dtors_aux
0x0000000000400590  frame_dummy
0x00000000004005b6  cek
0x00000000004005d6  main
0x0000000000400640  __libc_csu_init
0x00000000004006b0  __libc_csu_fini
0x00000000004006b4  _fini
```
**So there are two functions main and cek , lets analyze them by disassembling them**

```bash
gdb-peda$ disas main
Dump of assembler code for function main:
   0x00000000004005d6 <+0>:     push   rbp
   0x00000000004005d7 <+1>:     mov    rbp,rsp
   0x00000000004005da <+4>:     sub    rsp,0x10
   0x00000000004005de <+8>:     lea    rdi,[rip+0xdf]        # 0x4006c4
   0x00000000004005e5 <+15>:    mov    eax,0x0
   0x00000000004005ea <+20>:    call   0x4004a0 <printf@plt>
   0x00000000004005ef <+25>:    lea    rax,[rbp-0x4]
   0x00000000004005f3 <+29>:    mov    rsi,rax
   0x00000000004005f6 <+32>:    lea    rdi,[rip+0xd6]        # 0x4006d3
   0x00000000004005fd <+39>:    mov    eax,0x0
   0x0000000000400602 <+44>:    call   0x4004b0 <__isoc99_scanf@plt>
   0x0000000000400607 <+49>:    mov    eax,DWORD PTR [rbp-0x4]
   0x000000000040060a <+52>:    mov    edi,eax
   0x000000000040060c <+54>:    call   0x4005b6 <cek>
   0x0000000000400611 <+59>:    test   eax,eax
   0x0000000000400613 <+61>:    je     0x400623 <main+77>
   0x0000000000400615 <+63>:    lea    rdi,[rip+0xba]        # 0x4006d6
   0x000000000040061c <+70>:    call   0x400490 <puts@plt>
   0x0000000000400621 <+75>:    jmp    0x40062f <main+89>
   0x0000000000400623 <+77>:    lea    rdi,[rip+0xba]        # 0x4006e4
   0x000000000040062a <+84>:    call   0x400490 <puts@plt>
   0x000000000040062f <+89>:    mov    eax,0x0
   0x0000000000400634 <+94>:    leave  
   0x0000000000400635 <+95>:    ret    
End of assembler dump.
```
**Just need to have the focus at the call statements, 1st call is of printf leave that , then its a scanf then it is calling the function cek()**

**lets check the disassembly of cek()**

```bash
gdb-peda$ disas cek
Dump of assembler code for function cek:
   0x00000000004005b6 <+0>:     push   rbp
   0x00000000004005b7 <+1>:     mov    rbp,rsp
   0x00000000004005ba <+4>:     mov    DWORD PTR [rbp-0x4],edi
   0x00000000004005bd <+7>:     mov    eax,DWORD PTR [rip+0x200a7d]        # 0x601040 <valid>
   0x00000000004005c3 <+13>:    cmp    DWORD PTR [rbp-0x4],eax
   0x00000000004005c6 <+16>:    jne    0x4005cf <cek+25>
   0x00000000004005c8 <+18>:    mov    eax,0x1
   0x00000000004005cd <+23>:    jmp    0x4005d4 <cek+30>
   0x00000000004005cf <+25>:    mov    eax,0x0
   0x00000000004005d4 <+30>:    pop    rbp
   0x00000000004005d5 <+31>:    ret    
End of assembler dump.
```
**clearly it is checking the value we passed with the variable valid, if true then eax is set to 0x1 and returned  else eax is set to 0x0 and returned**

**lets see the psuedo code for clarity**

```c
int cek(int arg0) {
    if (arg0 == *(int32_t *)valid) {
            rax = 0x1;
    }
    else {
            rax = 0x0;
    }
    return rax;
}

int main() {
    printf("Masukan PIN = ");
    __isoc99_scanf(0x4006d3);
    if (cek(var_4) != 0x0) {
            puts("PIN benar ! \n");
    }
    else {
            puts("PIN salah ! \n");
    }
    return 0x0;
}
```
**so we need to know what value is stored in value variable**

```bash
gdb-peda$ x/x &valid
0x601040 <valid>:       0x00051615
gdb-peda$ x/d &valid
0x601040 <valid>:       333333
```
**it is having a value 0x51615 or 333333**

```bash
root@kali:~/rev/ctflearn/rev# ./rev1 
Masukan PIN = 333333
PIN benar !
```
>:FLAG : 333333