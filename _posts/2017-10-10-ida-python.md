---
layout:     post
title:      "IDA-python 的小小总结"
subtitle:   "从实际出发，感受IDA-python的魅力"
date:       2017-10-10 23:36:11
author:     "Carter"
header-img: "img/post-bg-os-metro.jpg"
tags:
    - IDA
    - python
---

### 写在前面

一直听说IDA python很强大，这段时间花了些时间做了些研究，并且根据在CTF中pwn题的实际需要，开发了两个小小功能的脚本来练练手，算是基本入了门。下面来做一个小小的总结

###  一. 基本介绍

   IDA python在2004年被开发出来，其目的是为了将python语言的简洁强大和IDA支持的IDC语言结合起来。

   IDA python有三个独立的模块组成：
      1. idc：这是兼容idc函数的模块
      2. idautils：很使用的一个模块，大多数处理都是需要依托于这个模块
      3. idaapi：允许使用者通过类的形式，访问更多底层的数据，

IDA python中的函数名一律采用“驼峰命名法”，而且是大小写敏感的。

### 二. 功能介绍

**IDA python到底可以做些什么呢？**我想其可以做的事情太多了，根据我目前的理解，是很难回答完全的，因为不同需求的用户对IDA python的感受是不同的。  
在我看来，其可以让我们对于一个二进制程序处理跟踪到指令寄存器级别，加深我们对于程序的理解。下面通过列举一些指令，让大家感受一下其语法用法和相关功能，希望对各位的研究功能有所启发。

>指令处理

1. 获取当前指令地址：ea=here()   print “0x%x %s”%(ea,ea)
2. 获取当前的汇编指令：idc.GetDisasm(ea)
3. 获取当前处于的段：idc.SegName()
4. 获取该段程序最低和最高地址：hex(MinEA())   hex(MaxEA())
5. 获取下(上)一条汇编指令地址：ea = here() ;  next_instr = idc.NextHead(ea)   PrevHead(ea)

>函数操作

