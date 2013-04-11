---
author: shupeng.lisp
comments: true
date: 2012-10-17 23:03:58
layout: post
slug: practice_in_objectc_object_message_method
title: Object-c实践之路：第二章 对象、消息与方法
wordpress_id: 103
categories:
- 心得
---

#### 摘要


本文主要就object-c中的类、对象、实例、消息和方法这些知识进行介绍。在理解上述概念的同时，介绍了对象的创建和释放，以及消息的发送等知识。希望在本文之后，能在一定程度上加深对objecy-c的了解。

1 基本概念

既然叫做object-c，那么它和传统的c语言的最大不同之处在哪里呢？可以说，object这个次就将主要区别阐释出来了，c语言是一门典型的面向过程的编程语言，而object-c就具有面向对象的特征了。

什么是对象呢？

<!--break-->

可以这样理解，在c语言中，我们要表现一个学生的信息，如姓名、年龄、家庭住址等，需要定义一个结构体struct，这个结构体具有多个数据成员，每个数据成员保存一部分信息，整个结构体保存完整信息。在创建一个学生的信息时，通过malloc函数分配内存，存放结构体，然后使用一个函数来读取、操作这些数据。

在object-c中，有一个新的方式来表现这样的信息，那就是类(class)。如果对OOP（面向对象编程）语言如Java、C++有一定了解的话，理解这个就不困难了。从OOP的角度来说，有一句名言，叫做“一切都是对象”。

类与对象(object)的关系类似与模具与成品之间的关系，类是模具，对象是成品，也就是说，类定义了通用结构，对象表示具体产品。
对象产生的每个类叫做实例(instance)。
类实例通过实例变量(instance variable)保存属性值。

与c语言中的函数对应的，object-c的类具有方法(method)，方法可以访问对象的实例变量，通过向对象发送消息(message)，能让对象执行相应的方法。还记得在[第一章的分析与术语](http://readsandthoughts.sinaapp.com/practice_in_object_c_j/)中说过的，object-c文件以.m结尾，代表message，指的是object-c的一个主要特性，称之为.m文件。

2 对象创建
在object-c中，对象的创建分为两步，分配与初始化：



	
  * 分配（allocation）
是一个新对象诞生的过程。是从OS获得一块内存并将其指定为存放对象的实例变量的位置。同时，alloc方法还顺便将这块内存区域全部初始化为0。

	
  * 初始化(initialization）
从OS取得一块内存，准备用于存储对象。


对象必须在分配并且初始化后才能使用，所以关于初始化的推荐写法如下：
[code lang="c"][[类名 alloc] init];[/code]
这种写法也叫嵌套消息发送(nested message send).

而不是使用：
[code lang="c"]类名 *指针名 = [类名 alloc] ;
[指针名 init];[/code]
后面一种写法可能会造成一定的问题，部分原因是与object-c使用的类簇具有关系。更深层次的原因在后面继续探讨。


3 方法与消息

在对象初始化完成之后，就可以向其发送更多的消息了。
在object-c中，消息的结构如下：
[code lang="c"][instance selector: argument];[/code]
其中instance是接收方，表示方法的执行者；selector是选择器，也就是要执行的方法名；argument是参数，可以有多个，也可以没有。
上述示例的描述就是：向instance实例请求以argument作为参数的方法selector。

如果方法具有多个参数，示例如下：
[code lang="c"][instance selector: argument label2: argument2 label3: arguments];[/code]
可以看出，与其它OOP语言如Java相比，object-c的参数形式更加明显，参数与标签的配对关系非常明确。需要注意的是，一对方括号表示一条正在发送的消息。
消息与方法的区别在于：消息是要求方法执行的动作；而方法是一块代码。


4 释放对象
可以通过如下方式释放对象：
[code lang="c"][instance release];[/code]


需要注意的是，虽说对象已经释放，但变量的值还在，也就是说变量的指针指向了一个已经被释放的对象，这时候再发送消息就会出问题。




可以将变量设为nil值，nil在object-c中是指针值为0的指针，类似与Java中的null，c中的NULL。给nil发消息不会发生任何事情。





5 总结

对象是对具体事物的抽象描述，类是对同一类型的对象的模子。对象具有可以执行的方法，通过给类实例发送消息可以请求对象执行相对应的方法。对象的创建分为分配和初始化两步，但推荐两步骤合二为一的写法。
