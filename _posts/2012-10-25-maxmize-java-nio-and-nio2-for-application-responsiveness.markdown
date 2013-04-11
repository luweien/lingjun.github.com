---
author: shupeng.lisp
comments: true
date: 2012-10-25 08:52:30
layout: post
slug: maxmize_java_nio_and_nio2_for_application_responsiveness
title: '最大化Java NIO和NIO.2的五种方式 ---利用NIO API构建高响应性的Java应用 '
wordpress_id: 116
categories: [技术]
---
 `本文为译文`
Java NIO---新输入/输出API包是在2002年发行的J2SE1.4中引入的。Java NIO的目的在于提升Java平台上I/O密集的编程。十年过去了，很多Java程序员仍然不知道如何最好的使用NIO，更少有人关注到了在Java SE7中引入的NIO.2（More NIO）。在这个教学中，将看到Java编码中利用NIO和NIO.2优点的五个简单示例。

Java中NIO和NIO.2的最初贡献是提升I/O处理的性能，而I/O处理是Java应用开发中的核心之一。该API既不容易上手，也不是日常必备。然而，如果正确使用Java NIO和NIO.2，便能大幅减少某些通用I/O操作的时间。这才是NIO和NIO.2的强大之处，本文列举了相关的五个简单方式。

1 变更通知（因为人人需要监听者）

2 利用选择器实现多路传输

3 通道---承诺和现实

4 内存映射---重要意义

5 字符编码和搜索


  <!--break-->

## NIO上下文


为什么达10年之久的增强仍然是Java的新输入/输出包？这是由于对于大多数Java程序员来说，基本的/O操作已经够用了。大多数程序员在日常工作不需要使用NIO。此外，NIO不仅仅是一个性能包，而是Java I/O相关功能的异构集合。NIO通过“离金属更近”来提高应用性能（意思是更接近底层---译者注），这意味着NIO和NIO.2 API暴露的是更底层的操作系统的入口点。NIO的这个折中使得在给我们在I/O上有更大控制权的同时，要求我们比使用基本I/O编程时要更仔细的练习。NIO的另一个值得注意的方面是应用表达，这一点在后面的例子将有所体现。




#### 开始NIO和NIO.2


有大量的NIO的好引用---看资源来查看部分选择的链接。为了开始体验NIO和NIO.2，J2SE和Java SE7的文档是必不可少的。为了运行本文的例子，需要使用JDK7或更高版本。

很多开发者第一次使用NIO很有可能是发生在日常维护中：应用的功能一切正常，但响应很慢，于是有人建议使用NIO来加速。NIO在用来提升处理性能时大放异彩，但具体结果依赖于底层平台。（注意NIO是平台无关的。）如果你是第一次使用NIO，将让你仔细的测量。你可能会发现NIO加速应用性能的能力不仅仅依赖于OS，而且和特定的JVM、主机虚拟化上下文、海量存储特性、甚至数据都大有干系。测量很难通用化，然而最好在心中牢记这一点，尤其是在多目标的移动部署时。

言归正传，现在开始一起来探索NIO和NIO.2的五个重要功能吧。




## 1 变更通知（因为人人需要监听者）


吸引开发者对NIO或NIO.2产生兴趣的通常是由于Java应用的性能。在我的经历中，NIO.2的文件变更通知是NIO API最吸引人的特性。

很多企业级的应用在如下场景中需要做采取指定动作：

l  上传文件到FTP文件夹

l  配置定义发生改变

l  草图文档发生更新

l  另一个文件系统事件发生

这些都是变更通知或变更响应的例子。在Java（和其它语言）的早期版本，轮询(polling)是典型的探测变更事件的最好方式。轮询是一种特定的无限循环：检查文件系统或其它对象，将它和它的上一个已知状态做比较，如果没有变更，在一定间隔之后重新检查，而间隔可能是几百毫秒，或几十秒。

NIO.2展示了一个更好的方式来探测变更通知，例1是一个简单的例子：

例1：NIO.2中的变更通知

[code lang="java"]import java.nio.file.Path;
import java.nio.file.Paths;
import java.nio.file.StandardWatchEventKinds;
import java.nio.file.WatchEvent;
import java.nio.file.WatchKey;
import java.nio.file.WatchService;
import java.util.List;

