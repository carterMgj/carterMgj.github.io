---
layout:     post
title:      "linux漏洞利用方式ROP探究"
subtitle:   "全是干货，一点即燃"
date:       2016-11-18 16:55:25
author:     "Carter"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - Linux
    - pwn
    - ROP
---



### 一. 系统调用法：
**适用情形:** 在gadget中能找到系统调用指令  ----‘ int 80 ’

x86_64系统调用时寄存器状况：

 - rax：0x3b,
 - rdi：'/bin/sh'地址
 - rsi：0
 - rdx：0

**构造思路：**

##### 1. 布置寄存器
在64bit系统中，参数需要通过 rdi、rsi、rdx进行传递，所以需要考虑如何将相应的数值放到寄存器中。需要寻找类似“pop edi”这样的指令，或者'曲线救国'寻找如下这样的指令集合
   
```c++
pop r12
mov edi r12
```


#### 2. 获取字符串'/bin/sh'在内存中地址 
对于如何获得'/bin/sh'字符串在内存中的地址，谈两点思路：
 - 思路一是调用scanf函数，控制scanf相关参数，将字符串读入到指定bss段地址
 - 思路二是利用已经泄露的so库函数地址，在IDA中静态查看so库中'/bin/sh'字符串的偏移值，通过计算得到字符串加载到的内存地址

![图片](https://raw.githubusercontent.com/carterMgj/blog_img/master/2016-11-18-rop-summary/3.png)

#### 3. 将'/bin/sh'字符串内存地址写入rax
需要寻找类似如下构造的指令序列（当然思路方法有很多，最终目的达到就ok了）

```
pop xxx
pop yyy
mov xxx，dword ptr[yyy]
```


### 二. x86_64架构下，libc特殊结构的利用
在64bit程序中会存在如下一段代码，该段代码应该是完成程序初始化或者其他的一些功能，但成为我们利用rop的一把利器

通过命令：objdump -S ./bin_name 反汇编二进制文件，可以找到该段神奇的代码

![图片](https://raw.githubusercontent.com/carterMgj/blog_img/master/2016-11-18-rop-summary/1.png)

**利用代码：**

```python
def ropInput(func_addr,first,second ,third,ret_addr):
	rop_addr1=  0x400d9a
	rop_addr2 = 0x400d80
	input =   p64(rop_addr1) + p64(0) + p64(1) + p64(func_addr) + p64(third) + p64(second) + p64(first) + p64(rop_addr2)  + '\x00'*56 + p64(ret_addr)
	return input
```

**使用说明：**

   1   需要根据实际情形，手动修改 rop_add1、rop_add2两个参数
   2   参数func_addr 应该是要执行函数的got表地址，不是plt表地址

**使用效果：** 
调用任何一个函数，并且能控制该函数的**前三个参数**和执行结束后的**返回地址**

**缺陷：**
该利用方式一次需要布置在栈上的字节数较多，需要**128字节**。如果在一次性读入栈上的字节数较少的情形下（如：read(100)），则不能适用

***
**改进：减少对事先栈上布置字节数的要求**

如果第一次进入到此结构时，rbx=0，则此时构造rop链长度只需要**112字节**，即可达到效果
![图片](https://raw.githubusercontent.com/carterMgj/blog_img/master/2016-11-18-rop-summary/2.png)

**利用代码：**

```python
def ropInput(func_addr,first,second ,third,ret_addr):
	rop_addr1=  0x400d9a
	rop_addr2 = 0x400d80
	input =   p64(rop_addr1) + p64(func_addr) + p64(third) + p64(second) + p64(first) + p64(rop_addr2)  + '\x00'*56 + p64(ret_addr)
	return input
```
	
**使用说明：**同上

***
**继续改进：让事先栈上布置字节数没有最少，只有更少！**

以上两种利用代码，最低要求都能是栈上事先布置112字节的rop。如果满足以下条件，则可以将该结构分成两部分来加以利用，对事先布置的字节数要求就更少了：

 1. 该程序中可以多次触发漏洞，控制栈上内容和eip
 2. 在第一次利用漏洞返回到第二次触发漏洞期间，程序对r12-r15寄存器没有任何操作 (条件有点苛刻)

如何利用：

 1. 第一次跳到rop_addr1，布置r12-r15
 2. 返回到主函数，重新触发漏洞
 3. 第二次跳到rop_addr2，执行函数达到目的

### 该结构的实战运用：

在利用该结构时，首先需要注意的就是你能够控制栈上多少字节的数据，以上介绍的几种方法中，均对最少字节有一定的要求。其次需要考虑的一点是该结构能否重复利用，接下来就此进行分析与讨论：

##### **能够多次触发漏洞，利用该结构**
利用完一次该结构以后，可以返回到主函数开头，此时可以重新恢复因为漏洞利用所破坏的堆栈，如果可以重新再次触发漏洞，那么就简单了，重复的利用该结构执行任意函数，直到拿到shell。

举个栗子：[2016_cdctf pwn100题解](http://note.youdao.com/noteshare?id=dbfe9805a30f34e1a9916e3b7010f54f)

##### **只能触发一次漏洞，利用该结构**
如果回到主函数后就再也不能触发漏洞了，此时就要考虑**二次利用**

1. 首先利用该结构调用 read(0,addr,0x500)函数，往栈上读入任意可控长度的rop。
 > Ps: addr为栈上存储read函数返回地址的地址，需要通过泄露栈地址或者其他方式得到。
2. 然后就可以一次性把以后需要调用函数的所有参数全部放到栈上，循环的利用该结构来执行各种函数，直到最终得到shell
>  Ps：此种情况下泄露system和/bin/sh地址要在第一次进入该结构之前

举个栗子：[2016华山杯线下赛  su-pwn](http://note.youdao.com/noteshare?id=1716ffb6e01c11170b374ba6525c744c)






