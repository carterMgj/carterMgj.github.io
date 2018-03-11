---
layout:     post
title:      "peach框架源码阅读"
subtitle:   "多实践、勤总结"
date:       2018-02-02 17:35:33
author:     "Carter"
header-img: "img/post-bg-ios9-web.jpg"
tags:
    - pwn
---



# peach源码阅读

### 前言
接触peach这个框架也有一年多时间了，平时只是停留在使用层面，最近有一个论文思路需要在peach的源码上进行修改来验证，借此机会好好的看了两天peach源码。

网上关于peach源码介绍的资料基本没有，现做一个记录，权当抛砖引玉，希望能多多交流。

PS：此次peach源码版本为3.1.124

##一. vs2010调试环境搭建
peach官方提供了源码编译方法（http://community.peachfuzzer.com/v3/Installation.html）
但是不能通过调试器进行调试，因此首先搭建了vs2010的调试环境
1. 生成vs2010能打开的sln格式文件
 ```shell
  waf.bat configure
  waf.bat msvs2010
 ```
2. 使用vs2010打开sln文件，解决方案下会存在多个项目，需要重点需要关注两个项目：
 - peach：整个框架的入口，调用peach.core.dll中的相关功能模块
 - peach.core：实现peach核心功能。以dll的形式编译输出，因此不能直接调试该项目，需要通过peach项目对peach.core.dll的调用，来进行调试跟踪

