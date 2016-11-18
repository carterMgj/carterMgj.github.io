---
layout:     post
title:      "windows格式化字符串漏洞探究"
subtitle:   "集Fsb漏洞构造、分析、利用的小实验"
date:       2016-11-18 11:23:25
author:     "Carter"
header-img: "img/post-bg-apple-event-2015.jpg"
tags:
    - Windows
    - pwn
---

### 一. 关于程序调用时压栈顺序
对于函数
```c++
func(arg[1],arg[2],.....,arg[n])
```
压栈顺序为：参数（从右到左）--> 返回地址 --> ebp

压栈后，栈上情况如下：
***
低地址：
　　　ebp
　　　函数返回地址
　　　arg[1]
　　　arg[2]
　　　 ....
　　　arg[n]　　
高地址：
***
因此存在诸如格式化字符串等任意内存写漏洞时，Get shell最直接的方法就是：

 改写程序返回地址为so库中system函数的地址，同时布置好栈，将参数“/bin/sh\x00”放在'返回地址往后两个单位内存地址'处即可 。为什么要放在其后两个单位内存地址处，原因见（探究程序进入到子函数时，栈的状态）讲解
 
当程序执行完到ret指令时，跳到system函数地址，就会把 /bin/sh 当做参数执行，获得shell


### 二. 探究程序进入函数前后，栈的状态
实验程序代码如下：
```c++
#include <stdio.h>
int  print(char* s)
{
	return 0;
}

int main()
{
	char s[32] = "Hello world";
	print(s);
 	return 0;
}
```
在print函数处下断，程序停在了调用print函数的地方

![图片](https://raw.githubusercontent.com/carterMgj/blog_img/master/2016-11-18-pwn-base-knowledge/1.png)

跟进到print函数，进入到函数的时候，栈的情况如下

![图片](https://raw.githubusercontent.com/carterMgj/blog_img/master/2016-11-18-pwn-base-knowledge/2.png)
</br>
</br>
二. 关于大端模式和小端模式：
> 小端模式：数据的高位对应高地址
> 大端模式：数据的高位对应低地址

　　举个栗子：对于 0xf76a3a84 这个数，按照其他进制计数规则，左边为高位，右边为地位，因此 0xf7 为最高位，相反，0x84为最低位

![图片](https://raw.githubusercontent.com/carterMgj/blog_img/master/2016-11-18-pwn-base-knowledge/3.png)

　　因为内存中地址从左到右是依次增大的，所以 0xf76a3a84 这个数在内存中实际分布为：
　　
![图片](https://raw.githubusercontent.com/carterMgj/blog_img/master/2016-11-18-pwn-base-knowledge/4.png)

也就是数据最高位对应高地址，最低位对应低地址

### 三.  关于ELF程序的几个段
一个程序一般分为3段:text段,data段,bss段
text段:  可读可执行，存放程序代码,编译时确定
data段:  只可读，存放在编译阶段(而非运行时)就能确定的数据,
就是通常所说的静态存储区,赋了初值的全局变量、静态变量和常量存放在这个区域
bss段:   可读可写，定义时没有赋初值的全局变量和静态变量,放在这个区域，比如程序的got表地址就位于该区域

如图：

![图片](https://raw.githubusercontent.com/carterMgj/blog_img/master/2016-11-18-pwn-base-knowledge/5.png)

1.  可读可执行，为程序的 text段--代码段
2.  只可读，为程序的 bss段
3.  可读可写，为程序的 data段

注意：当利用漏洞修改某个地址的值时，首先需要把握整个程序结构，搞清楚该变量是什么变量，其地址位于程序的哪个段，这个段有写入权限你才能修改，否则程序会报‘Segment Defaults’



### 四.   got表和plt表
 在linux漏洞利用中，很重要的两个概念

首先需要记住：

 - plt表中存放的是一段汇编代码(jmp *got_addr)
 - got表中存放的仅仅是一个地址数据，即so库中某函数加载的内存地址

在函数func被调用时，首先会跳转到func的plt表地址，该地址存放了一段jmp *addr的汇编代码，在该函数第一次调用时，addr中存放的是：查询func函数加载到内存的真正地址的相应代码的地址；查询到后，函数真实地址被写入到函数got表地址：got_addr中，当第二次调用时，plt表存储的汇编代码为 jmp *got_addr，即跳转到函数真正代码处执行

![图片](https://raw.githubusercontent.com/carterMgj/blog_img/master/2016-11-18-pwn-base-knowledge/6.png)


五.  二进制文件与python利用脚本交互 调试技巧

 2. 在写的python脚本中写入：
   from pwn import *
   conn = process('./pwn1')   #echo为要进行pwn的程序名称
   raw_input('pause')
 3. 运行 python 脚本
 4. 输入sudo gdb pwn1 `pidof pwn1` 进入到gdb调试，pdisass反汇编一下函数或者IDA静态查看，在你感兴趣的位置下好断点
 5. 在gdb中输入 c (continue) ，让程序继续执行
 6. 此时的python脚本会停在 raw_input('pause')这句，等待输入；当gdb attach上去并且下断都完成以后，在python脚本的bash里按一下回车，python脚本就跑起来了，就可以开始一步步跟踪python脚本与程序的交互过程了  

### 六. 对于 leave ，ret指令的理解
leave做的事情：

 - new_esp  = old_ebp + 4 （因为栈的地址是从高地址往低地址增长）
 - new_ebp = * old_ebp

如下图所示，在运行leave指令之前，EBP和ESP情况如下：

![图片](https://raw.githubusercontent.com/carterMgj/blog_img/master/2016-11-18-pwn-base-knowledge/7.png)

运行完leave指令后，新的EBP和ESP情况如下：

![图片](https://raw.githubusercontent.com/carterMgj/blog_img/master/2016-11-18-pwn-base-knowledge/8.png)

ret做的事情：
    跳转到栈顶位置内容所代表的地址
    
![图片](https://raw.githubusercontent.com/carterMgj/blog_img/master/2016-11-18-pwn-base-knowledge/9.png)

运行完ret指令 后，程序调转到 0x8049407 处，同时 ESP = ESP+4

### 七. 对于call指令对栈帧的影响探究
**32bit下：**

 call指令执行结束后会降低栈帧，降低的大小取决于该函数参数的个数n，即：栈帧esp降低大小 = n*4  (n为函数参数个数)
 
 原因也比较好理解，在调用call前，push指令将该函数的参数依次压入到栈中；在call指令执行后，将有相应多的pop指令将参数弹出，从而降低栈帧，最后返回的时候，自然esp也就降低了
    
**64bit下：**

因为64bit下，函数调用是通过寄存器传递参数

> rdi > rsi > rdx > rcx > r8 > r9


所以函数调用前后，对栈是没有影响的





