---

title: 安卓逆向（一）--Smali基础
date: 2019-07-31 14:29:43
categories: 
 - Reverse
---
# 安卓逆向（一）--Smali基础

转载自[吾爱破解安卓逆向入门教程](https://www.52pojie.cn/thread-395378-1-1.html)

### APK的组成

| 文件夹              | 作用                                                         |
| :------------------ | :----------------------------------------------------------- |
| asset文件夹         | 资源目录1：asset和res都是资源目录但有所区别，见下面说明      |
| lib文件夹           | so库存放位置，一般由NDK编译得到，常见于使用游戏引擎或JNI native调用的工程中 |
| META-INF文件夹      | 存放工程一些属性文件，例如Manifest.MF                        |
| res文件夹           | 资源目录2：asset和res都是资源目录但有所区别，见下面说明      |
| AndroidManifest.xml | Android工程的基础配置属性文件                                |
| classes.dex         | Java代码编译得到的DalvikVM能直接执行的文件，下面有介绍       |
| resources.arsc      | 对res目录下的资源的一个索引文件，保存了原工程中strings.xml等文件内容 |
| 其他文件夹          | etc.                                                         |
### asset资源目录和res资源目录的不同之处：
res目录下的资源文件在编译时会自动生成索引文件（R.java），在Java代码中用R.xxx.yyy来引用；
而asset目录下的资源文件不需要生成索引，在Java代码中需要用AssetManager来访问；
一般来说，除了音频和视频资源（需要放在raw或asset下），使用Java开发的Android工程使用到的资源文件都会放在res下；使用C++游戏引擎（或使用Lua Unity3D等）的资源文件均需要放在asset下。

其中在Davlik字节码中，寄存器都是32位的，能够支持任何类型，64位类型（Long/Double）用2个寄存器表示；Dalvik字节码有两种类型：原始类型；引用类型（包括对象和数组）

### 原始类型：
​    B---byte
​    C---char
​    D---double
​    F---float
​    I---int
​    J---long
​    S---short
​    V---void
​    Z---boolean
​    [XXX---array
​    Lxxx/yyy---object
数组的表示方式是：在基本类型前加上前中括号“[”，例如int数组和float数组分别表示为：[I、[F；对象的表示则以L作为开头，格式是`LpackageName/objectName;`（注意必须有个分号跟在最后），例如String对象在smali中为：`Ljava/lang/String;`，其中java/lang对应java.lang包，String就是定义在该包中的一个对象。或许有人问，既然类是用`LpackageName/objectName;`来表示，那类里面的内部类又如何在smali中引用呢？答案是：`LpackageName/objectName$subObjectName;`。也就是在内部类前加“$”符号，关于“$”符号更多的规则将在后面谈到。
### 方法：
方法的定义一般为：`Func-Name (Para-Type1Para-Type2Para-Type3...)Return-Type`
注意参数与参数之间没有任何分隔符，同样举几个例子就容易明白了

1. `hello ()V`。
没错，这就是`void hello()`
2. `hello (III)Z`。
这个则是`boolean hello(int, int, int)`
3. `hello (Z[I[ILjava/lang/String;J)Ljava/lang/String;`
  看出来这是`String hello (boolean, int[], int[], String, long)` 了吗？
  ### Smali基本语法:
   .field private isFlag:z　　定义变量
   .method　　方法
   .parameter　　方法参数
   .prologue　　方法开始
   .line 123　　此方法位于第123行
   invoke-super　　调用父函数
   const/high16  v0, 0x7fo3　　把0x7fo3赋值给v0
   invoke-direct　　调用函数
   return-void　　函数返回void
   .end method　　函数结束
   new-instance　　创建实例
   iput-object　　对象赋值
   iget-object　　调用对象
   invoke-static　　调用静态函数
  ### 条件跳转分支：
   "if-eq vA, vB, :cond_\**"   如果vA等于vB则跳转到:cond_**
   "if-ne vA, vB, :cond_\**"   如果vA不等于vB则跳转到:cond_**
   "if-lt vA, vB, :cond_\**"    如果vA小于vB则跳转到:cond_**
   "if-ge vA, vB, :cond_\**"   如果vA大于等于vB则跳转到:cond_**
   "if-gt vA, vB, :cond_\**"   如果vA大于vB则跳转到:cond_**
   "if-le vA, vB, :cond_\**"    如果vA小于等于vB则跳转到:cond_**
   "if-eqz vA, :cond_\**"   如果vA等于0则跳转到:cond_**
   "if-nez vA, :cond_\**"   如果vA不等于0则跳转到:cond_**
   "if-ltz vA, :cond_\**"    如果vA小于0则跳转到:cond_**
   "if-gez vA, :cond_\**"   如果vA大于等于0则跳转到:cond_**
   "if-gtz vA, :cond_\**"   如果vA大于0则跳转到:cond_**
   "if-lez vA, :cond_\**"    如果vA小于等于0则跳转到:cond_**

  ### Smali中的包信息：
```
.class public Lcom/aaaaa;
.super Lcom/bbbbb;
.source "ccccc.java"
```
这是一个由ccccc.java编译得到的smali文件（第3行）
它是com.aaaaa这个package下的一个类（第1行）
继承自com.bbbbb这个类（第2行）
### 关于寄存器的知识补充：
在smali里的所有操作都必须经过寄存器来进行：本地寄存器用v开头数字结尾的符号来表示，如v0、v1、v2。
参数寄存器则使用p开头数字结尾的符号来表示，如p0、p1、p2、...
特别注意的是，p0不一定是函数中的第一个参数，在非static函数中，p0代指“this”，p1表示函数的第一个参数，p2代表函数中的第二个参数…而在static函数中p0才对应第一个参数（因为Java的static方法中没有this方法。
寄存器简单实例分析：

```const/4 v0, 0x1
   iput-boolean v0, p0, Lcom/aaa;->IsRegistered:Z
```
我们来分析一下上面的两句smali代码，首先它使用了v0本地寄存器，并把值0x1存到v0中，然后第二句用iput-boolean这个指令把v0中的值存放到com.aaa.IsRegistered这个成员变量中。
即相当于：`this.IsRegistered = true;`（上面说过，在非static函数中p0代表的是“this”，在这里就是com.aaa实例）。
### smali中的成员变量
成员变量格式是：
`.field public/private [static] [final] varName:<类型>`
对于不同的成员变量也有不同的指令。
一般来说，获取的指令有：
`iget、sget、iget-boolean、sget-boolean、iget-object、sget-object`等。
操作的指令有：
`iput、sput、iput-boolean、sput-boolean、iput-object、sput-object`等。
没有“-object”后缀的表示操作的成员变量对象是基本数据类型，带“-object”表示操作的成员变量是对象类型，特别地，boolean类型则使用带“-boolean”的指令操作。
### Smali成员变量指令简析:
`sget-object v0, Lcom/aaa;->ID:Ljava/lang/String;`
`sget-object`就是用来获取变量值并保存到紧接着的参数的寄存器中，本例中，它获取ID这个String类型的成员变量并放到v0这个寄存器中。
注意：前面需要该变量所属的类的类型，后面需要加一个冒号和该成员变量的类型，中间是“->”表示所属关系。
`iget-object v0, p0, Lcom/aaa;->view:Lcom/aaa/view;`
可以看到`iget-object`指令比`sget-object`多了一个参数，就是该变量所在类的实例，在这里就是p0即“this”.
获取array的话我们用`aget`和`aget-object`，指令使用和上述一致
`put`指令的使用和`get`指令是统一的如下：

```smali
const/4 v3, 0x0
sput-object v3, Lcom/aaa;->timer:Lcom/aaa/timer;
```
 相当于：`this.timer = null;`
注意，这里因为是赋值object 所以是null，若是boolean的话，大家想应该相当于什么呢？

```
.local v0, args:Landroid/os/Message;
const/4 v1, 0x12
iput v1, v0, Landroid/os/Message;->what:I
```
相当于：`args.what = 18;`（`args`是`Message`的实例）
### Smali中函数的调用:
smali中的函数和成员变量一样也分为两种类型，分别为`direct`和`virtual`之分。
那么`direct method`和`virtual method`有什么区别呢？
简单来说，`direct method`就是`private`函数，其余的`public`和`protected`函数都属于`virtual method`。所以在调用函数时，有`invoke-direct`，`invoke-virtual`，另外还有`invoke-static`、`invoke-super`以及`invoke-interface`等几种不同的指令。
当然其实还有`invoke-XXX/range` 指令的，这是参数多于4个的时候调用的指令，比较少见，了解下即可。
#### **1.`invoke-static`：用于调用static函数的**
例如：
`invoke-static {}, Lcom/aaa;->CheckSignature()Z `
这里注意到`invoke-static`后面有一对大括号“{}”，其实是调用该方法的实例+参数列表，由于这个方法既不需参数也是`static`的，所以{}内为空，再看一个：

```smali
const-string v0, "NDKLIB" 
invoke-static {v0}, Ljava/lang/System;->loadLibrary(Ljava/lang/String;)V
```
这个是调用`static void System.loadLibrary(String)`来加载NDK编译的so库用的方法，同样也是这里v0就是参数"NDKLIB"了。
#### **2.invoke-super：调用父类方法用的指令，一般用于调用onCreate、onDestroy等方法。**
#### **3.invoke-direct：调用private函数：**
`invoke-direct {p0}, Landroid/app/TabActivity;-><init>()V`
这里`init()`就是定义在`TabActivity`中的一个`private`函数

#### **4.invoke-virtual：用于调用protected或public函数，同样注意修改smali时不要错用invoke-direct或invoke-static：**

```
sget-object v0, Lcom/dddd;->bbb:Lcom/ccc;
invoke-virtual {v0, v1}, Lcom/ccc;->Messages(Ljava/lang/Object;)V
```
这里相信大家都已经很清楚了：
v0是`bbb:Lcom/ccc`
v1是传递给`Messages`方法的`Ljava/lang/Object`参数。

#### **5.invoke-xxxxx/range：当方法的参数多于5个时（含5个），不能直接使用以上的指令，而是在后面加上“/range”，range表示范围，使用方法也有所不同：**

```
invoke-direct/range {v0 .. v5}, Lcmb/pb/ui/PBContainerActivity;->h(ILjava/lang/CharSequence;Ljava/lang/String;Landroid/content/Intent;I)Z
```
需要传递v0到v5一共6个参数，这时候大括号内的参数采用省略形式，且需要连续。
###Smali中函数返回的结果的操作:
在Java代码中调用函数和返回函数结果可以用一条语句完成，而在Smali里则需要分开来完成，在使用上述指令后，如果调用的函数返回非void，那么还需要用到`move-result`（返回基本数据类型）和`move-result-object`（返回对象）指令：

```
const-string v0, "Eric"
invoke-static {v0}, Lcmb/pbi;->t(Ljava/lang/String;)Ljava/lang/String;
move-result-object v2
```
v2保存的就是调用t方法返回的`String`字符串。

### Smali中函数实体分析--if函数分析：

```smail
.method private ifRegistered()Z
    .locals 2	//在这个函数中本地寄存器的个数
    .prologue
    const/4 v0, 0x1     // v0赋值为1
    .local v0, tempFlag:Z	
    if-eqz v0, :cond_0            // 判断v0是否等于0，等于0则跳到cond_0执行
    const/4 v1, 0x1            // 符合条件分支
    :goto_0	//标签
    return v1	//返回v1的值
    :cond_0	//标签
    const/4 v1, 0x0            // cond_0分支
    goto :goto_0	//跳到goto_0执行 即返回v1的值  这里可以改成return v1  也是一样的
.end method
```
### Smali中函数实体分析--for函数分析：

```
const/4 v0, 0x0   //vo =0;
.local v0, i:I
:goto_0
if-lt v0, v3, :cond_0     //  v0小于v3 则跳到cond_0并执行分支 :cond_0
return-void
    :cond_0                // 标签
iget-object v1, p0, Lcom/aaa/MainActivity;->listStrings:Ljava/util/List;        // 引用对象
const-string v2, "Eric"
invoke-interface {v1, v2}, Ljava/util/List;->add(Ljava/lang/Object;)Z    // List是接口, 执行接口方法add
add-int/lit8 v0, v0, 0x1　　　　// 将第二个v0寄存器中的值，加上0x1的值放入第一个寄存器中, 实现自增长
goto :goto_0                // 回去:goto_0标签
```
### Smali课后习题，翻译成Java代码。

```
    .locals 4
    const/4 v2, 0x1
    const/16 v1, 0x10
    .local v1, "length":I
    if-nez v1, :cond_1
    :cond_0
    :goto_0
    return v2
    :cond_1
    const/4 v0, 0x0
    .local v0, "i":I
    :goto_1
    if-lt v0, v1, :cond_2
    const/16 v3, 0x28
    if-le v1, v3, :cond_0
    const/4 v2, 0x0
    goto :goto_0
    :cond_2
    xor-int/lit8 v1, v1, 0x3b
    add-int/lit8 v0, v0, 0x1
    goto :goto_1
```

