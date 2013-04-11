---
author: shupeng.lisp
comments: true
date: 2012-10-05 16:05:05
layout: post
slug: understand_zk_concepts
title: ZooKeeper中的概念剖析
wordpress_id: 61
categories: [技术]

[ZooKeeper](http://zookeeper.apache.org)作为一个分布式协调服务，涉及到了很多具有特色的名词与概念，本文意在对此部分内容进行下梳理。本文的内容是连接[ZooKeeper学术文章](http://readsandthoughts.sinaapp.com/zookeeper_paper_notes/)是系统实现的纽带，增强本文的理解对ZooKeeper的整体的理解帮助巨大。

本文主要内容来自于官方文档：[Developing Distributed Applications that use ZooKeeper](http://zookeeper.apache.org/doc/trunk/zookeeperProgrammers.html)



#### 1. 数据模型



ZooKeeper具有和分布式文件系统一样的层次命名空间，不同之处在于每个节点(Node)可以有和孩子一样的数据，这一点和文件系统中目录也是文件一样（目录是一种特殊的文件）。节点路径都是标准的、以”/”分隔的绝对路径，没有相对路径。Unicode字符中除以下部分之外都可以用作路径：

<!--break-->

	
  * Null字符(\u0000)，在绑定c语言时会出问题；

	
  *  \u0001-\u0019，\u007F-\u009F，这些字符要么无法显示，要么显示有歧义；

	
  *  \ud800=uF8FFF,\uFFF0-uFFFF,\uXFFFE-\uXFFFF（X为数字1—E）,\uF0000-\uFFFFF；

	
  *  “.”可以用作部分名，但”.”、”..”不能单独用来表示一个节点。因为ZooKeeper不支持相对路径。




##### 1.1  Znodes


ZooKeeper树中的每一个节点被称为Znode。Znode维持一个包含数据改变、acl改变的版本号的Stat数据结构。Stat结构还包括了时间戳，版本号和时间戳一起来使得ZooKeeper可以验证缓存和协调更新。Znode的数据发生变化时，版本号递增。

注意:在ZooKeeper中，



	
  * Zonde表示数据节点

	
  * Server表示组成ZooKeeper服务的机器

	
  * Quorum peer表示组成全体的serve，

	
  * Client表示使用ZooKeeper服务的任何主机或进程。


为了保持原意，之后的名词将会原文与译文混合出现。

Znode是程序员访问的主要实体，有如下几个显著特征：

	
  *   **Watches**


Client可以在Znode上设置watch，对该Znode做出改变将触发并清除watch。当watch被触发，ZooKeeper发送给Client一个通知。

	
  *   **数据访问**


存储在Znode的数据时原子性读写的：读得到该znode的所有数据字节，写替换所有数据。每个节点维护一个ACL来限制访问。

ZooKeeper不是用来做通用数据库或存储大对象的，而是用来关系协调数据的。协调数据的特征就是都相当较小，以KB作单位计算足够了。ZooKeeper中有关于znode数据大小的健康型检查，默认是1MB。如果需要存储大数据，通常是将数据存储在如NFS和HDFS这样的大存储上，然后在ZooKeeper中存储存储位置的指针。

	
  *    **短命节点（Ephemeral Node****）**


只在创建znode的session是活动时才存在。Session中止则这些节点被自动删除，因此这种节点不能拥有孩子。

	
  *    **顺序节点（Sequence Nodes****）---****唯一命名**


在创建节点时，可以指定说明需要在路径名之后增加一个单调递增的计数器。这个计数器对于父znode来说是唯一的，这个计数器具有如下格式：%010d，计数器是有符号的int型，因此在超过2147483647后会溢出。



##### 1.2  时间管理



ZooKeeper以多种方式跟踪时间：



	
  *   Zxid：ZooKeeper的事务ID，每个ZooKeeper状态改变都会接受到一个zxid形式的邮戳。这个暴露了ZooKeeper所有改变的全序关系，每个改变具有唯一的zxid，并且如果zxid1<zxid2则zxid1发生在zxid2之前。

	
  *  版本号：node的每次改变都会触发其中一个版本号增加。三个版本号，version是用来表示znode数据改变数量的，cversion是用来表示znode孩子的改变数量的，aversion是用来表示znode的acl改变数量的。

	
  *  Ticks：在使用多机模式时，server使用tick来定义诸如状态上传、session超时、peers之间连接超时等事件定时。这个属性是通过最小session超时（2倍tick事件）来间接配置的。

	
  *  真实时间：除了在znode创建和修改时把时间戳放到stat结构时，ZooKeeper不使用真实时间（时钟时间）。





##### 1.3  Stat结构


每个znode的stat结构由以下几个字段组成：



	
  * Czxid：引起创建znode的zxid的改变；

	
  * Mzxid：最后修改该znode的zxid改变；

	
  * Ctime：从znode创建的epoch（时期）开始以ms计数的时间；

	
  * Mtime：从znode最后修改的epoch（时期）开始以ms计数的时间；

	
  * Version：znode数据改变的数量；

	
  *  Cversion：znode孩子改变的数量；

	
  *  Aversion：znode的acl改变的数量；

	
  *  ephemeralOwner：如果是短命节点，则为该znode所有者的session id；如果不是短命节点，则为0；

	
  *  dataLength：znode的数据字段长度；

	
  *  numChildren：znode的孩子数量。





#### 2.  会话（Session）



ZooKeeper client通过使用语言绑定创建服务handle来与ZooKeeper服务建立会话。一旦会话建立，handle开始进入CONNECTING状态，客户库开始连接组成ZooKeeper服务的server，并在成功后转换为CONNECTED状态。如果发生会话过期、验证失败等不可恢复的错误，或者是应用程序显示关闭handle，则进入CLOSED状态。

为了建立client会话，应用程序代码必须提供包含一组以逗号分隔的host:port对，然后会选择任意一个进行连接，在失败时会自动选择下一个server来重建连接。

在3.2.0版本之后：一个可选的”chroot”后缀可以添加到连接字符串。

在client获取到ZooKeeper服务的handle时，ZooKeeper创建了一个分配给client的64位数字表示的会话。如果client连接一个不同的ZooKeeper server，将把Seesion Id作为连接握手的一部分。

创建session的一个参数是session超时的ms数。

不建议在失去连接时建立一个新的session对象，ZooKeeper client库将自动重连。客户库有能处理羊群效应等的启发式方法。仅仅在被通知session过期后才重建session对象。Session过期由ZooKeeper集群管理。

ZooKeeper会话建立调用的另一个参数是默认watcher。在client端发生任何变化时通知watcher。

由client发送请求来保持session有效。如果session闲置了一段时间，client将发送一个PING请求。PING请求不但可以让server知道client是活动的，而且可以让client验证连接的server是活动的。

连接丢失的两种情况：



	
  * 应用程序在一个不再存活/有效的会话上调用操作；

	
  * Client在与有悬挂操作的server失去连接。比如，一个悬挂的异步调用。


3.2.0新增：SessionMovedException，这是client不可见的内部错误。发生在请求在一个会话的连接中被接受，会话重建连接到另外一个server上了。



#### 3.  观察（watch）



ZooKeeper的所有读操作----getData(), getChildren(), exists()都有设置watch的选项。

Watch的定义：watch事件是一次触发的，在设置了watch的数据发生变化时，watch发送给设置它的client。



	
  *  一次触发：触发一次并被清理；

	
  * 发送到client：意味着在成功返回码到达发起改变的client之前，事件可能在去client的路上而还没有到达。Watch是异步发送给watcher的。ZooKeeper提供一个有序保证：client在它设置了watch的资源上，在看到watch事件之前看不到改变。关键点在于不同client看见的具有一直的顺序。

	
  *  Watch设置的数据：表示node可以以不同的方式改变。ZooKeeper有两种watch，数据watch和孩子watch。getData()和exists()设置数据watch，getChildren()设置孩子watch。


Watch是由client连接的server本地维持的，这样可以使得watch可以轻量的设置、维持和分发。Watch可能丢失的场景在于：如果在znode断开连接器件创建并删除了znode，存在检查的watch将会丢失。




##### 3.1  对watch的担保





	
  * 相当于其它事件、其它watch和异步回复，watch是有序的。ZooKeeper客户库确保有序分发。

	
  * 如果设置了watch，Client先看到znode的watch事件，后看见响应的数据。

	
  * Watch事件的顺序与ZooKeeper服务看见的更新顺序一致。





##### 3.2  watch注意事项





	
  * Watch是一次触发的。期望持续获得通知，在获取watch事件之后要重新设置。

	
  * 由于watch是一次触发，加上网络延迟，在得到event与重新设置watch之间，znode可能已经被改变了多次。不可能可靠的得到znode的每次改变。

	
  * Watch对象，或者function/context对，对于一个给定的通知只会触发一次。

	
  * 如果从server失去连接，直到连接重建之后才能获取到watch。





#### 4. 访问控制（Access control）


ZooKeeper使用ACLs来控制znode的访问。ACL实现非常类似UNIX文件访问权限：使用权限位来允许/不允许对权限位应用节点和范文的各种操作。与标准UNIX权限的不同之处是，ZooKeeper节点不是由用户、组和所有人三个标准范围来限制的，ZooKeeper没有znode所有者的概念，作为替代，ACL指定一组id和与这些id关联的权限。

值得注意的是，ACL仅仅呼吁某个指定znode，不适用与孩子节点，ACLs不是递归的。

ZooKeeper支持可插拔的验证模式；ACLs以（schema:expression, perms）对来表示。



##### 4.1  ACL权限





	
  * CREATE：可以创建一个孩子节点；

	
  * READ：可以从节点得到数据，可以查看它的孩子列表；

	
  * WRITE：可以给节点设置数据；

	
  * DELETE：可以删除一个孩子节点；

	
  * ADMIN：可以设置权限。


ADMIN权限是因为ZooKeeper没有文件所有者的概念，ADMIN权限有一定的所有者意思。

ZooKeeper不支持LOOKUP权限，每个人都有隐含的LOOKUP权限，允许stat一个节点，除此之外不能做其它的。



##### 4.2  内建ACL模式





	
  * world：具有单id，anyone，表示任何人；

	
  * auth：不使用任何id，表示任何通过验证的用户；

	
  * digest：使用”username:password”字符串产生MD5哈希，随后使用这个值作为ACL ID标识。验证仪明文发送。在ACL表达式时将使用”username:base64”编码乘SHA1密码摘要。

	
  * ip：使用client主机IP作为ACL ID标识。





#### 5. 一致性保证（Consistence guarantees）


ZooKeeper读比写快的原因是由于读可以提供较早的数据，这是由于ZooKeeper具有一致性保证：



	
  * 顺序一致性：从一个client发出的申请将以发送的顺序采纳；

	
  * 原子性：更新要么成功，要么失败，不存在部分成功；

	
  * 单一系统镜像：client不管连接到哪个server，看到的服务视图是相同的；

	
  * 可靠性：一旦更新被采纳，从该时间点直到有client覆盖，该更新一直有效，这个保证由两个推论：

	
    * 如果client得到成功返回码，更新被采纳。

	
    * 任何client可见的更新，包括读请求或成功的更新，在server失效也不会被回滚。






	
  * 时效性：系统的client视图在给定时间界限被更新（几十秒量级）。要么client在这个界限看见系统改变，要么client探测到服务终端。

	
  * ZooKeeper不担保：跨client视图的同时一致。ZooKeeper自身不担保在所有servers的同步更新，但ZooKeeper原语可以用来构造提供有效client同步的高层功能。


