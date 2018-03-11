---
layout:     post
title:      "pwn--数组越界读写"
subtitle:   "通过题目，实战理解数组越界读写漏洞"
date:       2017-11-1 23:36:11
author:     "Carter"
header-img: "img/wallhaven-524071.jpg"
tags:
    - pwn
---

### 写在前面

数组越界读写是一种很常见的漏洞，其成因是因为对数组下标值检验不严格。在实际的漏洞利用过程中，既可以任意内存读，泄露关键地址，突破aslr、pie等令人头疼的保护机制；同时也可以任意地址写，劫持控制流。可以说是“居家旅行必备的攻击武器”。下面就通过两个例子来实际感受一下它的魅力。

###  TUCTF guestbook

#### 1. 题目描述
题目功能很简单，首先默认分配四块堆，存储四个guest的name。然后三个功能，如下图所示
![img](https://raw.githubusercontent.com/carterMgj/blog_img/master/2017-11-01-array-over-read-and-write/1.png)

#### 2. 漏洞分析
>数组越界读

使用View name”功能可以进入到readName函数中，其中对于输入的数字只进行了大于等于0的判断，没有进行小于等于3的判断，导致可以通过数组越界读，读取栈上高地址的值
![img](https://raw.githubusercontent.com/carterMgj/blog_img/master/2017-11-01-array-over-read-and-write/2.png)

>数组越界写

数组越界写：漏洞成因和上面一样，可以往栈上高地址任意写。因为任意写的输入来自gets函数，所以可以任意写的字节数理论上是无限多的（在此gets函数可导致栈溢出，但是不可用此直接覆盖返回地址，因为特殊的栈布局，在覆盖到返回地址前会覆盖到一个变量，导致在ret时程序崩溃，第四部分详讲）
![img](https://raw.githubusercontent.com/carterMgj/blog_img/master/2017-11-01-array-over-read-and-write/3.png)

#### 3. 漏洞利用
程序故意将system地址放在栈上，首先利用“数组越界读”可以读取system地址和四个堆块的起始地址
![img](https://raw.githubusercontent.com/carterMgj/blog_img/master/2017-11-01-array-over-read-and-write/4.png)

注意到dest[6]这个地址，该地址是一个“三重跳地址”，内容指向dest[0]。利用栈上这个特殊的“三重跳”地址，通过“数组越界写”修改dest[6]，即可让gets函数读入payload时从栈上dest[0]开始覆盖，直到覆盖到返回地址为system，拿到shell


#### 题目坑点
>为什么不能用gets直接覆盖栈上的返回地址？
>![img](https://raw.githubusercontent.com/carterMgj/blog_img/master/2017-11-01-array-over-read-and-write/5.png)

从上图可以看到，gets覆盖的途中会覆盖到v5这个变量，如下图所示，该变量最后在dest[v5]中引用，如果被覆盖为很大的数字，那么在strcpy时，会造成非法地址写，程序直接崩溃掉
![img](https://raw.githubusercontent.com/carterMgj/blog_img/master/2017-11-01-array-over-read-and-write/6.png)



###  PinganCTF supermarket

#### 1. 题目描述
典型的菜单栏程序：
 - 选择a：购买东西
 - 选择b：查看购物车
 - 选择c：结账
 - 选择d：退出
     ![img](https://raw.githubusercontent.com/carterMgj/blog_img/master/2017-11-01-array-over-read-and-write/7.png)

#### 2. 漏洞分析
>数组越界写

问题出在Item List函数内，该函数是列出各个商品的价格，用户选择相应的编号进行购买。
![img](https://raw.githubusercontent.com/carterMgj/blog_img/master/2017-11-01-array-over-read-and-write/8.png)

每购买一个商品，下图 漏洞代码2 处的代码，就会把 "栈上的一个地址x" 加1，而地址x则是由 一个基地址减去输入的数字决定的。理想情况下是输入正数，因此只会把栈上某个低地址自增1，为了保证输入为正数，出题人通过 漏洞代码1 处的代码，检查输入的第一个字符是否是负号，这种检查方式显然很容易被绕过，从而输入负数
![img](https://raw.githubusercontent.com/carterMgj/blog_img/master/2017-11-01-array-over-read-and-write/9.png)

如果输入为 “空格-1”，则可以绕过 漏洞代码1 的检查，从而修改栈上高地址内容。由下图可以看到，已经成功的购买了商品
![img](https://raw.githubusercontent.com/carterMgj/blog_img/master/2017-11-01-array-over-read-and-write/10.png)

#### 3. 漏洞利用
利用漏洞 修改栈上存储 main_0 函数的返回地址处内容为一个我们可控输入的地址。在此在checkout函数中会让我们输入cart和number，内容均存放在bss段中，因此我们可以选取把返回地址修改为该bss段的某个地址
![img](https://raw.githubusercontent.com/carterMgj/blog_img/master/2017-11-01-array-over-read-and-write/11.png)

然而，存储main_0函数返回地址的栈地址原本内容为 0x80485dc
![img](https://raw.githubusercontent.com/carterMgj/blog_img/master/2017-11-01-array-over-read-and-write/12.png)

由于漏洞只能让 某个地址内容+1（不能减小），所以我们修改  0x80485dc 中的 '\x85' --> '\xb1'，这样返回地址被修改为： 0x804b1dc，就位于存储number的bss段地址内部了，从该地此开始布置shellcode即可拿shell
![img](https://raw.githubusercontent.com/carterMgj/blog_img/master/2017-11-01-array-over-read-and-write/13.png)