3. 在解决方案中设置启动项目顺序为“Single startup project”，并将最先启动项目名称设置为——peach
  ![img](https://raw.githubusercontent.com/carterMgj/blog_img/master/2018-02-02-peach-source/1.png)

4. 设置peach项目的命令行参数1为pit文件路径
  ![img](https://raw.githubusercontent.com/carterMgj/blog_img/master/2018-02-02-peach-source/2.png)

5. 完成以上设置，就可以下断点进行调试peach源码了，列一下vs2010调试的一些快捷键
 - F9：  下断点
 - F10：单步步过，执行一条指令
 - F11：单步步入，执行一条指令
 - F5：  执行到下一个断点


##二. 处理流程分析
1. **peach -- program.cs**，此处是整个peach框架执行的起始位置，在此处通过调用，开始执行peach.core工程中的核心代码
```c#
40  public class Program 
	{
		static int Main(string[] args)
		{
			Peach.Core.AssertWriter.Register();
			return new 	Peach.Core.Runtime.Program(args).exitCode;
		}
	}
```

2. 进入到**peach.core —— Runtime —— program.cs**，该模块会进行一些初始化操作，包括屏幕输出开始信息，检查用户提供的命令行参数是否合法，用户是否提供了额外的参数设置（包括agent、monitor、test，如果没有提供则从pit文件中进行读取）。
  接下来会做两件事：一是初始化Engine(Engine是peach框架的一个引擎，控制着整个fuzz过程) ；二是对pit文件进行解析
```c#
284	Engine e = new Engine(GetUIWatcher());
	dom = GetParser(e).asParser(parserArgs, extra[0]);
	config.pitFile = extra[0];
```
对pit文件的解析结果保存在一个叫dom的变量中，其中包括了pit文件中的dataModels、stateModels和tests的解析，dom变量在后续的过程将会被多次使用
![img3](https://raw.githubusercontent.com/carterMgj/blog_img/master/2018-02-02-peach-source/3.png)
相关初始化工作完成后，通过以下语句进入到Engine模块
```c#
298	e.startFuzzing(dom, config);
```

3. 进入到**peach.core —— Runtime —— Engine.cs**，此模块做的工作如下：
 - 检查dom，config，test等变量是否初始化完毕;
 - 初始化变异策略；
 - 初始化context类变量：context顾名思义是''上下文''的意思，该类变量中记录了与此次fuzz相关的各种信息，包括pit的解析信息、迭代信息、崩溃信息、控制信息等，是整个fuzz过程的“神经中枢”
     ![img4](https://raw.githubusercontent.com/carterMgj/blog_img/master/2018-02-02-peach-source/4.png)
     将context作为参数，传入OnTestStarting函数，开始整个fuzz过程
```c#
341	OnTestStarting(context);
```
之前说了，Engine.cs控制着整个fuzz过程的进行，那么肯定是要实现循环和迭代的。在此通过一个大的while循环，在while循环中实现迭代
```c#
358	while ((firstRun || iterationCount <= iterationStop) && context.continueFuzzing)
	{
		...
413		test.stateModel.Run(context);
	}	
```

4. 上图中test.stateModel.Run函数会继续调用StateModel模块，遂进入到**peach.core —— Dom —— StateModel.cs**。StateModel模块中会确定当前的State；通过updateToOriginalDataModel函数，使用datamodel对种子文件进行解析，从而完成对datamodel的初始化
```c#
171	foreach (State state in states)
	{
		state.UpdateToOriginalDataModel();
	}
```
updateToOriginalDataModel函数调用**peach.core —— Cracker —— DataCracker.cs**，最终通过调用**peach.core —— Data.cs** 中的函数完成解析功能
```c#
64	try
	{
		DataCracker cracker = new DataCracker();
		cracker.CrackData(model, new BitStream(File.OpenRead(FileName)));
	}
```
此时函数参数正是我们的datamodel和种子文件
  ![img10](https://raw.githubusercontent.com/carterMgj/blog_img/master/2018-02-02-peach-source/10.png)

完成对种子文件解析后，datamodel初始化便已经完成；接下来通过currentState.Run函数继续调用State模块对当前的State进行操作

```c#
182	try
	{
		currentState.Run(context);
		break;
	}
```

5. 进入到**peach.core —— Dom —— State.cs**。State的构造函数会解析得到State中的action列表，保存在actions变量中
```c#
60	public State()
	{
		actions = new NamedCollection<Action>();
	}
```
  ![img5](https://raw.githubusercontent.com/carterMgj/blog_img/master/2018-02-02-peach-source/5.png)
接着通过for循环，分别对于每一个action进行相应的操作。由上图所示，第二个action为output，即输出变异后的测试文件，其中会包含变异操作，所以重点关注第二次循环
```c#
201	foreach (Action action in actions)
		action.Run(context);
```

6. 进入到**peach.core —— Dom —— Action.cs**，以下代码就是变异生成新的种子文件的全过程
```c#
345	OnStarting();                                               //对种子文件按照pit的规定进行变异
	logger.Debug("ActionType.{0}", GetType().Name.ToString());
	RunScript(onStart);
	
	foreach (var item in outputData)                             // 输出新的变异内容，并保存为文件
		parent.parent.SaveData(item.outputName, item.dataModel.Value);
```

7. 接下来我们继续跟踪如何对种子文件进行变异的。当执行上图Onstarting函数时，通过c#中的**“委托机制”**，实际上是调用了多个模块中的函数。了解这一机制在peach源码阅读过程中非常重要，在此以Onstarting函数为例进行简要说明
  Action.cs :    Onstarting函数会调用starting事件
```c#
203	protected virtual void OnStarting()
	{
		if (Starting != null)				
		Starting(this);
	}
```

Watch.cs:  给Starting这一委托事件绑定了处理函数——Action_Starting
```c#
49	public void Initialize(Engine engine, RunContext context)
	{
	 	...	
70		Core.Dom.Action.Starting += new 	 
		ActionStartingEventHandler(Action_Starting);
	 	...
	}

```
因此当执行Starting时，就会执行peach.core —— Dom —— file.cs和peach.core —— MutationStrategy —— RandomStrategy.cs模块中的Action_Starting函数

8. 首先会进入**peach.core —— Dom —— file.cs**，在Action_Starting函数中会初始化变量rec
```c#
320	protected override void Action_Starting(Core.Dom.Action action)
	{
		var rec = new Fault.Action()
        {
			name = action.name,
			type = action.type,
            models = new List<Fault.Model>()
		};

		foreach (var data in action.allData)
		{
        	rec.models.Add(new Fault.Model()
			{
				name = data.dataModel.name,
				parameter = data.name ?? "",
                dataSet = data.selectedData != null ? data.selectedData.name : "",
				mutations = new List<Fault.Mutation>(),
			});
		}
```
rec记录了此次变异依据的种子文件和参照的datamodel
  ![img6](https://raw.githubusercontent.com/carterMgj/blog_img/master/2018-02-02-peach-source/6.png)

9. 随后进入到**peach.core —— MutationStrategy —— RandomStrategy.cs**，同样是调用名为Action_Starting的函数，其中存在一个if判断，如果_context.controlIteration变量为false则开始调用MutateDateaModel函数进行变异
```c#
247	else if (!_context.controlIteration)
	{
		MutateDataModel(action);
	}
```
此时参数action变量如下图所示
  ![img7](https://raw.githubusercontent.com/carterMgj/blog_img/master/2018-02-02-peach-source/7.png)
执行MutateDateModel函数，会调用ApplyMutation函数。其中outputData是一个迭代类型，通过for循环从中取数据进行ApplyMutation
```c#
407	foreach (var item in action.outputData)
	{
		ApplyMutation(item);
	}
```
此时item变量如下图所示：
  ![img8](https://raw.githubusercontent.com/carterMgj/blog_img/master/2018-02-02-peach-source/8.png)
ApplyMutation函数会继续调用OnDataMutating函数
```c#
387	if (elem != null && elem.MutatedValue == null)
	{
		Mutator mutator = Random.Choice(item.Mutators);
		OnDataMutating(data, elem, mutator);
		logger.Debug("Action_Starting: Fuzzing: " + item.ElementName);
		logger.Debug("Action_Starting: Mutator: " + mutator.name);
		mutator.randomMutation(elem);
	}
```
OnDataMutating函数有3个参数：data，elem和mutator，指定了使用哪一个变异器对原始文件的哪一个模块进行变异
  ![img9](https://raw.githubusercontent.com/carterMgj/blog_img/master/2018-02-02-peach-source/9.png)

10. OnDataMutating会通过委托机制，首先进入 **peach.core —— Dom —— file.cs**对第8步中提到的变量rec进行数据读取等操作
```c#
359	protected override void MutationStrategy_DataMutating(ActionData data, DataElement element, Mutator mutator)
	{
	var rec = states.Last().actions.Last();
	var tgtName = data.dataModel.name;
	var tgtParam = data.name ?? "";
	var tgtDataSet = data.selectedData != null ? data.selectedData.name : "";
	var model = rec.models.Where(m => m.name == tgtName && m.parameter == 
	tgtParam && m.dataSet == tgtDataSet).FirstOrDefault();
	System.Diagnostics.Debug.Assert(model != null);

	model.mutations.Add(new Fault.Mutation() { element = 
	element.fullName, mutator = mutator.name });
	}
```
然后会根据mutator的选取，调用相应的变异模块。在此是进入到 **peach.core —— Mutators —— ArrayReverseOrderMutator.cs**
```c#
83	public override void randomMutation(DataElement obj)
	{
		performMutation(obj);
		obj.mutationFlags = MutateOverride.Default;
	}
```
obj的内容非常丰富，如下图所示
  ![img11](https://raw.githubusercontent.com/carterMgj/blog_img/master/2018-02-02-peach-source/11.png)

## 3. 后记
 - 此时重点阅读的源码只是peach源码中初始化和变异的部分，对于agent、monitor、通信交互等模块均没有涉及
 - 原本计划通过理解源码，将peach变异的模块分解出来方便日后其他框架的移植，但由于参数的数据结构太复杂，最终也没有能够成功，这一部分作为以后的工作来持续跟进
 - 以上对于peach源码的理解还远远不够，由于能力有限，有说明有误的地方还请多多指正；也欢迎各位对peach源码有一定理解的朋友一起讨论，共同进步