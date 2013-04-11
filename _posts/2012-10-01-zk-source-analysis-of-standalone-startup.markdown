---
author: shupeng.lisp
comments: true
date: 2012-10-01 20:44:57
layout: post
slug: zk_source_analysis_of_standalone_startup
title: ZooKeeper源码分析之一——单机模式启动过程
wordpress_id: 24
categories:
- 读代码
---

了解和学习了下zookeeper，将阅读的部分源代码过程分享下，关于zookeeper的原理性文章请参考[ZooKeeper论文阅读笔记](http://readsandthoughts.sinaapp.com/zookeeper_paper_notes/)
：

<!--break-->

第一步，需要了解启动的配置文件在哪里，根据[官方文档的单机模式配置](http://zookeeper.apache.org/doc/trunk/zookeeperStarted.html#sc_InstallingSingleMode)，需要在conf目录下配置一个叫zoo.cfg的文件，在不更改其它的条件下，查看下环境脚本吧：

window下：

[sourcecode lang="shell"]
set ZOOCFGDIR=%~dp0%..\conf
set ZOOCFG=%ZOOCFGDIR%\zoo.cfg
[/sourcecode]

Linux下：

[sourcecode lang="shell"]
ZOOBINDIR=${ZOOBINDIR:-/usr/bin}
ZOOCFGDIR="$ZOOBINDIR/../conf"
ZOOCFG="zoo.cfg"
ZOOCFG="$ZOOCFGDIR/$ZOOCFG"
[/sourcecode]

对比发现是一样的，都是在与bin平行的conf下的zoo.cfg，所以，zookeeper的教学文档中也就有配置这个zoo.cfg的要求了。将启动文件调成DEBUG模式，便于跟踪启动过程，当然，不设置也没关系，知道了在哪里读取，直接看代码也是一样的。作为一个职业看源码的，调成DEBUG模式会更有好处。

启动主函数：org.apache.zookeeper.server.quorum.QuorumPeerMain中的main方法：

很简单，只有一句关键内容:

[sourcecode language="java"]

QuorumPeerMain main = new QuorumPeerMain();
 try {
 main.initializeAndRun(args);
 } catch{...}
[/sourcecode]

这里的args就是zoo.cfg;
朝下走，进入到关键的一步：

[sourcecode lang="java"]
protected void initializeAndRun(String[] args)
throws ConfigException, IOException
 {
 QuorumPeerConfig config = new QuorumPeerConfig();
 if (args.length == 1) {
 config.parse(args[0]);
 }

// Start and schedule the the purge task
 DatadirCleanupManager purgeMgr = new DatadirCleanupManager(config
 .getDataDir(), config.getDataLogDir(), config
 .getSnapRetainCount(), config.getPurgeInterval());
 purgeMgr.start();

if (args.length == 1 && config.servers.size() > 0) {
 runFromConfig(config);
 } else {
 LOG.warn("Either no config or no quorum defined in config, running "
 + " in standalone mode");
 // there is only server in the quorum -- run as standalone
 ZooKeeperServerMain.main(args);
 }
 }[/sourcecode]

初看流程，就可以发现这个方法大致分为三步：

1. 解析配置文件；

2. 数据清理；

3. 选择模式并转发；

我们先看第一步：解析配置文件：

[sourcecode lang="java"]
public void parse(String path) throws ConfigException {
 File configFile = new File(path);
LOG.info("Reading configuration from: " + configFile);
try {
 if (!configFile.exists()) {
 throw new IllegalArgumentException(configFile.toString()
 + " file is missing");
 }

Properties cfg = new Properties();
 FileInputStream in = new FileInputStream(configFile);
 try {
 cfg.load(in);
 } finally {
 in.close();
 }

parseProperties(cfg);
 } catch (IOException e) {
 throw new ConfigException("Error processing " + path, e);
 } catch (IllegalArgumentException e) {
 throw new ConfigException("Error processing " + path, e);
 }
 }
[/sourcecode]

这里没什么新奇的事情，配置文件是k-v的，因此就以读取Properties的形式load进来，然后将和实际场景相关的内容放到parseProperties(cfg)中，来看一下：

[sourcecode lang="java"]
public void parseProperties(Properties zkProp)
 throws IOException, ConfigException {
 int clientPort = 0;
 String clientPortAddress = null;
 for (Entry entry : zkProp.entrySet()) {
 String key = entry.getKey().toString().trim();
 String value = entry.getValue().toString().trim();
 if (key.equals("dataDir")) {
 dataDir = value;
 } else if (key.equals("dataLogDir")) {
 dataLogDir = value;
 } else if (key.equals("clientPort")) {
 clientPort = Integer.parseInt(value);
 } else if (key.equals("clientPortAddress")) {
……
if (dataDir == null) {
 throw new IllegalArgumentException("dataDir is not set");
 }
 if (dataLogDir == null) {
 dataLogDir = dataDir;
 } else {
 if (!new File(dataLogDir).isDirectory()) {
 throw new IllegalArgumentException("dataLogDir " + dataLogDir
 + " is missing.");
 }
 }
……
if (servers.size() == 0) {
 if (observers.size() > 0) {
 throw new IllegalArgumentException(
"Observers w/o participants is an invalid configuration");
 }
 // Not a quorum configuration so return immediately - not an error
 // case (for b/w compatibility), server will default to standalone
 // mode.
 return;
 }
[/sourcecode]

这里更加简单的，就是将设置的配置信息与系统需要的关键字进行对比，然后设置；后面部分就是检查设置的值

是否符合系统约束，这一点也是一个严格的系统常用的知识；在检查时，检查到servers总数是0时(设置server是

集群模式使用的，单机模式得到的是0），就表示系统的配置读取与设置结束了；

第二步：清理数据

[sourcecode lang="java"]
// Start and schedule the the purge task
 DatadirCleanupManager purgeMgr = new DatadirCleanupManager(config
 .getDataDir(), config.getDataLogDir(), config
 .getSnapRetainCount(), config.getPurgeInterval());
 purgeMgr.start();
[/sourcecode]

初始化一个DatadirCleanupManager并启动,看看启动逻辑有没特别之处：

[sourcecode lang="java"]
public void start() {
 if (PurgeTaskStatus.STARTED == purgeTaskStatus) {
 LOG.warn("Purge task is already running.");
 return;
 }
 // Don't schedule the purge task with zero or negative purge interval.
 if (purgeInterval仍然是先检查设置，是否需要清理；需要的话则根据设置的值启动个定时线程任务，线程名字叫"PurgeTask"； 下面进入第三步，也是最复杂的一步，选择模式并转发处理：
[sourcecode lang="java"] if (args.length == 1 && config.servers.size() > 0) {
 runFromConfig(config);
 } else {
 LOG.warn("Either no config or no quorum defined in config, running "
 + " in standalone mode");
 // there is only server in the quorum -- run as standalone
 ZooKeeperServerMain.main(args);
 }
[/sourcecode]

很明显config.servers.size() = 0,开始请出ZooKeeperServerMain的main吧。

[sourcecode lang="java"]
if (args.length == 1 && config.servers.size() > 0) {
 runFromConfig(config);
 } else {
 LOG.warn("Either no config or no quorum defined in config, running "
 + " in standalone mode");
 // there is only server in the quorum -- run as standalone
 ZooKeeperServerMain.main(args);
 }
[/sourcecode]
[sourcecode lang="java"]
public static void main(String[] args) {
 ZooKeeperServerMain main = new ZooKeeperServerMain();
 try {
 main.initializeAndRun(args);
 } catch{……}
[/sourcecode]

惊人的相似啊，哈哈，直接走：

[sourcecode lang="java"]
protected void initializeAndRun(String[] args)
 throws ConfigException, IOException
 {
 try {
 ManagedUtil.registerLog4jMBeans();
 } catch (JMException e) {
 LOG.warn("Unable to register log4j JMX control", e);
 }

ServerConfig config = new ServerConfig();
 if (args.length == 1) {
 config.parse(args[0]);
 } else {
 config.parse(args);
 }

runFromConfig(config);
 }
[/sourcecode]

这里的config解析就不说了，进去一看发现……还是相似，单参数直接代理给QuorumPeerConfig，多参数也就限制了2-4个参数，差不多可以说，是将前面读取的配置（那里主要是针对整个系统的）重新设置到当前机器上（这里是针对具体机器的）。
真正的家伙终于来了，嘿嘿~

[sourcecode lang="java"]
public void runFromConfig(ServerConfig config) throws IOException {
 LOG.info("Starting server");
 try {
 // Note that this thread isn't going to be doing anything else,
 // so rather than spawning another thread, we will just call
 // run() in this thread.
 // create a file logger url from the command line args
 ZooKeeperServer zkServer = new ZooKeeperServer();

FileTxnSnapLog ftxn = new FileTxnSnapLog(new
 File(config.dataLogDir), new File(config.dataDir));
 zkServer.setTxnLogFactory(ftxn);
 zkServer.setTickTime(config.tickTime);
 zkServer.setMinSessionTimeout(config.minSessionTimeout);
 zkServer.setMaxSessionTimeout(config.maxSessionTimeout);
 cnxnFactory = ServerCnxnFactory.createFactory();
 cnxnFactory.configure(config.getClientPortAddress(),
 config.getMaxClientCnxns());
 cnxnFactory.startup(zkServer);
 cnxnFactory.join();
 if (zkServer.isRunning()) {
 zkServer.shutdown();
 }
 } catch (InterruptedException e) {
 // warn, but generally this is ok
 LOG.warn("Server interrupted", e);
 }
 }
[/sourcecode]

创建ZooKeeperServer，初始化事务快照日志，设置server基本参数；之后获取连接工厂：

[sourcecode lang="java"]
static public ServerCnxnFactory createFactory() throws IOException {
 String serverCnxnFactoryName =
 System.getProperty(ZOOKEEPER_SERVER_CNXN_FACTORY);
 if (serverCnxnFactoryName == null) {
 serverCnxnFactoryName = NIOServerCnxnFactory.class.getName();
 }
 try {
 return (ServerCnxnFactory) Class.forName(serverCnxnFactoryName)
 .newInstance();
 } catch (Exception e) {
 IOException ioe = new IOException("Couldn't instantiate "
 + serverCnxnFactoryName);
 ioe.initCause(e);
 throw ioe;
 }
 }
[/sourcecode]

根据设置的类型来选择连接工厂，这里我们得到的是NIOServerCnxnFactory；获取工厂之后，需要配置与初始化一些本工厂特定的属性，也就是configure方法的职责：

[sourcecode lang="java"]
public void configure(InetSocketAddress addr, int maxcc) throws IOException {
 if (System.getProperty("java.security.auth.login.config") != null) {
 try {
 saslServerCallbackHandler = new SaslServerCallbackHandler(Configuration.getConfiguration());
 login = new Login("Server",saslServerCallbackHandler);
 login.startThreadIfNeeded();
 }
 catch (LoginException e) {
 throw new IOException("Could not configure server because SASL configuration did not allow the "
 + " Zookeeper server to authenticate itself properly: " + e);
 }
 }
 thread = new Thread(this, "NIOServerCxn.Factory:" + addr);
 thread.setDaemon(true);
 maxClientCnxns = maxcc;
 this.ss = ServerSocketChannel.open();
 ss.socket().setReuseAddress(true);
 LOG.info("binding to port " + addr);
 ss.socket().bind(addr);
 ss.configureBlocking(false);
 ss.register(selector, SelectionKey.OP_ACCEPT);
 }[/sourcecode]

这里启动了一个daemon线程，线程名： "NIOServerCxn.Factory:" + $addr，并做了NIO的操作初始化；随后，工厂启动该ZooKeeperServer；

[sourcecode lang="java"]
public void startup(ZooKeeperServer zks) throws IOException,
 InterruptedException {
 start();
 zks.startdata();
 zks.startup();
 setZooKeeperServer(zks);
 }
[/sourcecode]
[sourcecode lang="java"]
public void start() {
 // ensure thread is started once and only once
 if (thread.getState() == Thread.State.NEW) {
 thread.start();
 }
 }
[/sourcecode]

如果线程未启动，则启动它；

[sourcecode lang="java"]
public void startdata()
 throws IOException, InterruptedException {
 //check to see if zkDb is not null
 if (zkDb == null) {
 zkDb = new ZKDatabase(this.txnLogFactory);
 }
 if (!zkDb.isInitialized()) {
 loadData();
 }
 }
[/sourcecode]

启动ZooKeeper使用的db，是内存数据库；

[sourcecode lang="java"]
public void startup() {
 createSessionTracker();
 setupRequestProcessors();

registerJMX();

synchronized (this) {
 running = true;
 notifyAll();
 }
 }
[/sourcecode]

一步步来吧：

[sourcecode lang="java"]
protected void createSessionTracker() {
 sessionTracker = new SessionTrackerImpl(this, zkDb.getSessionWithTimeOuts(),
 tickTime, 1);
 ((SessionTrackerImpl)sessionTracker).start();
 }
[/sourcecode]

这里SeesionTrackerImpl是Thread的子类，这里启动了个新线程，并且线程的名字为：SessionTracker；

[sourcecode lang="java"]
protected void setupRequestProcessors() {
 RequestProcessor finalProcessor = new FinalRequestProcessor(this);
 RequestProcessor syncProcessor = new SyncRequestProcessor(this,
 finalProcessor);
 ((SyncRequestProcessor)syncProcessor).start();
 firstProcessor = new PrepRequestProcessor(this, syncProcessor);
 ((PrepRequestProcessor)firstProcessor).start();
 }
[/sourcecode]

这里具有多个RequestProcessor，并且使用了职责链将其串联起来：

其中这里的SyncRequestProcessor和PrepRequestProcessor都是Thread的子类，这里也启动了两个线程，

SyncRequestProcessor的名字是："SyncThread:" + zks.getServerId()，单机模式下serverId为0；

PrepRequestProcessor的名字为："ProcessThread(sid:" + zks.getServerId() + " cport:" + zks.getClientPort() + "):"，cport在连接工厂为空时返回-1；

随后注册JMX，并发送通知给其他线程；至此，server端启动结束；

这时，可以使用线程dump工具，查看下启动了哪些线程，验证下分析过程：

[sourcecode lang="shell"]
"ProcessThread(sid:0 cport:-1):" prio=6 tid=0x16c24400 nid=0x1154 waiting on condition [0x173bf000]
 java.lang.Thread.State: WAITING (parking)
 at sun.misc.Unsafe.park(Native Method)
 - parking to wait for  (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObj
 at java.util.concurrent.locks.LockSupport.park(Unknown Source)
 at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(Unknown Source)
 at java.util.concurrent.LinkedBlockingQueue.take(Unknown Source)
 at org.apache.zookeeper.server.PrepRequestProcessor.run(PrepRequestProcessor.java:119)

"SyncThread:0" prio=6 tid=0x170b8400 nid=0x158c waiting on condition [0x1736f000]
 java.lang.Thread.State: WAITING (parking)
 at sun.misc.Unsafe.park(Native Method)
 - parking to wait for  (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObj
 at java.util.concurrent.locks.LockSupport.park(Unknown Source)
 at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(Unknown Source)
 at java.util.concurrent.LinkedBlockingQueue.take(Unknown Source)
 at org.apache.zookeeper.server.SyncRequestProcessor.run(SyncRequestProcessor.java:97)

"SessionTracker" prio=6 tid=0x16c25800 nid=0x1014 in Object.wait() [0x1731f000]
 java.lang.Thread.State: TIMED_WAITING (on object monitor)
 at java.lang.Object.wait(Native Method)
 - waiting on  (a org.apache.zookeeper.server.SessionTrackerImpl)
 at org.apache.zookeeper.server.SessionTrackerImpl.run(SessionTrackerImpl.java:146)
 - locked  (a org.apache.zookeeper.server.SessionTrackerImpl)

"NIOServerCxn.Factory:0.0.0.0/0.0.0.0:2181" daemon prio=6 tid=0x1708cc00 nid=0x1004 runnable [0x172cf000]
 java.lang.Thread.State: RUNNABLE
 at sun.nio.ch.WindowsSelectorImpl$SubSelector.poll0(Native Method)
 at sun.nio.ch.WindowsSelectorImpl$SubSelector.poll(Unknown Source)
 at sun.nio.ch.WindowsSelectorImpl$SubSelector.access$400(Unknown Source)
 at sun.nio.ch.WindowsSelectorImpl.doSelect(Unknown Source)
 at sun.nio.ch.SelectorImpl.lockAndDoSelect(Unknown Source)
 - locked  (a sun.nio.ch.Util$2)
 - locked  (a java.util.Collections$UnmodifiableSet)
 - locked  (a sun.nio.ch.WindowsSelectorImpl)
 at sun.nio.ch.SelectorImpl.select(Unknown Source)
 at org.apache.zookeeper.server.NIOServerCnxnFactory.run(NIOServerCnxnFactory.java:194)
 at java.lang.Thread.run(Unknown Source)

"main" prio=6 tid=0x00317400 nid=0x1274 in Object.wait() [0x0093f000]
 java.lang.Thread.State: WAITING (on object monitor)
 at java.lang.Object.wait(Native Method)
 - waiting on  (a java.lang.Thread)
 at java.lang.Thread.join(Unknown Source)
 - locked  (a java.lang.Thread)
 at java.lang.Thread.join(Unknown Source)
 at org.apache.zookeeper.server.NIOServerCnxnFactory.join(NIOServerCnxnFactory.java:318)
 at org.apache.zookeeper.server.ZooKeeperServerMain.runFromConfig(ZooKeeperServerMain.java:113)
 at org.apache.zookeeper.server.ZooKeeperServerMain.initializeAndRun(ZooKeeperServerMain.java:86)
 at org.apache.zookeeper.server.ZooKeeperServerMain.main(ZooKeeperServerMain.java:52)
 at org.apache.zookeeper.server.quorum.QuorumPeerMain.initializeAndRun(QuorumPeerMain.java:116)
 at org.apache.zookeeper.server.quorum.QuorumPeerMain.main(QuorumPeerMain.java:78)
"Low Memory Detector" daemon prio=6 tid=0x16be5c00 nid=0x1400 runnable [0x00000000]
 java.lang.Thread.State: RUNNABLE

"C1 CompilerThread0" daemon prio=10 tid=0x16bdb400 nid=0x814 waiting on condition [0x00000000]
 java.lang.Thread.State: RUNNABLE

"JDWP Command Reader" daemon prio=6 tid=0x16bd2400 nid=0x1300 runnable [0x00000000]
 java.lang.Thread.State: RUNNABLE

"JDWP Event Helper Thread" daemon prio=6 tid=0x16bd0800 nid=0x1190 runnable [0x00000000]
 java.lang.Thread.State: RUNNABLE

"JDWP Transport Listener: dt_socket" daemon prio=6 tid=0x16bcd400 nid=0x274 runnable [0x00000000]
 java.lang.Thread.State: RUNNABLE

"Attach Listener" daemon prio=10 tid=0x16bbc400 nid=0x17c8 waiting on condition [0x00000000]
 java.lang.Thread.State: RUNNABLE

"Signal Dispatcher" daemon prio=10 tid=0x16bbb000 nid=0xf28 runnable [0x00000000]
 java.lang.Thread.State: RUNNABLE

"Finalizer" daemon prio=8 tid=0x16baa800 nid=0x1278 in Object.wait() [0x16d1f000]
 java.lang.Thread.State: WAITING (on object monitor)
 at java.lang.Object.wait(Native Method)
 - waiting on  (a java.lang.ref.ReferenceQueue$Lock)
 at java.lang.ref.ReferenceQueue.remove(Unknown Source)
 - locked  (a java.lang.ref.ReferenceQueue$Lock)
 at java.lang.ref.ReferenceQueue.remove(Unknown Source)
 at java.lang.ref.Finalizer$FinalizerThread.run(Unknown Source)

"Reference Handler" daemon prio=10 tid=0x16ba5c00 nid=0xfa0 in Object.wait() [0x16ccf000]
 java.lang.Thread.State: WAITING (on object monitor)
 at java.lang.Object.wait(Native Method)
 - waiting on  (a java.lang.ref.Reference$Lock)
 at java.lang.Object.wait(Object.java:485)
 at java.lang.ref.Reference$ReferenceHandler.run(Unknown Source)
 - locked  (a java.lang.ref.Reference$Lock)

"VM Thread" prio=10 tid=0x16ba2000 nid=0xaa4 runnable

"VM Periodic Task Thread" prio=10 tid=0x16be7c00 nid=0xdf4 waiting on condition
[/sourcecode]

到这里，服务器移动完毕，线程检查完毕，可以等待客户端发请求了。
总体看来，这里的整体启动流程还是很清晰的：



	
  1. 首先读取整个服务的配置信息并做检查；

	
  2. 然后是根据配置的信息做一些清理工作；

	
  3. 第三步是启动具体的server，而启动单个server会有很多事情做，会开启一些新线程，这在之后的交互会显得有些复杂。


但总体来说，流程就在这里了，关于其中的一些细节问题和代码亮点，会在后续文章中慢慢整理。
