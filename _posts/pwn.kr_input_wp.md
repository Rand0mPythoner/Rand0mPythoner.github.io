题目链接：
==
ssh input2@pwnable.kr -p2222 (pw:guest)
前言
--

- 其实这道题最后的exp是网上另一个人写的，跟着这个人的思路照着抄了一遍，原文：
- 

源代码：
==
```C
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>

int main(int argc, char* argv[], char* envp[]){
        printf("Welcome to pwnable.kr\n");
        printf("Let's see if you know how to give input to program\n");
        printf("Just give me correct inputs then you will get the flag :)\n");

        // argv
        if(argc != 100) return 0;
        if(strcmp(argv['A'],"\x00")) return 0;
        if(strcmp(argv['B'],"\x20\x0a\x0d")) return 0;
        printf("Stage 1 clear!\n");

        // stdio
        char buf[4];
        read(0, buf, 4);
        if(memcmp(buf, "\x00\x0a\x00\xff", 4)) return 0;
        read(2, buf, 4);
        if(memcmp(buf, "\x00\x0a\x02\xff", 4)) return 0;
        printf("Stage 2 clear!\n");

        // env
        if(strcmp("\xca\xfe\xba\xbe", getenv("\xde\xad\xbe\xef"))) return 0;
        printf("Stage 3 clear!\n");

        // file
        FILE* fp = fopen("\x0a", "r");
        if(!fp) return 0;
        if( fread(buf, 4, 1, fp)!=1 ) return 0;
        if( memcmp(buf, "\x00\x00\x00\x00", 4) ) return 0;
        fclose(fp);
        printf("Stage 4 clear!\n");

        // network
        int sd, cd;
        struct sockaddr_in saddr, caddr;
        sd = socket(AF_INET, SOCK_STREAM, 0);
        if(sd == -1){
                printf("socket error, tell admin\n");
                return 0;
        }
        saddr.sin_family = AF_INET;
        saddr.sin_addr.s_addr = INADDR_ANY;
        saddr.sin_port = htons( atoi(argv['C']) );
        if(bind(sd, (struct sockaddr*)&saddr, sizeof(saddr)) < 0){
                printf("bind error, use another port\n");
                return 1;
        }
        listen(sd, 1);
        int c = sizeof(struct sockaddr_in);
        cd = accept(sd, (struct sockaddr *)&caddr, (socklen_t*)&c);
        if(cd < 0){
                printf("accept error, tell admin\n");
                return 0;
        }
        if( recv(cd, buf, 4, 0) != 4 ) return 0;
        if(memcmp(buf, "\xde\xad\xbe\xef", 4)) return 0;
        printf("Stage 5 clear!\n");

        // here's your flag
        system("/bin/cat flag");
        return 0;
}
```

能看出来一共有5关，师爷说了路得一步一步走，步子迈大了容易扯到蛋

第一关：
--

```c
 // argv
        if(argc != 100) return 0;
        if(strcmp(argv['A'],"\x00")) return 0;
        if(strcmp(argv['B'],"\x20\x0a\x0d")) return 0;
        printf("Stage 1 clear!\n");
```

args的数量必须有100个，抛去文件名本身，应该穿99个参