public class Watcher {
    public static void main(String[] args) {
        Path this_dir = Paths.get(".");
        System.out.println(this_dir.toAbsolutePath());
        System.out.println("Now watching the current directory ...");

        try {
            WatchService watcher = this_dir.getFileSystem().newWatchService();
            this_dir.register(watcher, StandardWatchEventKinds.ENTRY_CREATE);

            WatchKey watckKey = watcher.take();

            List<WatchEvent> events = watckKey.pollEvents();
            for (WatchEvent event : events) {
                System.out.println("Someone just created the file '" + event.context().toString() + "'.");

           }

       } catch (Exception e) {
           System.out.println("Error: " + e.toString());
       }
    }
}[/code]

编译源代码，然后命令行执行。在同一个目录下，创建一个新文件；你可以touch example1，或者甚至copy Watcher.class example1。你将看到如下的变更通知消息：

Someone just create the file ‘example’.

这简单的例子描述了如何开始用Java访问NIO的语言功能。它也介绍了NIO.2的Watch类，对于变更通知来说，与基于轮询的传统I/O方法相比，Watch类更加直接和易用。

Watch的打印错误：

从本文复制源代码的时候请小心。注意：比如，在例一中的StardardWatchEventKinds对象拼写为复数。甚至Java.net.documentstion都丢掉了复数形式。（原文的？没有转义，译文已修正---译者注）

小建议：

与原先的轮询相比，NIO的通知是如此易用，甚至会引诱你遗漏需求分析。当你第一次使用监听器的时候，你应该仔细考虑这些语义。比如说，知道文件什么时候修改结束比知道什么时候开始就要有用的多。这类分析需要非常小心，尤其是在像在FTP中删除文件夹这种通用场景。NIO是一个带有一些不易察觉的强大包，它可以惩罚不速之客。




## 2 选择器和异步I/O：利用选择器实现多路传输


NIO的新人有时候会将NIO与“非阻塞I/O”联系起来。NIO远远不止是非阻塞，但这种误解也很有意义：Java中的基本I/O是阻塞的---意味着它将一直等待操作完成，然而非阻塞或者异步I/O是NIO中用的最多的几个功能之一。

与例一中所描述的文件系统监听器一样，NIO的非阻塞I/O是基于事件的。这意味着需要为I/O通道定义一个选择器（回调、监听器），然后处理之后的动作。当在选择器上发生了事件---例如一行输入到达，选择器苏醒并执行。所有这些都是在一个线程里发生的，这与传统Java I/O形成了鲜明对比。

例二描述了在一个多端口网络回应中使用NIO选择器，这个例子源自于Greg Travis在2003年写的例子，做了些许修改。Unix和类Unix系统具有有效的选择器实现，所以这种网络程序是Java编码网络程序的一个高性能模型。

例二：NIO选择器

[code lang="java"]import java.io.IOException;
import java.net.InetSocketAddress;
import java.net.ServerSocket;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.util.Iterator;
import java.util.Set;

public class MultiPortEcho {
    private int        ports[];
    private ByteBuffer echoBuffer = ByteBuffer.allocate(1024);

    public MultiPortEcho(int ports[]) throws IOException {
        this.ports = ports;

        configure_selector();
    }

    private void configure_selector() throws IOException {
        // Create a new selector
        Selector selector = Selector.open();

        // Open a listener on each port, and register each one
        // with the selector
        for (int i = 0; i < ports.length; ++i) {
            ServerSocketChannel ssc = ServerSocketChannel.open();
            ssc.configureBlocking(false);
            ServerSocket ss = ssc.socket();
            InetSocketAddress address = new InetSocketAddress(ports[i]);
            ss.bind(address);

            SelectionKey key = ssc.register(selector, SelectionKey.OP_ACCEPT);

            System.out.println("Going to listen on " + ports[i]);
        }

        while (true) {
            int num = selector.select();

            Set selectedKeys = selector.selectedKeys();
            Iterator it = selectedKeys.iterator();

            while (it.hasNext()) {
                SelectionKey key = (SelectionKey) it.next();

                if ((key.readyOps() & SelectionKey.OP_ACCEPT) == SelectionKey.OP_ACCEPT) {
                    // Accept the new connection
                    ServerSocketChannel ssc = (ServerSocketChannel) key.channel();
                    SocketChannel sc = ssc.accept();
                    sc.configureBlocking(false);

                    // Add the new connection to the selector
                    SelectionKey newKey = sc.register(selector, SelectionKey.OP_READ);
                    it.remove();

                    System.out.println("Got connection from " + sc);
                } else if ((key.readyOps() & SelectionKey.OP_READ) == SelectionKey.OP_READ) {
                    // Read the data
                    SocketChannel sc = (SocketChannel) key.channel();

                    // Echo data
                    int bytesEchoed = 0;
                    while (true) {
                        echoBuffer.clear();

                        int number_of_bytes = sc.read(echoBuffer);

                        if (number_of_bytes                             break;
                        }

                        echoBuffer.flip();

                        sc.write(echoBuffer);
                        bytesEchoed += number_of_bytes;
                    }

                    System.out.println("Echoed " + bytesEchoed + " from " + sc);

                    it.remove();
                }

            }
        }
    }

