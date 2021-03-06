---
title: JVM内存管理(三)
date: 2020-01-22 08:57:0
updated:
tags: JVM
categories: 技术
thumbnail: chateau.jpg
---

## 前言

这是该系列的最后一篇了。本打算和前两篇一样周更，无奈由于各种事由拖到了寒假。博客这件事，还须坚持，坚持，再坚持！

本文主要结合HotSpot这款流行的JVM实现，主要从分代收集、根枚举和垃圾收集器几个方面做简要介绍。

## 分代收集

在HotSpot中，将堆内存分为新生代和老年代。

{% asset_img heap.png "HotSpot中的堆划分" %}

其中新生代区域被进一步划分为一块Eden区和两块Survivor区，它们之间的比例为8：1：1。所以我们看，这种设计的空间利用率还是比较高的，只有10%的空间空闲。GC时，使用复制算法把Eden和Servivor中还存活的对象复制到另一个Survivor区域中，最后对清理Eden和刚才Survivor中的空间。对于老年代，使用的是标记-整理算法。

在HotSpot中，对象优先在Eden分配，当Eden区没有足够的空间进行分配时，将触发一次minor GC，也就是对新生代进行垃圾收集。当然，并不是所有的对象都一定分配在Eden，对于大对象，比如很长的数组或字符串，它们会直接被分配到老年代。这是因为新生代采用复制算法，进行大对象的复制代价很大。具体多大算大，这个阈值可以通过参数进行设置。

新生代每经过一次minor GC并存活下来，年龄便会加一，当它达到一定年龄之后，默认为15岁，便可以晋升为老年代了。当然并不是一定得达到年龄要求后才能进入老年代，虚拟机对某些场景进行了一些优化。比如：在survivor中处于某个年龄的所有对象的总大小超过了survivor空间的一半，那么大于或等于这个年龄的对象可以直接进入老年代。

与minor GC相对的是full GC, 比如当老年代空间告急时就会触发一次full GC，即在整个堆上进行一次垃圾收集。

## 根枚举

前面我们说到，任何垃圾收集算法在第一阶段要做的都是对对象进行标记。进行可达性分析的时候，就要从GCRoots开始进行root tracing。

我们知道GCRoots的定义了：被全局性引用（常量或静态属性）或执行上下文（如栈帧中本地变量）所引用的对象。但如何才能找到这些活跃引用呢，即如何完成根枚举（root enumeration）呢？要知道搜寻活跃引用的范围并不小，而且在进行根节点枚举时有一个**Stop The World**过程，也就是再GC进行枚举的时候，其他用户线程都要停下来。我们想象一下，垃圾收集其实就像打扫房间一样，在你妈妈打扫房间的时候，你还会继续往地上扔垃圾吗？你是不是还得把脚抬一抬？Stop The World听起来很酷，但我们都不喜欢。那么如何能够高效快速地完成根节点枚举呢？

以栈上引用的查找为例，主要有两种不同的实现方式。

首先引入保守式GC的概念。在这种实现方式中，JVM不会记录栈帧内的数据类型，因此也无法区分内存里某个位置上的数据到底应该解读为引用类型还是其他类型。在进行GC的时候，JVM开始从一些已知位置（例如说JVM栈）开始扫描内存，扫描的时候每看到一个数字就看看它“像不像是一个指向GC堆中的指针”。这里会涉及上下边界检查（GC堆的上下界是已知的）、对齐检查（通常分配空间的时候会有对齐要求，假如说是4字节对齐，那么不能被4整除的数字就肯定不是指针），之类的。然后这样递归地扫描出去。

保守式GC的缺点很明显：由于它找到的都是疑似引用，那么真正的引用固然不会错过，但一些假引用也被纳入GCRoots，所以会造成垃圾收集的不充分。

