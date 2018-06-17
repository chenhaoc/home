## Berkeley的RISC-V开源项目为什么选用Chisel？

作者：郭雄飞

事实胜于雄辩，让我们先跳过过程看下结果吧。

- SiFive作为Berkeley孵化出的高科技公司，几乎所有的研发均使用chisel
- rocket-chip项目在Github上做为标志性的chisel项目，包含一个可定制型很强的CPU和总线等其他IP
- rocket-chip项目到2018年6月有接近6000次提交
- 2017年，SiFive在众筹网站CrowdSupply上发布了Hifive1，板子上搭载的芯片就是基于rocket-chip并添加其他第三方IP的**FE310**芯片，工艺为TSMC 180nm，这是一块Arduino兼容的开发板
- 2018年，SiFive又在CrowdSupply上发布了其4+1核心的可以运行Linux操作系统的HiFive Unleashed开发板，芯片代号**FEU540**，工艺为TSMC 28nm，这款SoC芯片依然是基于rocket-chip的。

所以在我看来，为什么用chisel而不用其他语言自然是因为人家觉得好用，Berkeley可以称得上是在诸多大学里非常接地气的大学了，他们似乎除了提出超前的概念之外，更有能力去让这些想法落地，去看看人家孵化出多少硅谷项目(Spark，Unix/BSD)就知道了。

所以，今天我们就在这里聊聊chisel吧（客观的

chisel没打算取代verilog或者systemverilog，而只是希望在这个基础之上做一个高层次的**构建**语言，所以chisel叫做Hardware **construction** language，要明白这和高层次综合（HLS）并无半点关系。（但是你也可以在chisel的基础上构建HLS，这是另一码事。）

聊到chisel，很多前端工程师就会想到Java和Scala，这是比较常见的误区，所以不如先聊聊Scala吧。

### Scala：天生适合做"新语言"的"高级玩具"

先想象下你有一盒含有MindStorm套件的乐高积木，用它来做了一辆遥控车。Scala就是这样的"高级玩具"。

Scala是由洛桑联邦理工学院的*Martin Odersky* **精心设计**的一门*多范式、函数式和面向对象*的编程语言。我们可以不用太关心前面的定语，我们只要记住，Scala很适合做*领域特定语言(DSL)*，这一定程度上是设计出来的，而非偶然。当然，”函数式“这个属性必须得提一下，因为函数式编程语言往往和硬件模块存在一些等价转换关系。