    static public void main(String args[]) throws Exception {
        if (args.length             System.err.println("Usage: java MultiPortEcho port [port port ...]");
            System.exit(1);
        }

        int ports[] = new int[args.length];

        for (int i = 0; i < args.length; ++i) {
            ports[i] = Integer.parseInt(args[i]);
        }

        new MultiPortEcho(ports);
    }
}
[/code]

编译源代码，然后从命令行带参数启动，类似java MultiPortEcho 8005 8006.一旦MultiPortEcho运行，启动一个简单的telnet或者其它终端模拟器在8005和8006端口运行，将看到程序回显了接收到的字符---这些都是在单一线程里完成的。




## 3 通道：承诺与现实


在NIO中，通道可以是任意读写对象。它的工作就是抽象文件和套接字。NIO通道支持方法的持续性收集，所以它能够用来编程而不需要依赖是否stdout、网络连接或已经在使用的其他通道的特殊场景。通道分享了Java基本I/O流的特性，流提供阻塞I/O，通道支持异步I/O。

NIO通常以性能优势作为卖点，其实更准确的说法应该是高响应性。实际上在某些情况下NIO比基本Java I/O还要差。比如对于一些小文件的简单顺序读写，直接的流实现可能比面向事件的基于通道的编码快两到三倍，这是由于通道在单独的线程中---比注册选择器在单一线程的通道要慢很多。

下次你需要定义涉及维度到流或者通道的编程问题，尝试着问自己以下几个问题：

l  必须要读写的I/O对象有多少？

l  在不同的I/O对象间是否有自然顺序，它们是否需要同时发生？

l  I/O对象仅仅持续一个短间隔还是在整个进程的生命周期都存在？

l  单线程与几个不同的线程，哪一个更自然？

l  网络交通可能与本地I/O一样还是具有不同模式？

这类分析是在决定什么时候使用流，什么时候使用通道的良好实践。记住，NIO和NIO.2不能代替基本I/O，它们仅仅是补充。




## 4 内存映射---重要之处


在使用NIO时，最一致的显著性能提升是在内存映射这里。内存映射是OS级别的服务，它使编程可以使用文件分段，类似于内存分区。

内存映射有大量的结果和暗示，远超过这里所说的。在高层上，它可以以内存访问而不是文件访问的速度做I/O访问。前者通常比后者快两个量级。例三是NIO内存映射功能的最小描述。

例三：NIO中的内存映射

[code lang="java"]import java.io.RandomAccessFile;
import java.nio.MappedByteBuffer;
import java.nio.channels.FileChannel;

public class mem_map_example {
    private static int    mem_map_size = 20 * 1024 * 1024;
    private static String fn           = "example_memory_mapped_file.txt";

    public static void main(String[] args) throws Exception {
        RandomAccessFile memoryMappedFile = new RandomAccessFile(fn, "rw");

        //Mapping a file into memory
        MappedByteBuffer out = memoryMappedFile.getChannel().map(FileChannel.MapMode.READ_WRITE, 0,
                mem_map_size);

        //Writing into Memory Mapped File
        for (int i = 0; i < mem_map_size; i++) {
            out.put((byte) 'A');
        }
        System.out.println("File '" + fn + "' is now " + Integer.toString(mem_map_size)
                + " bytes full.");

        // Read from memory-mapped file.
        for (int i = 0; i < 30; i++) {
            System.out.print((char) out.get(i));
        }
        System.out.println("\nReading from memory-mapped file '" + fn + "' is complete.");
    }
}
[/code]