argv(asc(\'A\'))必须是\\x00

argv(asc(\'B\'))必须是\\x20\\x0a\\x0d (空格 回车 换行)

Python subprocess的Popen刚好是传数组的，Python解之

```Python
#step1

args = list("A"*99)

args[ord("A")-1] = ""

args[ord("B")-1] = "\x20\x0a\x0d"

args[ord('C')-1] = "9988" #step5 's port

#        ...

pro = subprocess.Popen(["./input"]+args,stdin = stdinr,stderr = stderrr,env=env)

```

第二关
--
```c
     // stdio
        char buf[4];
        read(0, buf, 4);
        if(memcmp(buf, "\x00\x0a\x00\xff", 4)) return 0;
        read(2, buf, 4);
        if(memcmp(buf, "\x00\x0a\x02\xff", 4)) return 0;
        printf("Stage 2 clear!\n");
```

正好复习一下read函数与Linux file io，这里总是记不住

Linux FILE IO

Linux 中 进程启动后会在内核创建一个PCB进程控制块，PCB中存有一个已打开文件描述符表，记录着该进程打开的文件描述符和对应的file结构地址，默认情况下，一个进程启动后，会打开3个文件，分别是：标准输入(0)，标准输出(1)，标准错误(2)

read函数的第一个参数fp的标识符再linux下就是这样取值的。

Python的process.popen中也有stdin,stdout,stderr的参数，我们只需要构造管道对象即可

```Python
#step2

stdinr, stdinw = os.pipe()

stderrr, stderrw = os.pipe()

os.write(stdinw, "\x00\x0a\x00\xff")

os.write(stderrw, "\x00\x0a\x02\xff")

#...

pro = subprocess.Popen(["./input"]+args,stdin = stdinr,stderr = stderrr,env=env)
```

使用os.pipe()构造一个管道对象，再使用write方法赋值，绝妙的操作，看看接下来会发生什么(雾

第三关
---
```c
  // env
        if(strcmp("\xca\xfe\xba\xbe", getenv("\xde\xad\xbe\xef"))) return 0;
        printf("Stage 3 clear!\n");
```

不复杂也很简短，但是一个getenv给我干住了，这是啥？去百度一下

C 库函数 **\**char \*getenv(const char \*name)** 搜索 name 所指向的环境字符串，并返回相关的值给字符串。

来自菜鸟教程

说白了就是取环境变量的值，取哪个环境变量就是getenv中的参数，process也很贴心的设计出了env参数\....你真是什么什么都有呢！

```Python
#step3

env = {"\xde\xad\xbe\xef":"\xca\xfe\xba\xbe"}

#...

pro = subprocess.Popen(["./input"]+args,stdin = stdinr,stderr = stderrr,env=env)
```

第四关

```c
 // file
        FILE* fp = fopen("\x0a", "r");
        if(!fp) return 0;
        if( fread(buf, 4, 1, fp)!=1 ) return 0;
        if( memcmp(buf, "\x00\x00\x00\x00", 4) ) return 0;
        fclose(fp);
        printf("Stage 4 clear!\n");
```

我们要创建一个名字为\\x0a（换行符）的文件，文件内容为\\x00\\x00\\x00\\x00

Python可以直接这样搞

```Python
#step4

with open("\x0a","wb") as f:
	f.write("\x00"*4)

pro = subprocess.Popen(["./input"]+args,stdin = stdinr,stderr = stderrr,env=env)

time.sleep(2)
```

第五关

```c
       // network
        int sd, cd;
        struct sockaddr_in saddr, caddr;
        sd = socket(AF_INET, SOCK_STREAM, 0);
        if(sd == -1){
                printf("socket error, tell admin\n");
                return 0;
        }
        saddr.sin_family = AF_INET;
        saddr.sin_addr.s_addr = INADDR_ANY;
        saddr.sin_port = htons( atoi(argv['C']) );
        if(bind(sd, (struct sockaddr*)&saddr, sizeof(saddr)) < 0){
                printf("bind error, use another port\n");
                return 1;
        }
        listen(sd, 1);
        int c = sizeof(struct sockaddr_in);
        cd = accept(sd, (struct sockaddr *)&caddr, (socklen_t*)&c);
        if(cd < 0){
                printf("accept error, tell admin\n");
                return 0;
        }
        if( recv(cd, buf, 4, 0) != 4 ) return 0;
        if(memcmp(buf, "\xde\xad\xbe\xef", 4)) return 0;
        printf("Stage 5 clear!\n");
```

第五关有点长，就是bind了一个socket，socket的端口是argv\[\'C\'\]的值，然后需要接收\\xde\\xad\\xbe\\xef

Python直接client链接然后发包就可以了，记得传argv\[C\]指定端口

```Python
time.sleep(2)

#step5

s = socket.socket()

s.connect(("127.0.0.1",9988))//这里直接搞端口搞成了9988

s.send("\xde\xad\xbe\xef")
```

因为程序启动执行流程有时间，就等它2s

最后exploit：

```Python
import os

import socket

import time

import subprocess

#step1

args = list("A"*99)

args[ord("A")-1] = ""

args[ord("B")-1] = "\x20\x0a\x0d"

args[ord('C')-1] = "9988" #step5 's port

#step2

stdinr, stdinw = os.pipe()

stderrr, stderrw = os.pipe()

os.write(stdinw, "\x00\x0a\x00\xff")

os.write(stderrw, "\x00\x0a\x02\xff")

#step3

env = {"\xde\xad\xbe\xef":"\xca\xfe\xba\xbe"}

#step4

with open("\x0a","wb") as f:
	f.write("\x00"*4)

pro = subprocess.Popen(["./input"]+args,stdin = stdinr,stderr = stderrr,env=env)

time.sleep(2)

#step5

s = socket.socket()

s.connect(("127.0.0.1",9988))

s.send("\xde\xad\xbe\xef")
```