DSL意味着他可以像橡皮泥或者积木一样被组合或者塑造成为新的语言，一个经典的案例就是把**Scala捏成BASIC语言**。(Github Repo: [https://github.com/fogus/baysick](https://github.com/fogus/baysick))

![Baysick](/assets/images/articles/scala-baysick.png)

看到了么，因为Scala中几乎所有的元素都是对象，并且都能够被赋予新的功能，同时配合大量的语法糖，所以这个无聊的小哥用它来实现了一门新的语言，而且只用了300行。（Code: [https://git.io/vhMmO](https://github.com/fogus/baysick/blob/master/src/main/scala/fogus/baysick/Baysick.scala))

### Chisel要解决什么问题？

请允许我举一个非常不恰当的栗子，我们以设计一个CPU为例吧。

你本科熬了几年图书馆，挤破了头进入国内某微电子学院做了研究生，老师进来和你说我有个很好的想法，能够有效的改进指令效率或者多核性能或者功耗。

老师说你做个5级流水线CPU把，还要把cache、总线、外设之类也做了（没缓存搞什么多核？）。好吧，我承认你很聪明，不出几个月你把CPU写的差不多了，然后cache、总线、外设这些大头还远着呢。又过了几个月你天天啃《量化研究方法》，然后终于把cache实现了。然后你写了个GPIO，又挂了个SRAM，好吧，你终于实现了一个小的CPU了。为了降低难度，你用了学术界最爱的MIPS体系结构，用了最土的wishbone总线。然后你开始了撸软件了，因为用了MIPS，你的难度已然降低了很多，而且你不用考虑编译器的问题了，你又吭哧了好几个月，写了个巨土的bootloader，终于把程序加载了。尽管后面可能还要在FPGA上跑起来，要发顶作的同学还要去申请经费流个片，这估计又要好久好久。但到目前为止你终于可以开始评估下你的设计的好坏了。

你跑了一堆benchmark，得出了一些结果，然后你才开始把导师的idea应用到你的设计中。然后，然后，你就硕士毕业了，放心吧，你的这个摊子，你的学弟们会接锅的。

这里尽管很多东西不那么真实，但是不得不说大学教授的很多项目，都是好几届学生慢慢做才做下来的，而且做归做，评估归评估。做完了哪里不好还得继续改进，因为有了架构，离实现到最后变成芯片还远着呢。更何况，评估一个设计好坏这件事本身或许难度更大。

以上的故事暴露了一个问题，对于改进硬件架构这件事，反馈环实在太长了。

所以扯了半天，我其实就想说一句话，硬件设计太耗时，Verilog写的蛋疼，需求要是变一点，那些个接口就得跟着变。要是速度上不去了，我要是想换个架构，又要花好久。sv或许好一些，但你真的爱她吗？

### RISC-V和Chisel的小故事

所以当Berkeley的Krste教授和他的学生们决定要去做一些研究的时候（比如下一代数据中心的CPU架构、后摩尔定律时代的CPU架构或是AI加速处理器的时候），他们当然希望反馈环足够短啊！尽早评估、快速迭代对一个研究者来说太重要了啊。

差不多在同一时间，Berkeley的Joseph Whitworth教授也正好开始了Chisel的研究和开发工作，那他们自然就要决定联合起来做些有趣的事情。当他们决定做他们的第五代RISC CPU指令集的时候，他们需要设计一个真实的CPU来评估他们设计指令集过程中的每一个选择。所以他们从头用Chisel写了CPU，因为chisel面向对象的一些属性，他们能用很少的时间就把设计做好并且评估，chisel只要几十秒就可以生成verilog或者C++model，然后直接扔进仿真器里去跑benchmark。

就这样他们没用多少年就做了一个全新的开源指令集RISC-V，这个指令集有多好呢？我说你肯定不信，我引用一段最近David Ditzel采访里的话。Dave在Sun参与过SPARC ISA的设计，后面创立了全美达（Transmeta）曾经让Intel也胆战心惊。他最近成立了一家新的公司做RISC-V的高性能CPU。以下是采访内容：

> RISC-V wasn't even on the shopping list of alternatives, but the more Esperanto's engineers looked at it, the more they realized it was more than a toy or just a teaching tool. “We assumed that RISC-V would probably lose 30% to 40% in compiler efficiency [versus Arm or MIPS or SPARC] because it’s so simple,” says Ditzel. “But our compiler guys benchmarked it, and darned if it wasn't within 1%.”
> 
> “RISC-V最开始甚至不在我们可考虑范围之内，但是我们Esperanto的工程师越深入的了解它，就越发现RISC-V不仅仅是个玩具或者教学用的工具。我们还假定说RISC-V在编译器效率上相比Arm/MIPS/SPARC会损失30%到40%左右，因为它实在是太简单了。但我们的编译器工程师对他进行了评测，发现只损失了可恨的不到1%。”Ditzel如是说

基于这几个事实我得出的推论就是，当我能够更快的评估我的硬件时，我就能更快的改进它，也就是能比别人更早的靠近不断变化中的有效边界。

### Chisel的未来

当我们看待一件新事物的时候，千万不要用他现有的状态去预测它的未来。你一定得想想，Chisel未来会发展成什么样。

Chisel仍然是一门不断发展和进化中的项目，一个重要的节点就是chisel开始分离成为两个项目，*chisel*和*firrtl*。简单讲，如果吧chisel理解为把chisel代码转化为verilog的话，那么这个步骤被分为了两步：

1. 从chisel到firrtl（一种硬件描述中间表达）
2. 从firrtl到verilog

在笔者看来这个思路就是LLVM的思路，熟悉编译器的朋友或许知道，LLVM设计了一种**描述清楚**的中间表达`llir`，编译器前端可以将任何语言转化位llir，而所有的中间优化步骤会一级级的处理llir，优化过的内容仍然是llir表达的，最终的llir会被编译器后端(Backend)转化为到目标机器汇编代码。我们可以说几乎大部分编译器都是这样工作的，但是当llir足够开放并且易于被他使用的时候，就会有更多的人来加入这个阵营，发挥他们各自的专业优势来实现更好的编译器PASS。这也是为什么LLVM本质上是*编译器基础设施(compiler infrastructure)*而非仅仅是*编译器软件*。

所以请允许我用下面的图不专业的理解一下LLVM所做的事的话。

![LLVM Stages](/assets/images/articles/llvm-stages.svg)

那么让我再次发挥一下想象力，想像一下chisel+firrtl想要做的。

![Chisel Stages](/assets/images/articles/chisel3-stages.svg)

这里我不想解释太多，但我会问个问题：*如果RISC-V会抹平过去高昂的CPU授权费，那么昂贵的EDA工具这个山头谁来抹平呢？*

不要小看开源的力量！

### 请保持开放的心态

冷静的看待chisel的话，我认为这是就是未来或者是未来路上的一站，原因是它能提高生产力，我们也必须抱着同样包容的心态去看待其他类似的方法和途径。谁也看不清未来是怎样的，但是你要明白，当未来到来的时候，要想领先别人，你现在就得去做些什么对自己未来有利的事情。

所以，当你和我抱怨chisel是scala写的，我不会scala的时候，看我的大白眼！

![看我的大白眼](/assets/images/articles/call-dabaiyan.jpg)

----

以下是广告时间，2018年6月30日上海会有一场关于RISC-V的研讨会 **RISC-V Day 2018 Shanghai**，也是本年度大陆地区唯一官方举办的研讨会，欢迎来一起聊聊RISC-V或者Chisel。**参会者会有纪念T恤和神秘纪念品，绝对值回票价。**（早鸟票6月17日截止）。

- 会议议程见： [https://riscv.org/2018/06/risc-v-day-in-shanghai/](https://riscv.org/2018/06/risc-v-day-in-shanghai/)
- 注册网址：[https://tmt.knect365.com/risc-v-day-shanghai/](https://tmt.knect365.com/risc-v-day-shanghai/)
- 在校学生还可以申请参会和差旅资助：[https://cnrv.io/articles/risc-v-day-2018-shanghai-student-sponorship](https://cnrv.io/articles/risc-v-day-2018-shanghai-student-sponorship)。

![RISC-V Day 2018 Shanghai](/assets/images/bi-weekly-rpts/2018-06-08/riscv-day-shanghai.png)
