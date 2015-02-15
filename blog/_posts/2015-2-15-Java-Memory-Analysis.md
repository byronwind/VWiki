---
layout: post
title: Java 内存分析
summary: 本文概要介绍了JVM中的内存模型，堆内存的分析方式和对象内存占用的衡量方式，目的在于对JVM中的内存分析基础方法做一个入门总结。
date: 2015-02-15
tags: "java,jvm,memory analysis,mat"
---

{{ page.title }}
================

# 导言 

由于 Java 虚拟机的自动管理内存机制，Java 程序员不需要像C/C++一样去关注内存的分配和释放，也不容易出现内存泄露和内存溢出的问题。但是如果一旦出现此类问题，如果我们对虚拟机的内存管理机制不了解，要解决这类问题还是比较困难的。

在 Android 平台上（Dlavik模式），Java 代码最终编译为 DalvikVM 的字节码（dex），DalvikVM 是一个类JVM的虚拟机，并针对移动操作系统的特性进行了优化。在进行内存分析时，通常还会转换成 Java HeapDump 格式的文件，因此，本文的一些背景知识还是基于标准JVM。

# JVM 运行时数据区域

Java 虚拟机在执行 Java 程序过程中会把它所管理的内存划分为几个不同的数据区域。这些区域都有各自的用途，以及创建和销毁的时间，有的区域随着虚拟机进程的启动而存在，有些区域则是依赖用户线程的启动和结束而建立和销毁。

《Java 虚拟机规范》中，Java 虚拟机所管理的内存包含的运行时数据区域如图：

![Java 虚拟机运行时数据区](./img/jvm_architecture.png)

这几个区域如下：

- 方法区（Method Area）
- Java 虚拟机栈（Java Stack）
- 本地方法栈（Natvie Method Stack）
- 堆（Heap）
- 程序计数器（Program Counter Register）

关于这几个区域的详细解释可以参见JVM的相关文档，或者[《深入理解 Java 虚拟机》]一书。我们这个主题关注的部分是 **Heap** 这一个区域。

# Java VM Stack 和 Heap

我们经常会把 Java 内存分为堆内存（Heap）和栈内存（Stack），这种区分方式比较粗糙。我们平时所关注的也主要是这两个区域。栈指的是上图中的Java虚拟机栈。Java 虚拟机栈是线程私有的，其生命周期和线程相同。虚拟机栈描述的是Java方法执行的内存模型，每个方法被执行时都会创建一个栈帧（Stack Frame），用来存储局部变量表、操作数栈、动态链接、方法出口等信息，每一个方法被调用直至执行完成的过程，就对应着一个栈帧在虚拟机栈中从入栈到出栈的过程。

局部变量表存放了编译期可知的各种基本数据类型（boolean、byte、char、short、int、float、long、double）、对象引用（reference）和 returnAddress（指向一条字节码指令的地址）。

如果线程请求的栈深度大于虚拟机允许的深度，会抛出 StackOverflowException 异常；如果虚拟机栈可以动态扩展，扩展时无法申请到足够内存时会抛出 OutOfMemoryError 异常。

# Java 堆

对于大多数应用程序来说，Java 堆（Java Heap）是Java虚拟机管理的内存中最大的一块。Java 堆是所有线程共享的一块内存区域，几乎所有的对象实例都在这里分配内存。根据Java虚拟机规范的规定，Java堆可以处于物理上不连续的内存空间中，只要逻辑上是连续的即可，就像我们的磁盘空间一样。

## 对象访问

在 Java 语言中，即便是最简单的对象访问，也会涉及Java栈、Java堆、方法区这三个最重要的内存区域。例如下面这行代码：

```
 Object obj = new Object();
```

假如这段代码出现在方法体中，`Object obj` 这部分的语意会反映到Java虚拟机栈的本地变量表中，做为一个 reference 类型数据出现。`new Object()`这部分语意会反映在Java堆中，形成一块存储了 `Object` 类型所有实例数据值（Instance Data，对象中各个实例字段的数据）的结构化内存。不同虚拟机的对象访问方式有所不同（虚拟机规范中没有具体规定），主流的访问方式有两种：使用句柄和直接指针。具体详见[《深入理解 Java 虚拟机》] 对象访问一节内容。

## 堆转储快照

