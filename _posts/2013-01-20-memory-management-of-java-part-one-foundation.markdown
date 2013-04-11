---
author: shupeng.lisp
comments: true
date: 2013-01-20 15:25:26
layout: post
slug: memory_management_of_java_part_one_foundation
title: Java内存管理知识(基础篇)
wordpress_id: 145
categories: [技术]
---

第一部分 基础知识






  * 1.1 内存模型


  * 1.2 垃圾回收算法


  * 1.3  垃圾回收时机


  * 1.4 OOM时机


  * 1.5 Heap dump文件


  * 1.6  Shallow size与retained size


    <!--break-->

第二部分 内存分析






  * 2.1  内存泄露


  * 2.2  内存泄露现象


  * 2.3  内存分析方法


  * 2.4  内存分析工具


  * 2.5  问题答疑




第三部分 内存调优






  * 3.1  调优目标





## 第一部分 基础知识





本文很多都是针对Oracle 的HotSpot的，内存和垃圾回收的更一般知识请参考JVM规范或者[Java虚拟机之垃圾收集](http://shupeng.org/2013/01/20/memory-management-of-java-part-one-foundation/)获取。



### 1.1  内存模型






JVM内存分代处理，不同的代生命周期不同。




  * 年轻代(New, young generation)：用来存放JVM刚分配的Java对象。


  * 年老代(Tenured)：年轻代中经过垃圾回收没有回收掉的对象被copy到年老代。


  * 永久代(perm)：存放Class、Method元信息，其大小与项目的规模、类、方法的数量有关。一般设置为128M就足够，设置原则是预留30%的空间。


其中年轻代和年老代属于堆内存，堆内存会从JVM启动参数(-Xmx:3G)指定的内存中分配，perm不属于堆内存，由虚拟机直接分配，可以通过-XX:PermSize，-XX:MaxPermSize等参数调整其大小。

其中，年轻代又分为几个部分：


  * Eden：存放JVM刚分配的对象。


  * Survivor1


  * Survivor2：两个Survivor一样大，当Eden中的对象经过垃圾回收没有被回收掉时，会在两个Survivor之间来回copy；当满足某个条件，比如copy次数，就会被copy到Tenured。显然，Survivor只是增加了对象在年轻代中的逗留时间，增加了被垃圾回收的可能性。





### 1.2  垃圾回收算法





JVM会根据机器的硬件配置对每个内存代选择适合的回收算法。JVM有三种垃圾回收器，都基于标记-清除(复制)算法：分别是




  * throughput collector，用来做并行young generation回收，由参数-XX:+UseParallelGC启动；用多线程进行垃圾回收，回收期间会暂停程序的执行。


  * concurrent low pause collector，用来做tenured generation并发回收，由参数-XX:+UseConcMarkSweepGC启动；也是多线程回收，但期间不停止应用执行。


  * incremental low pause collector，可以认为是默认的垃圾回收器。






### 1.3  垃圾回收时机





绝大多数的对象都在年轻代被分配，也在年轻代被收回，当年轻代的空间被填满，GC会进行次回收(minor collection)，这次回收不涉及到heap中的其他代，次回收根据弱年代假设(weak generational hypothesis)来假设年轻代中大量的对象都是垃圾需要回收，次回收的过程会非常快。年轻代中未被回收的对象被转移到年老代，然而年老代也会被填满，最终触发主回收(major collection)，这次回收针对整个heap，由于涉及到大量对象，所以比次回收慢得多。

Heap中各代空间是如何划分的？通过JVM的-Xmx=n参数可指定最大heap空间，而-Xms=n则是指定最小heap空间。在JVM初始化的时候，如果最小heap空间小于最大heap空间的话，JVM会把未用到的空间标注为Virtual。除了这两个参数还有-XX:MinHeapFreeRatio=n和 -XX:MaxHeapFreeRatio=n来分别控制最大、最小的剩余空间与活动对象之比例。

由于年老代的主回收较慢，所以年老代空间小于年轻代的话，会造成频繁的主回收，影响效率。Server JVM默认的年轻代和年老代空间比例为1:2，也就是说年轻代的eden和survivor空间之和是整个heap（当然不包括perm gen）的三分之一，该比例可以通过-XX:NewRatio=n参数来控制，而Client JVM默认的-XX:NewRatio是8。至于调整年轻代空间大小的NewSize=n和MaxNewSize=n参数就不讲了。
 
年轻代中幸存的对象被转移到年老代，但不幸的是concurrent collector线程在这里进行主回收，而在回收任务结束前空间被耗尽了，这时将会发生Full Collections(Full GC)，整个应用程序都会停止下来直到回收完成。Full GC是高负载生产环境的噩梦……

perm gen，它是JVM用来存储无法在Java语言级描述的对象，这些对象分别是类和方法数据（与class loader有关）以及interned strings(字符串驻留)。一般32位OS下perm gen默认64m，可通过参数-XX:MaxPermSize=n指定。
归纳出来，垃圾回收的时机就是：




  * 当年轻代内存满时，会引发一次普通GC，该GC仅回收年轻代。需要强调是指Edne代满，Survivor满不会引发GC；


  * 当年老代满时会引发Full GC，Full GC将会同时回收年轻代、年老代；


  * 当永久带满时也会引起Full GC，会导致Class、Method元信息的卸载。





### 1.4  OOM(OutOfMemoryError)的时机





不是内存被耗空的时候才抛出，具体时机：




  * JVM98%的时间都花费在内存回收；


  * 每次回收的内存少于2%。


满足这两个条件将触发OutOfMemoryError。这将会留给系统一个微小的间隙以做一些Down之前的操作，比如手动打印Heap Dump。
 


### 1.5  Heap dump文件





heap dump是特定时间点，java进程的内存快照。有不同的格式来存储这些数据，总的来说包含了快照被触发时java对象和类在heap中的情况。由于快照只是一瞬间的事情，所以heap dump中无法包含一个对象在何时、何地（哪个方法中）被分配这样的信息。
 
在不同平台和不同java版本有不同的方式获取heap dump。

A Java heap dump is an image of the complete Java object graph at a certain point in time. It includes all objects, Fields, Primitive types and object references.
 


### 1.6  Shallow size与retained size





Shallow size就是对象本身占用内存的大小，不包含对其他对象的引用，也就是对象头加成员变量（不是成员变量的值）的总和。在32位系统上，对象头占用8字节，int占用4字节，不管成员变量（对象或数组）是否引用了其他对象（实例）或者赋值为null它始终占用4字节。故此，对于String对象实例来说，它有三个int成员（3*4=12字节）、一个char[]成员（1*4=4字节）以及一个对象头（8字节），总共3*4 +1*4+8=24字节。

Retained size是该对象自己的shallow size，加上从该对象能直接或间接访问到对象的shallow size之和。换句话说，retained size是该对象被GC之后所能回收到内存的总和。



## 第二部分 内存分析







### 2.1  内存泄露





通俗的来讲，就是出现了部分内存不在管理范围之类，永远回收不掉了，只会造成内存使用率越来越高，最后导致内存耗尽，引起OOM。

内存为什么会耗尽其余与什么对象会被垃圾回收所面对的问题是相同的：

GC发现通过任何reference chain(引用链)无法访问某个对象的时候，该对象即被回收。名词GC Roots正是分析这一过程的起点，例如JVM自己确保了对象的可到达性(那么JVM就是GC Roots)，所以GC Roots就是这样在内存中保持对象可到达性的，一旦不可到达，即被回收。通常GC Roots是一个在current thread(当前线程)的call stack(调用栈)上的对象（例如方法参数和局部变量），或者是线程自身或者是system classloader(系统类加载器)加载的类以及native code(本地代码)保留的活动对象。所以GC Roots是分析对象为何还存活于内存中的利器。

对象引用又分为以下几个部分，从最强到最弱，不同的引用（可到达性）级别反映了对象的生命周期。




  * Strong Ref(强引用)：通常我们编写的代码都是Strong Ref，于此对应的是强可达性，只有去掉强可达，对象才被回收。


  * Soft Ref(软引用)：对应软可达性，只要有足够的内存，就一直保持对象，直到发现内存吃紧且没有Strong Ref时才回收对象。一般可用来实现缓存，通过java.lang.ref.SoftReference类实现。


  * Weak Ref(弱引用)：比Soft Ref更弱，当发现不存在Strong Ref时，立刻回收对象而不必等到内存吃紧的时候。通过java.lang.ref.WeakReference和java.util.WeakHashMap类实现。


  * Phantom Ref(影子引用)：根本不会在内存中保持任何对象，你只能使用Phantom Ref本身。一般用于在进入finalize()方法后进行特殊的清理过程，通过 java.lang.ref.PhantomReference实现。


 
有了上面的种种很容易就能把heap和perm gen撑破了，就是利用Strong Ref，存储大量数据，直到heap撑破；利用interned strings（或者class loader加载大量的类）把perm gen撑破。





### 2.2  内存泄露现象





可能发生内存泄露的一些典型现象如下：




  * 每次GC的时间越来越长，，Full GC的时间也延长；


  * Full GC的次数越来越多，最频繁时隔不到1min就进行一次Full GC；


  * 年老大的内存越来越大并且每次Full GC后年老代没有内存被释放；



之后系统会无法响应新的请求，逐渐达到OutOfMemoryError的临界值。




### 2.3  分析方法





实时分析：
实时profiling/monitoring之类的工具，用一种非常实时的方式来分析哪里存在内存泄漏。这样的工具本身就要消耗性能，且在某些条件下还发现不了泄漏。在吞吐量很高的时候profiler工具自己可能也无法响应.

离线分析：
通过将内存快照记录下来，线下进行分析这些离线数据。也就是分析heap dump文件。



### 2.4  分析工具









  * jmap

JDK自带的一个工具，是JVM Heap导出的必备工具。
jmap -dump:format=b,file=xxx.bin pid   pid是java程序pid

此命令会将虚拟机heap镜像导成文件。不过jmap也有直接分析功能：jmap -histo pid。优点是可以直接查看对象的大小和类型，缺点是无法查看详细的对象引用信息。


  * jhat

JDK自带的dump文件分析工具，会启动一个webserver，可以直接浏览对象大小、类型及对象引用信息。缺点是对大的dump文件力不从心，分析时间长而且web界面会因为对象太多而无响应或者OOM。
jhat -J-mx512m -port <端口号:默认为7000> xxx.bin -mx512m表示所用最大内存512M


  * mat

使用参考：http://www.ibm.com/developerworks/cn/opensource/os-cn-ecl-ma/index.html
后续会分享一篇这样的实例。




### 2.5  问题答疑





1：为什么崩溃前垃圾回收的时间越来越长？
A：垃圾回收分为两部分：内存标记、清除（复制）。标记部分只要内存大小固定，时间是不变的；变化的是第二阶段，每次都有回收不掉的内存，复制量增加，导致时间延长。因此可以作为判断依据。

2：为什么Full GC的次数越来越多？
A：内存的积累逐渐耗尽了年老代的内存，导致没有新对象分配所需的足够空间，从而导致频繁的Full GC。

3：为什么年老大占用的内存越来越大？
A：年轻代的无法回收，copy到年老代的越来越多。




## 第三部分 性能调优





性能调优和具体场景密切相关，本部分只提供部分常用做法，以及根据前面知识的理论分析。部分不调优也全部集中在JVM参数上。


### 3.1  期望目标





在JVM启动参数中，可以设置跟内存、垃圾回收相关的一些参数设置，默认情况不做任何设置JVM会工作的很好，但对一些配置很好的Server和具体的应用必须仔细调优才能获得最佳性能。通过设置我们希望达到一些目标：




  * GC的时间足够的小


  * GC的次数足够的少


  * 发生Full GC的周期足够的长



参考文章：