与保守式相对的是准确式GC，这也是主流JVM，包括HotSpot所采用的方式。采用准确式GC就是说JVM准确地知道栈帧内各变量中哪些是引用，这样在查找GCRoots时就不用遍历栈上的每个变量了。如何做到这一点呢？以空间换时间。主流的做法是使用额外的数据结构从外部记录下类型信息。在HotSpot中，这个数据结构叫做OopMap。

Oop指的是Ordinary Object Pointer：普通对象指针。OopMap的作用就是记录栈上那些地方是引用。实现这样的功能，需要解释器和JIT编译器的共同支持：它们知道变量确切的地址，由它们生成足够的元数据再GC的时候使用。我们知道程序运行过程中，引用关系一直在变化，那我们要一直跟着生成OopMap来追踪引用的位置信息吗？即：何时生成OopMap呢？

首先，在类加载过程中，对象的类型信息里便会生成自己的OopMap，记录了在该类型的对象内什么偏移量上是什么类型的数据。所以从对象开始向外的扫描可以是准确的。

而为了记录其他位置，比如栈上的引用，HotSpot会在某些指令流的特定位置记录下OopMap，指示执行到该方法的某条指令的时候，栈上哪些位置是引用。这样GC在扫描栈的时候就会查询这些OopMap就知道哪里是引用了。
这些特定的位置主要在：
1. 循环的末尾
2. 方法临返回前 / 调用方法的call指令后
3. 可能抛异常的位置

这种位置被称为“安全点”（safepoint）。

之所以要选择一些特定的位置来记录OopMap，是因为如果对每条指令（的位置）都记录OopMap的话，这些记录就会比较大，那么空间开销会显得不值得。选用一些比较关键的点来记录就能有效的缩小需要记录的数据量，但仍然能达到区分引用的目的。因此，HotSpot中GC不是在任意位置都可以进入，而只能在safepoint处进入。

那么在GC发生时如何让所有的线程（不包括执行JNI调用的线程）都能到达安全点呢？有两种方法：一种是抢先式中断，另一种是主动式中断。抢先式中断是指GC时，首先把所有的线程都中断，如果被中断的线程不在安全点上，则恢复该线程让它继续跑到安全点上。主动式中断是指GC时不中断线程，只设置一个中断标志，各个线程在执行时主动轮询这个标志，当标志为真时就主动中断挂起。何时轮询呢？一般是在安全点处以及创建对象分配内存时。现在一般都采用主动式中断方案。

正在运行的线程可以跑到安全点，那没有正在执行的线程怎么办呢？比如处于Blocked状态的线程，它是不可能自己主动到达安全点的。为此我们引入安全域（safe region）的概念。安全区域是指引用关系不会发生变化的代码区域。比如线程被挂起后就不会改变引用关系。在线程执行到安全域中的代码之后，首先会标识自己已经进入了安全域。这样gc时便不用关心那些safe region状态的线程了；而当线程要离开安全域时，首先检查根枚举是否完成，如果是则可以安全离开，否则等待离开信号。

以上我们所说的都是Java线程，对于Java线程中的JNI方法，它们既不是由JVM里的解释器执行的，也不是由JVM的JIT编译器生成的，所以会缺少OopMap信息。那么GC碰到这样的栈帧该如何维持准确性呢？HotSpot的解决方法是：所有经过JNI调用边界（调用JNI方法传入的参数、从JNI方法传回的返回值）的引用都必须用“句柄”（handle）包装起来。JNI需要调用Java API的时候也必须自己用句柄包装指针。在这种实现中，JNI方法里写的“jobject”实际上不是直接指向对象的指针，而是先指向一个句柄，通过句柄才能间接访问到对象。这样在扫描到JNI方法的时候就不需要扫描它的栈帧了——只要扫描句柄表就可以得到所有从JNI方法能访问到的GC堆里的对象。但这也就意味着调用JNI方法会有句柄的包装/拆包装的开销，是导致JNI方法的调用比较慢的原因之一。

