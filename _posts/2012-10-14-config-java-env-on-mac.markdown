---
author: shupeng.lisp
comments: true
date: 2012-10-14 22:09:57
layout: post
slug: config_java_env_on_mac
title: Mac下配置Java开发环境
wordpress_id: 96
categories: [配置]
---

### 内容概要


本文主要描述在Mac下开发Java程序的环境配置，主要内容包括：



	
  * Java安装与JDK源码配置

	
  * Eclipse配置

	
  * Maven配置

	
  * Git配置

	
  * SVN配置

	
  * JBoss配置

	
  * Httpd安装与配置


  <!--break-->

### 1 Java安装与JDK源码配置





	
  * 下载


不建议直接在mac系统上安装，那样安装会没有jdk源代码，不适合给开发者用。

[官网](https://developer.apple.com/downloads/index.action)下载：https://developer.apple.com/downloads/index.action下查找类似

javadeveloper_for_os_x_2012005__11m3811.dmg的文件。

这里要注意与系统的版本匹配，有些是适合10.6的，有的则是10.8的，查看一下说明。



	
  * 安装


直接安装

	
  * jdk源代码配置


添加软链接

[code lang="shell"]
sudo ln -s /Library/Java/JavaVirtualMachines/1.6.0_35-b10-428.jdk/Contents/Home/src.jar
/System/Library/Frameworks/JavaVM.framework/Home/src.jar
sudo ln -s /Library/Java/JavaVirtualMachines/1.6.0_35-b10-428.jdk/Contents/Home/docs.jar
/System/Library/Frameworks/JavaVM.framework/Home/docs.jar
sudo ln -s /Library/Java/JavaVirtualMachines/1.6.0_35-b10-428.jdk/Contents/Home/appledocs.jar
/System/Library/Frameworks/JavaVM.framework/Home/appledocs.jar
[/code]


### 2 Eclipse配置





	
  * 下载


[官网](http://www.eclipse.org/)：http://www.eclipse.org/ 选择合适的版本



	
  * 安装


直接安装

	
  * 配置


做一下个人配置，如workspace的位置、字体大小等。

添加jdk源文件导入。


### 3 Maven配置


Mac系统自带安装了Maven3，可以在终端运行下命令查看版本：

[code lang="shell"]mvn -v[/code]

如果版本和期望的相同，则不需要重新安装。可选择性的看看settings文件是否需要更改，此文件在$M2_HOME/conf下；

如果期望换个版本，可以到[Maven官网](http://maven.apache.org/)下载合适的版本，解压即可。


### 4 Git配置





	
  * [Mac版本的下载地址](http://code.google.com/p/git-osx-installer/)

	
  * 安装

	
  * 设置ssh

	
    * 检查SSH key:
cd ~/.ssh; 没有则创建一个

	
    * 备份已有的key，（如果有的话）：
mkdir key_backup；
mv id_rsa* key_backup；

	
    * 生成SSH key：
ssh-keygen -t rsa -C mailaddress；
mailaddress为github上的注册邮箱

	
    * 将SSH key添加到GitHub：
登录到GitHub页面，Account Settings->SSH Public Keys->Add another key将生成的key(id_rsa.pub文件）内容copy到输入框中，save。

	
    * 测试链接：ssh -T git@github.com;
You've successfully authenticated, but GitHub does not provide shell access. 表示成功






	
  * 设置个人属性


配置主要也就是配置git的验证信息和一些全局属性。


### 5 SVN配置


系统自带


### 6 Jboss配置


[官方地址](http://www.jboss.org/)

解压


### 7 Apache Httpd配置


[官方地址](http://httpd.apache.org/)下载

安装：

[code lang="shell"]</pre>
gzip -d httpd-NN.tar.gz
tar xvf httpd-NN.tar
cd httpd-NN
./configure --prefix=PREFIX
make
make install
vi PREFIX/conf/httpd.conf
PREFIX/bin/apachectl -k start
<pre>[/code]

在configure阶段可能会报一个Error：

[code lang="shell"]configure: error: C compiler cannot create executables See &lt;code&gt;config.log' for more details.&lt;/code&gt;<br /><br />[/code]

打开config.log，如果有下面内容：

[code lang="shell"]</pre>
configure:4453: checking for gcc
configure:4480: result: /Applications/Xcode.app/Contents/Developer/Toolchains/OSX10.8.xctoolchain/usr/bin/cc
configure:4709: checking for C compiler version
configure:4718: /Applications/Xcode.app/Contents/Developer/Toolchains/OSX10.8.xctoolchain/usr/bin/cc --version >&5
./configure: line 4720: /Applications/Xcode.app/Contents/Developer/Toolchains/OSX10.8.xctoolchain/usr/bin/cc: No such file or directory
configure:4729: $? = 127
configure:4718: /Applications/Xcode.app/Contents/Developer/Toolchains/OSX10.8.xctoolchain/usr/bin/cc -v >&5
./configure: line 4720: /Applications/Xcode.app/Contents/Developer/Toolchains/OSX10.8.xctoolchain/usr/bin/cc: No such file or directory
<pre>[/code]

由于我的大部分命令行工具都是用的xcode统一安装的，怀疑是gcc安装位置有异常，进入到：

[code lang="shell"]cd /Applications/Xcode.app/Contents/Developer/Toolchains/
ls[/code]

发现该目录下只有：
`XcodeDefault.xctoolchain
`
进去发现有usr/bin/cc文件，因此只需要做一个软链接就ok：

[code lang="shell"]sudo ln -s XcodeDefault.xctoolchain OSX10.8.xctoolchain[/code]

再次configure，make与make  install。


### 8 总结


本文主要介绍了在Mac下做Java开发所需的常用工具的安装与配置，而上述工具之间的整合内容没有涉及，这是由于一方面这方面的插件比较多，大家对此褒贬不一；其次很多人不喜爱这样的插件，因此此部分也就没有做介绍，但具有了上述基本工具，满足日常开发需求已经不成问题了。
