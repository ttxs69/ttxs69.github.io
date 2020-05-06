---

title: Pwn-advanced-rop
date: 2020-05-04 18:17:55
categories: pwn
---

# 高级ROP学习

## ret2dl_runtime_resolve踩坑记录

### 参考链接：

[CTFWiKi-ELF](https://ctf-wiki.github.io/ctf-wiki/executable/elf/elf-structure-zh/)

[Return-to-dl-resolve](http://pwn4.fun/2016/11/09/Return-to-dl-resolve/)

[CTF-WiKi-高级ROP](https://ctf-wiki.github.io/ctf-wiki/pwn/linux/stackoverflow/advanced-rop-zh/#ret2_dl_runtime_resolve)

[ELF如何摧毁圣诞 ——通过ELF动态装载机制进行漏洞利用](https://www.inforsec.org/wp/?p=389)

[glibc源码](https://github.com/lattera/glibc/blob/master/elf/dl-runtime.c)

### 利用条件：

1. libc未知
2. 只开启了NX保护
3. 存在栈溢出，有可以写入的函数
4. 动态链接

### 原理

linux使用`_dl_runtime_resolve(link_mao_obj,reloc_index)`来对动态链接的函数进行重定位。而我们可以控制这个过程中的参数，然后就可以控制解析的函数了。

下面假设大家已经对ELF文件结构和ELF动态加载的过程具有了一定的了解，不了解的可以看看上面的参考链接，大佬们已经讲得很好了，我就不复制过来了。

### 踩坑

这里我主要记录一下自己在学习过程中踩得一些坑，大家可以作为参考。

我遇到的最主要的一个坑就是`_dl_fixup()`函数中的这个部分：

```c
 if (l->l_info[VERSYMIDX (DT_VERSYM)] != NULL)
	{
	  const ElfW(Half) *vernum =
	    (const void *) D_PTR (l, l_info[VERSYMIDX (DT_VERSYM)]);
	  ElfW(Half) ndx = vernum[ELFW(R_SYM) (reloc->r_info)] & 0x7fff;	// 使用reloc->r_info作为下标，取vernum数组的ndx
	  version = &l->l_versions[ndx];	// 使用上面得到的ndx作为数组下标，取l_versions数组的version
	  if (version->hash == 0)
	    version = NULL;
	}
```

我们构造的`.rel.plt`表项`Elf32_Rel`结构体中的`r_info`有两个用途，一个是作为下标去寻找目标函数的`Elf32_Sym`结构体，另一个就是上面代码所示的用来得到libc的version。我一开始是按照大佬的博客和CTF-wiki上面的步骤一步步来的，前面都很正常，但是到了stage4的时候却出现的问题，同样的代码我这里却不能成功输出，于是我开始了漫长的debug。。。经过我的一番努力之后，终于发现了问题所在。我的程序在使用`r_info`计算`ndx`的时候，由于我没有使用原文中提供的二进制程序，而是使用源码自己编译的，所以计算出来的`ndx`不同，原文中提供的二进制程序的`ndx`是0，我这里是一个很大的数值，于是导致后面作为下标的时候数组越界了，导致地址不可访问而出错，后来我再次查看CTF-WiKi，发现文中也提到了这一点，还作为需要注意的事情写了出来，但是我当时看到这里的时候还看不懂这里是什么意思，就没有在意。这个地方困扰了我整整两天。。。

明白了问题所在之后，事情就变得简单了。可以直接使用原文中提供的二进制程序继续学习，也可以根据自己的二进制程序修改exp，将`Elf32_Sym`结构体的位置向后移，使得`r_info`计算出来的`ndx`为0,就行了。

### 总结

自己刚开始学习的时候并没有真正的理解ELF的文件结构和动态加载的过程，只是看了少数几篇博客中的讲解，并没有自己去看源码，到了后面踩坑了之后，为了找到问题关键，又从头梳理了一遍，发现自己对这个过程还不是真正的了解，才去找的源码，然后对着源码调试。后来我又拿着我自己的二进制程序和原文中给的二进制程序进行对比调试才真正发现了问题。虽然花费了两天的时间，但是终于把这个知识点搞懂了。

以后再学习新知识的时候一定要有源码的去看源码，基础知识要先理解，不然遇到问题会一头雾水。