但这还不够。前面说到，GC分为MinorGC和FullGC。在进行FullGC的时候，这样好像没问题。但是MinorGC的时候就不一定了。比如说，枚举根节点时，根节点有可能在新生代中，也有可能在老年代中。这里由于我们只想收集新生代，所以没有必要对位于老年代的 GC Roots 做全面的可达性分析。但问题是，确实可能存在位于老年代的某个 GC Root，它引用了新生代的某个对象，这个对象你是不能清除的。
那怎么办呢？

HotSpot使用RememberedSet 用于处理这类问题：仍然是拿空间换时间的办法。事实上，对于位于不同年代对象之间的引用关系，虚拟机会在程序运行过程中给记录下来。对应上面所举的例子，“老年代对象引用新生代对象”这种关系，会在引用关系发生时，在新生代边上专门开辟一块空间记录下来，这就是 RememberedSet 。所以“新生代的 GC Roots ” + “ RememberedSet 存储的内容”，才是Minor GC时真正的 搜索根节点。后面说的G1收集器也用到了这种数据结构。

## 垃圾收集器

下面就来简单介绍一下HotSpot的几款具有代表性的垃圾收集器：Serial、CMS以及G1。

{% asset_img gcs.jpg "HotSpot支持的垃圾收集器"%}

Serial收集器是最基本、也是发展历史最悠久的收集器。它使用的是复制算法，用于新生代。是一个单线程的收集器。它在收集垃圾时，必须暂停其他所有线程，Stop The World，直到垃圾收集结束。以下是它的运行示意图。可以看到它是完全单线程工作的。

{% asset_img serial.jpg "Serial垃圾收集器"%}

CMS，Concurrent Mark Sweep，并发标记-清除，用于老年代收集。它以低停顿为目标，它的优势在于标记与清除阶段可以并发进行。主要阶段为：
1. 初始标记
2. 并发标记
3. 重新标记
4. 并发清除

其中，初始标记以及并发标记这两个阶段仍然需要stw。初始标记指是标记与gcroots直接关联的对象，时间很短。完全的Root tracing在并发标记阶段进行。由于在这个阶段由于用户程序也在运行因此可能会改变之前的标记，这时还需要重新标记来进行修正，这个阶段时间会比初始标记稍长，但远比并发标记短。最后是并发清除过程。

{% asset_img cms.jpg "CMS垃圾收集器"%}

从图中可以看出，整个GC过程中耗时最长的root tracing和清除过程都可以并发进行。因此CMS可以提供较低的gc停顿。

最后介绍一下G1，Garbage Fisrt，这是一款能够独当一面的收集器，它既可以用于新生代也可以用于老年代。

{% asset_img g1.jpg "G1垃圾收集器" %}

主要分为如下几个阶段：
1. 初始标记
2. 并发标记
3. 最终标记
4. 筛选回收

从流程上来，它和CMS很相似。G1采用了化整为零的思想，把一块大的内存划分成很多个域（ Region ），以域为单位进行gc。

{% asset_img regions.png "G1对堆划域管理" %}

可以看到，虽然还保留了新生代和老年代的概念，但它们不再是那样泾渭分明，物理隔离的了，都是region的集合。

G1能够提供可预测的停顿，因为它有计划地避免在Java堆中进行全域的垃圾收集。G1跟踪各个Region里垃圾堆的价值大小，即回收所获得的空间大小以及回收所需时间的经验值，并在后台维护一个优先列表，每次根据允许的收集时间优先回收价值最大的region，从而达到尽可能高的gc效率。只对一个region进行回收，听起来不错。但问题是，各个region的对象间难免存在引用关系。那么在进行标记时岂不是还要扫描整个堆吗？为了达到可以以 Region 为单位进行垃圾回收的目的，G1 收集器也使用了 RememberedSet 的数据结构，在各个 Region 上记录自己的对象被外面对象引用的情况。gc时，以gcroot连同remembered set为起点开始标记。

## 后记

JVM内存管理系列暂时告一段落。希望新的一年里能保持稳定的内容输出。2020，奥利给！