分析内存问题时，可以进程的内存信息dump到文件中分析（JDK中的jmap工具，Android中ddms）。分析工具可以用[Eclipse Memory Analyzer]。MAT中可以对内存泄露、对象内存占用情况进行多方面的分析。

### 内存分析的几个概念

- Shallow Heap 
	对象自身占用的内存大小（包含基本数据类型），不包括它引用对象的大小。

- Retained Heap 
	Shallow Heap + 所有直接或者间接引用对象占用的内存。（即该对象被GC回收后，可以被回收的内存）。

	为了计算Retained Heap，可以将堆中的对象关系理解为一张图，堆中的对象构成图中的节点，图的边表示对象的引用关系。

	![Retained Objects](./img/retained_objects.png)

	图中，假设 obj1 被回收，GC 可以回收 obj1、obj2、obj4 三个对象（obj3 被 GC Roots 引用，不会被回收），因此 obj1 的 Retained Heap大小即为 obj1、obj2 和 obj4 的 Shallow Heap 大小。

- GC Root 
	被堆外对象引用的对象。

- Unreachable Objects
	不可达对象，可以被GC回收的。

- [Dominator Tree]
	以支配树方式描述的对象引用关系。
	
	在对象关系图中，如果从某一点做为根节点出发，通过引用关系到达对象 Y 的所有路径中，都必须经过 X ，则对象X支配对象 Y。

	在支配树中，每个节点存储了该对象的RetainedHeap大小。下图是一个DominatorTree的示例：

	![Dominator Tree](./img/dominator.png) 

	- A、B、C节点被一个虚拟的root节点支配。
	- 支配关系是可传递的，C 支配 E，E 支配 G ，所以有 C 支配 G。


### Java 对象的内存占用分析

有了上面几个概念，如果我们想计算某些 Java 对象占用的堆内存的大小，就可以用 Retained Heap 的大小表示。计算Retained Heap的方法，可以先构造Dominator Tree，该对象节点的Retained Heap就是需要的结果。

这些计算，在[Eclipse Memory Analyzer]工具中都提供了，并且有很详细的报告可以输出。关于[Eclipse Memory Analyzer]工具的使用，可以参考官方文档。

如果我们需要自己用程序分析HeapDump文件来计算一些数据，有什么方法呢？我们下面讨论一下。

## 用代码方式分析 Java HeapDump

[Eclipse Memory Analyzer]工具提供了很多分析方式，这个工具是开源的，我们可以用他的代码来实现我们自己的分析方式。[Eclipse Memory Analyzer]是运行在Eclipse上的，代码结构基于Eclipse插件的规范。但是除依赖GUI之外的代码部分（HeapDump文件解析，索引构建、Dominator Tree构造等）比较独立，我们可以引用到自己的项目中。

### Dominator Tree 构造方式

[Eclipse Memory Analyzer]中DominatorTree的实现在 `org.eclipse.mat.parser.internal.DominatorTree` 类中。算法基于[Lengauer-Tarjan 算法]。

MAT 中通过一个 Snapshot 对象构造一个描述 DominatorTree 结果的对象，并可以按照 Class 、ClassLoader 和 Package级别分组，如果我们想计算某个 Class 的对象的 DominatorTree 情况，可以按 Class 分组，如果想看某个 Package 下的 RetainedHeap，可以按照 Package 分组。

示例代码如下：

```
DominatorQuery.Tree tree = DominatorTreeCal.Factory.create(snapshot, new int[] { -1 },new VoidProgressListener());
int[] roots = tree.getRoots();
IResultTree resultTree = DominatorTreeCal.Factory.groupByPackage(snapshot, roots, new VoidProgressListener());
```

# 总结

本文概要介绍了JVM中的内存模型，堆内存的分析方式和对象内存占用的衡量方式，目的在于对JVM中的内存分析基础方法做一个入门总结，如果想进一步研究其中的原理和方法，可以进一步参考相关的书籍和文档，也可以深入研究[MAT的源码](http://wiki.eclipse.org/index.php?title=MemoryAnalyzer/Contributor_Reference)。

[《深入理解 Java 虚拟机》]: http://book.douban.com/subject/6522893
[Eclipse Memory Analyzer]: http://www.eclipse.org/mat
[Dominator Tree]:http://help.eclipse.org/indigo/topic/org.eclipse.mat.ui.help/concepts/dominatortree.html
[Lengauer-Tarjan 算法]:http://www.cl.cam.ac.uk/~mr10/lengtarj.pdf