1. 获取程序所有函数的名称：
```python
for func in idautils.Functions():
	print hex(func), idc.GetFunctionName(func)
```
2. 计算当前函数有多少条指令
```python
ea = here()
len(list(idautils.FuncItems(ea)))
```
3. 获取当前IDB中记录的所有有名字函数的地址和名称： idautils.Names()
  返回的是一个以元组组成的列表，函数的起始地址指向了其plt表
  ![img](https://raw.githubusercontent.com/carterMgj/blog_img/master/2017-10-10-IDA-python/3.png'')

>指令操作

1. 给定地址，打印指令  idc.GetDisasm(ea)
2. 给定函数中一个地址，得到整个函数的指令列表   idautils.FuncItems(here())
3. 获取函数的一些flag信息：  idc.GetFunctionFlags(func)
4. 对一条汇编指令进行拆解：
   获取指令的操作：idc.GetMnem(here())
   获取指令的操作数：idc.GetOpType(ea,n)   根据返回的数值，可以判断操作数类型（普通寄存器、常量字符串等）
5. 对汇编指令中用到的操作数，求取其引用的地址，也就是双击该操作数后跳转到的地址  hex(idc.GetOperandValue(here(),1))

>交叉引用

1. 指令从哪个地方来：idautils.CodeRefsTo(here(),0)
2. 指令下一步去到哪儿：idautils.CodeRefsFrom(here(),0)
3. 数据从哪个地方来：idautils.DataRefsTo(here(),0)
4. 数据下一步去到哪儿：idautils.DataRefsFrom(here(),0)
5. 较为通用的一种获取xref：idautils.XrefsTo(here(),flag)  其中flag=0时，所有的交叉引用都会被显示

### 三. 现学现卖

学习理论知识必须和实践结合起来才有意义。因此我边学边尝试着结合自己的需求写一些IDA-python的小脚本来练练手。下面我贴出两个小脚本，从中可以体会到IDA-python在处理二进制程序时的方便和强大。

>发现危险函数

在做CTF比赛pwn题时，尽快发现漏洞就能先人一步。而容易出漏洞的函数基本就那一些。因此该脚本的功能就是扫描整个程序的函数列表，列出存在于“危险列表”中的函数的起始位置和名称，以便我们有针对性的审计源码，快速发现漏洞进行利用。

```python
import idautils

danger_func = ["strcpy","strncpy","memcpy","memncpy","gets","read"]

for func in idautils.Functions():
	if idc.GetFunctionName(func) in danger_func:
		print hex(func), idc.GetFunctionName(func)
```

>格式化字符串漏洞自动发现

格式化字符串漏洞在日常中很普遍，因此我想着编写一个小脚本自动来发现程序中的格式化字符串漏洞。下面我们通过 ：不存在和存在 格式化字符串的几段汇编代码比较，从中发现如何判断一段代码是否存在该漏洞。

**不存在格式化字符串漏洞的printf函数调用**
```c
printf("level:%d\n",v8);
printf("secret:%s\n",src);
```
![img](https://raw.githubusercontent.com/carterMgj/blog_img/master/2017-10-10-IDA-python/4.png)
***
```c
printf("Your Message: ");
```
![img](https://raw.githubusercontent.com/carterMgj/blog_img/master/2017-10-10-IDA-python/6.png)



**存在格式化字符串漏洞的printf函数调用**

```c
printf("&dest");
```
![img](https://raw.githubusercontent.com/carterMgj/blog_img/master/2017-10-10-IDA-python/5.png)
***
```c
printf((const char *)ptr);
```
![img](https://raw.githubusercontent.com/carterMgj/blog_img/master/2017-10-10-IDA-python/8.png)

从上面几个例子可以看到，判断时需要紧密关注printf函数的第一个参数中的值，是怎么来的。对于32bit和64bit程序，其由于传参顺序不同，因此对于32bit程序我们需要关注 *esp，对于64bit程序，我们需要关注rdi ；当存在格式化字符串漏洞时，其第一个参数必定来自于寄存器或者寄存器与某个常量相加后的值。（但反之不成立）

```python
import idautils

'''find the fsb instruction addresses.The basic thing is below:
 func_got -> func_plt -> use xref to get addresses of "call func" ->  judge the pre_instrution's
 second operand if whether original registers. 
'''

def GoKeyInstrution(start):
	now_intr = start
	while 1:
		intr = idc.GetDisasm(now_intr)
		if 'esp' in intr or 'rdi' in intr:
			return now_intr
		else:
			now_intr = idc.PrevHead(now_intr)

call_array = []
vuln_array = []
func_name = 'printf'
func_got = idc.LocByName(func_name) 
print 'func_got='+hex(func_got)

if idc.SegName(func_got)=='.plt.got':
	print 'Located in .got...............'
	for addr in idautils.CodeRefsTo(func_got, 0):
		if idc.GetMnem(addr) == "call":
			call_array.append(addr)
elif idc.SegName(func_got)=='extern':
	print 'Located in extern..............'
	for func_plt in idautils.CodeRefsTo(func_got, 0):
		print 'printf_plt=' + hex(func_plt)
		for addr in idautils.CodeRefsTo(func_plt, 0):
			if idc.GetMnem(addr) == "call":
				call_array.append(addr)


for item in call_array:
	keyInstr = GoKeyInstrution(item)
	print 'Key-Instruction:'
	print hex(keyInstr), idc.GetDisasm(keyInstr)
	if GetMnem(keyInstr) == "mov":
		if GetOpType(keyInstr,1) == 1 or GetOpType(keyInstr,1) == 3:
			vuln_array.append(item)

if len(vuln_array)>0:
	print 'This instruction is vulnerable'
	for item in vuln_array:
		print hex(item), idc.GetDisasm(item)
```
以上代码大致思路如下：
1. 得到printf_got表地址
2. 根据printf_got表地址，利用交叉引用，回溯找到printf_plt表地址
3. 根据printf_plt表地址，利用交叉引用，回溯找到调用printf_plt表地址指令的指令集合
4. 如果该指令是call指令，则往前找指令，找到给第一个参数赋值的那条汇编语句
5. 判断该条汇编语句的第二个参数是否是寄存器或寄存器与常量相加值，如果是则判定存在漏洞（一定程度上存在误判的可能，但'宁可错杀三千，不肯放走一个'）

以下是该脚本判定结果，测试的几个CTF题目均可以直接定位到漏洞点：

>32bit下发现fsb漏洞

![img](https://raw.githubusercontent.com/carterMgj/blog_img/master/2017-10-10-IDA-python/1.png)

>64bit下发现fsb漏洞

![img](https://raw.githubusercontent.com/carterMgj/blog_img/master/2017-10-10-IDA-python/7.png)

>64bit下未发现fsb漏洞

![img](https://raw.githubusercontent.com/carterMgj/blog_img/master/2017-10-10-IDA-python/2.png)




```

```