例三的这个模型快速创建一个20M’的文件example_memory_mapped_file.txt，使用字符A填充，然后读取这个文件的前30个字节。在实际应用中，内存映射不仅仅在于原始的I/O加速，而且在于可以将多个不同的读者与写者同时连接到相同的文件镜像。这个技术强大的令人可怕，但如果使用得当则可以实现显著的加速。众所周知的华尔街交易操作，为了得到比竞争者秒、甚至毫秒级的优势，使用内存映射。




## 5 字符编码和搜索




这篇文章最后想介绍的NIO的特征就是字符集，一个在不同的字符编码自检进行转换的包。甚至在NIO之前，Java通过 getBytes方法内建了很多相似的功能。然而字符集非常受欢迎，因为它比getBytes更灵活，在底层也更容易实现，因此产生相当显著的性能。这对于那些对编码、排序和英语之外其它语言特征等非常敏感的搜索意义重大。




 例四展示了一个从Java本地Unicode字符转换到latin-1的例子。



[code lang="java"]String some_string = "This is a string that Java natively stores as Unicode.";
        Charset latin1_charset = Charset.forName("ISO-8859-1");
        CharsetEncoder latin1_encoder = latin1_charset.newEncoder();
        ByteBuffer latin1_bbuf = latin1_encoder.encode(CharBuffer.wrap(some_string));[/code]

[code lang="java"]CharsetEncoder latin1_encoder = latin1_charset.newEncoder();[/code]
本例这行有两个错误，已修正(译者注);注意字符集和通道被设计来一起使用，为了确保程序需要在内存映射、异步I/O、编码转换等地方合作时恰当的执行。



## 总结：当然还有更多


本文的目的是在让Java开发者对NIO和NIO.2的主要功能有一定了解。可以以本文的例子为基础，来深入了解更多的NIO功能;举个例子来说，刚了解的通道的知识可以帮助你在管理文件系统的符号链接时使用NIO的Path功能。查看[参考资源](resources)部分，那里有一个深入了解NIO API的文章列表。

原文链接：[Five ways to maximize Java NIO and NIO.2](http://www.javaworld.com/javaworld/jw-10-2012/121016-maximize-java-nio-and-nio2-for-application-responsiveness.html)



**参考资源**  

                              


                                 


                                    
  * See the [Java 2 SDK Standard Edition (SE) documentation](http://docs.oracle.com/javase/1.4.2/docs/guide/nio/) and [Java SE 7 documentation](http://docs.oracle.com/javase/7/docs/api/java/nio/file/package-summary.html) to learn more about NIO and NIO.2.
                                    

                                    
  * You'll need [JDK 7](http://www.oracle.com/technetwork/java/javase/downloads/java-se-jdk-7-download-432154.html) or greater to access the NIO.2 package APIs.
                                    

                                    
  * "[Master Merlin's new I/O classes](http://www.javaworld.com/jw-09-2001/jw-0907-merlin.html)" (Michael T. Nygard, JavaWorld, September 2001) introduces the `java.nio` package and offers tips for leveraging its nonblocking I/O and memory-mapped buffers.
                                    

                                    
  * See "[Use select for high-speed networking](http://www.javaworld.com/jw-04-2003/jw-0411-select.html)" (Greg Travis, JavaWorld, April 2003) for a high-level overview of the New Input/Output APIs, followed by a demo using selectors
                                       to develop a high-speed server.
                                    

                                    
  * The NIO selector demonstration in this article is modified from one developed by Greg Travis for his article "[Getting started with new I/O (NIO)](http://www.ibm.com/developerworks/java/tutorials/j-nio/section9.html)" (IBM developerWorks, July 2003).
                                    

                                    
  * For more about channels in NIO.2, see "[An NIO.2 primer, Part 1: The asynchronous channel APIs](http://www.ibm.com/developerworks/java/library/j-nio2-1/index.html)" (Catherine Hope and Oliver Deakin, IBM developerWorks, September 2010).
                                    

                                 
                              


                             



                      
