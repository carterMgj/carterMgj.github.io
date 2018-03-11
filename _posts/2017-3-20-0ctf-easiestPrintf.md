---
layout:     post
title:      "0ctf-easiestPrintf"
subtitle:   "fsb漏洞利用新思路"
date:       2017-12-20 21:16:11
author:     "Carter"
header-img: "img/wallhaven-448464.jpg"
tags:
    - pwn
---

### 写在前面

刚刚参加完0ctf，这道题费了我很多时间思考，最终还是没有getshell（愁苦脸）。赛后看了大佬的wp，其利用思路是我第一次见，在本文将进行阐述和记录。

### 一. 漏洞分析
>内存泄露

在此可以通过printf函数输出一个内存地址内容，进而泄露libc
![img](https://raw.githubusercontent.com/carterMgj/blog_img/master/2017-3-20-0ctf-easiestPrintf/1.png)

>格式化字符串

很明显的一个printf格式化字符串漏洞
![img](https://raw.githubusercontent.com/carterMgj/blog_img/master/2017-3-20-0ctf-easiestPrintf/2.png)

>分析

按照常规思路，覆盖exit_got为main函数起始地址，从而多次利用fsb漏洞，但是在此程序会随机的抬高栈顶，导致无法预测printf参数相对栈顶的偏移。当时的我，思路就卡在了这个地方...

### 二. 漏洞利用

>基本知识

该题目苛刻的条件，要求我们必须在执行完printf函数后就能够控制EIP寄存器，因此介绍一种“新奇”的利用思路

printf函数在输出的内容较多的时候，会调用malloc分配缓冲区，输出结束后会调用free释放申请的内存。因此要在printf函数结束前控制EIP，思路是通过覆盖__free_hook、__malloc_hook劫持EIP

在调试过程中，将会运用到glibc源码调试相关的内容，介绍如下：

###### glibc源码调试
1. 下载带符号的libc库
```sh
sudo apt-get source libc6-dbg
```

2. 本题中malloc和free函数的实现在malloc.c中
3. find指令查询到malloc.c文件的绝对路径为：  /lib/glibc-2.23/malloc/malloc.c
4. 在gdb中，通过 directory将c文件目录引入
```sh
directory /lib/glibc-2.23/malloc
```

5. l 指令，可以查看指定列的libc的源文件
  ![img](https://raw.githubusercontent.com/carterMgj/blog_img/master/2017-3-20-0ctf-easiestPrintf/9.png)

>利用思路

1. 程序提供了一次读取任意地址数据的机会，可以泄露libc的地址，从而得到free_hook和malloc_hook的值

2. 先把free_hook修改为leave函数的入口地址，同时往指定内存写入一个"sh\x00"字符串;第二次触发漏洞时修改malloc_hook为system地址，输出的字符数量为保存"sh\x00"字符串的地址，那么在触发malloc操作时就可以getshell了

>利用过程

在printf函数处第一次断下，利用fsb漏洞修改free_hook地址内容为leave函数入口地址，同时将'/bin/sh\x00'字符串放入到bss段。为了能够调用malloc函数，在此将printf函数输出的字符填充到1000000个字符
![img](https://raw.githubusercontent.com/carterMgj/blog_img/master/2017-3-20-0ctf-easiestPrintf/3.png)

在free函数下断，检查 free_hook 的值是否被改变
![img](https://raw.githubusercontent.com/carterMgj/blog_img/master/2017-3-20-0ctf-easiestPrintf/4.png)

加入源码调试，可以更加清晰的看到函数的调用过程。在free函数开头处，会检查free_hook处是否为0，如果非0，就跳转到free_hook中存储地址执行
![img](https://raw.githubusercontent.com/carterMgj/blog_img/master/2017-3-20-0ctf-easiestPrintf/5.png)

执行完free_hook后，程序跳转到leave函数，进而可以再一次触发printf

再一次利用fsb修改malloc_hook的值为system函数地址,并且控制输出的字符数量为保存"sh\x00"字符串的地址。在调用malloc时断下
![img](https://raw.githubusercontent.com/carterMgj/blog_img/master/2017-3-20-0ctf-easiestPrintf/6.png)

继续运行便可getshell




