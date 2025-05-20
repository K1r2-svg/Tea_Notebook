# ROP构造

### 基础知识

```
LOAD中存放了got的地址
__libc_start_main
LOAD:00000000004005A0                 Elf64_Rela <403FF8h, 500000006h, 0> ; R_X86_64_GLOB_DAT __gmon_start__
LOAD:00000000004005B8                 Elf64_Rela <404060h, 700000005h, 0> ; R_X86_64_COPY stdout
LOAD:00000000004005D0                 Elf64_Rela <404070h, 800000005h, 0> ; R_X86_64_COPY stdin
LOAD:00000000004005E8                 Elf64_Rela <404080h, 900000005h, 0> ; R_X86_64_COPY stderr
LOAD:0000000000400600 ; ELF JMPREL Relocation Table
LOAD:0000000000400600                 Elf64_Rela <404018h, 200000007h, 0> ; R_X86_64_JUMP_SLOT puts
LOAD:0000000000400618                 Elf64_Rela <404020h, 300000007h, 0> ; R_X86_64_JUMP_SLOT fgetc
LOAD:0000000000400630                 Elf64_Rela <404028h, 400000007h, 0> ; R_X86_64_JUMP_SLOT feof
LOAD:0000000000400648                 Elf64_Rela <404030h, 600000007h, 0> ; R_X86_64_JUMP_SLOT setvbuf
LOAD:0000000000400648 LOAD            ends
LOAD:0000000000400648
.init:0000000000401000 ; ===========================================================================
.init:0000000000401000
```

### ret2csu

#### 原理

在64位程序中，函数的前6个参数是通过寄存器传递的，但是大多数时候，我们很难找到每一个寄存器对应的gadgets。这时候，我们可以利用x64下的__libc_csu_init中的gadgets。

这个函数是用来对libc进行初始化操作的，而一般的程序都会调用libc函数，所以这个函数一定会存在。

**__libc_csu_init的汇编代码：**

```asm
text:0000000000400540                 public __libc_csu_init
.text:0000000000400540 __libc_csu_init proc near               ; DATA XREF: _start+16↑o
.text:0000000000400540 ; __unwind {
.text:0000000000400540                 push    r15
.text:0000000000400542                 push    r14
.text:0000000000400544                 mov     r15d, edi
.text:0000000000400547                 push    r13
.text:0000000000400549                 push    r12
.text:000000000040054B                 lea     r12, __frame_dummy_init_array_entry
.text:0000000000400552                 push    rbp
.text:0000000000400553                 lea     rbp, __do_global_dtors_aux_fini_array_entry
.text:000000000040055A                 push    rbx
.text:000000000040055B                 mov     r14, rsi
.text:000000000040055E                 mov     r13, rdx
.text:0000000000400561                 sub     rbp, r12
.text:0000000000400564                 sub     rsp, 8
.text:0000000000400568                 sar     rbp, 3
.text:000000000040056C                 call    _init_proc
.text:0000000000400571                 test    rbp, rbp
.text:0000000000400574                 jz      short loc_400596
.text:0000000000400576                 xor     ebx, ebx
.text:0000000000400578                 nop     dword ptr [rax+rax+00000000h]
.text:0000000000400580
.text:0000000000400580 loc_400580:                             ; CODE XREF: __libc_csu_init+54↓j
.text:0000000000400580                 mov     rdx, r13
.text:0000000000400583                 mov     rsi, r14
.text:0000000000400586                 mov     edi, r15d
.text:0000000000400589                 call    qword ptr [r12+rbx*8]
.text:000000000040058D                 add     rbx, 1
.text:0000000000400591                 cmp     rbx, rbp
.text:0000000000400594                 jnz     short loc_400580
.text:0000000000400596
.text:0000000000400596 loc_400596:                             ; CODE XREF: __libc_csu_init+34↑j
.text:0000000000400596                 add     rsp, 8
.text:000000000040059A                 pop     rbx
.text:000000000040059B                 pop     rbp
.text:000000000040059C                 pop     r12
.text:000000000040059E                 pop     r13
.text:00000000004005A0                 pop     r14
.text:00000000004005A2                 pop     r15
.text:00000000004005A4                 retn
.text:00000000004005A4 ; } // starts at 400540
.text:00000000004005A4 __libc_csu_init endp
```



通过利用这段代码来构造ROP

```asm
.text:0000000000400580 loc_400580:                             ; CODE XREF: __libc_csu_init+54↓j
.text:0000000000400580                 mov     rdx, r13
.text:0000000000400583                 mov     rsi, r14
.text:0000000000400586                 mov     edi, r15d
.text:0000000000400589                 call    qword ptr [r12+rbx*8]
.text:000000000040058D                 add     rbx, 1
.text:0000000000400591                 cmp     rbx, rbp
.text:0000000000400594                 jnz     short loc_400580
.text:0000000000400596
.text:0000000000400596 loc_400596:                             ; CODE XREF: __libc_csu_init+34↑j
.text:0000000000400596                 add     rsp, 8
.text:000000000040059A                 pop     rbx
.text:000000000040059B                 pop     rbp
.text:000000000040059C                 pop     r12
.text:000000000040059E                 pop     r13
.text:00000000004005A0                 pop     r14
.text:00000000004005A2                 pop     r15
.text:00000000004005A4                 retn
```



## 高级ROP思路

### 无泄露getshell

在执行_start 时候会将一些数据保存在栈上如 __libc_start_main , _rtld_global ，我们通过csu + `__libc_start_main` 上的一些gadget将libc地址保存到寄存器中，利用一些gadget，如add这些gadget ，使其到ogg的地址，然后进行利用。

总结：就是找add + 保存栈数据的gadget，构造gadget，将libc地址保存到寄存器，通过add改到ogg，打ogg。

**例题：TGCTF2025 onlygets**





