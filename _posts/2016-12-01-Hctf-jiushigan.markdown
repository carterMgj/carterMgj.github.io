---
layout:     post
title:      "2016-Hctf题解 -- 就是干"
subtitle:   "fast-bin内存机制的探究利用"
date:       2016-12-01 16:55:25
author:     "Carter"
header-img: "img/post-bg-2015.jpg"
tags:
    - Linux
    - pwn
    - UAF
---



### 一. 知识点：
**1. 代码段开了随机化怎么调试：**

需要两个文件，其一是debug，通过peda获取每次程序加载代码段基址，内容如下：

```javascript
python
code_addr = peda.get_vmmap('binary')[0][0]
peda.execute('set $code='+hex(code_addr))
end
source aa
```

另一个是断点文件aa，通过source指令链接到debug文件中，内容如下：

```python
break * $code + 0xe93
break * $code + 0xd65
```

在exp中只需要加入一句话就可以挂载gdb，并在启动gdb后自动加载断点信息

```python
gdb.attach(pidof('pwn1')[-1],open('debug'))
```

### 二. 题目描述：
该题主要有两个功能：

 - 一是开辟内存输入字符串
 - 二是删除字符串并释放该内存块

明显就是考察申请释放堆块的相关姿势。首先来看**输入字符串函数**：

每个字符串都会分配一个长度为32的堆块A「实际分配长度为48，包括16字节堆块头」。接着判断输入字符串长度，如果长度小于15，则直接把字符串存储在该堆块A中；如果大于15，则另外开辟一块内存B存储字符串，堆块A中仅仅存储堆块B的地址。

**长度小于15的字符串堆块：**

布局如下：
![图片](https://raw.githubusercontent.com/carterMgj/blog_img/master/2016-12-01-Hctf-jiushigan/10.png)
对应内存如下：
![图片](https://raw.githubusercontent.com/carterMgj/blog_img/master/2016-12-01-Hctf-jiushigan/1.png)

**长度大于15的字符串堆块：**
![图片](https://raw.githubusercontent.com/carterMgj/blog_img/master/2016-12-01-Hctf-jiushigan/2.png)
内存中还有一个堆块总管理地址，对每个字符串有两个变量来标志它：1.该字符串是否被删除的标志位; 2.为字符串分配的堆块地址
![图片](https://raw.githubusercontent.com/carterMgj/blog_img/master/2016-12-01-Hctf-jiushigan/3.png)

再来看看**删除字符串函数**:首先判断字符串对应的堆管理结构的第二个位置是否为0[**漏洞所在：第二个位置是函数地址永远不可能为0,判断无效**]，然后调用堆块中存储的'处理删除字符串函数'。如果我们可以覆盖此函数地址为其他地址，再释放该堆块，那么就可以任意函数执行了
![图片](https://raw.githubusercontent.com/carterMgj/blog_img/master/2016-12-01-Hctf-jiushigan/4.png)



### 三. 漏洞利用：
1. 首先输入3个长度小于15的字符串，申请3块内存，此时堆管理结构如下：
![图片](https://raw.githubusercontent.com/carterMgj/blog_img/master/2016-12-01-Hctf-jiushigan/5.png)
2. 接着删除字符串1、2、3，因为堆块A、B、C都是小于128字节，所以分配和释放都用fast-bin方式进行。释放后的堆块在链表中将后进先出，此时该fast-bin链如下：
C --> B --> A 

3. 再输入一个长度大于15的字符串：首先会分配一块32字节的堆，此时将分配到空闲块C；接着分配一个字符串长度的堆块，此时应该保证字符串长度在15-32之间，就可以顺利的分配到空闲堆块B，然后字符串的内容由我们输入存放在堆块B中，就可以覆盖原本的'处理删除字符串函数'为我们想调用的函数printf

在这里有一个坑：printf函数的偏移为0x9d0，但是我们只能覆盖整数字节，也就是说得把原本'处理删除字符串函数'的后面4个字母数覆盖为'x9d0'，其中x是一个未知数，有16种可能，在这里只能fuzz撞概率咯~

覆盖前：
![图片](https://raw.githubusercontent.com/carterMgj/blog_img/master/2016-12-01-Hctf-jiushigan/6.png)
覆盖后：
![图片](https://raw.githubusercontent.com/carterMgj/blog_img/master/2016-12-01-Hctf-jiushigan/7.png)
4. 接着释放第2块会调用存放于堆块B中的'处理删除字符串函数'，此时已经被我覆盖为printf函数的地址
![图片](https://raw.githubusercontent.com/carterMgj/blog_img/master/2016-12-01-Hctf-jiushigan/8.png)
由调试可知，参数正好是输入字符串，所以就可以了任意函数执行并控制参数
![图片](https://raw.githubusercontent.com/carterMgj/blog_img/master/2016-12-01-Hctf-jiushigan/9.png)
5. 顺利的调用了printf('%170$p')，可以泄露存放在栈上的__libc_start_main+240的地址，从而可以计算出system函数的地址

6. 接着再执行一遍以上过程，从新触发任意函数执行的漏洞，执行system('/bin/sh')即可获取shell

### 唠叨几句：
这是第一次接触到还带猜的pwn，所以详细的记录了一下解题过程。猜的过程也比较有意思，由于主办方采取了一定的防DOS措施：当检测到访问过于频繁时就会禁止访问该题。加之学校ip出口都是一个，校内还有一些大佬也在fuzz这个题，所以导致fuzz过程异常艰难！

总的来说，这个题还是出得很棒的，给主办方杭州电子科技大的大佬们赞一个~