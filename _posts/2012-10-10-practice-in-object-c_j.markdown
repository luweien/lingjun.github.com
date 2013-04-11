---
author: shupeng.lisp
comments: true
date: 2012-10-10 23:11:57
layout: post
slug: practice_in_object_c_j
title: Object-c实践之路:第一章 起步
wordpress_id: 86
categories:
- 心得
---

#### 摘要


主要介绍object-c的准备工作，包括xcode下载、安装、检查与配置。并对基本的object-c语法做了一个介绍。

写在前面

Object-c已经有近30年历史了，而随着Mac OS X系统的广受欢迎，作为Mac OS X核心的object-c重新受到广大编程爱好者的喜爱，本系列就将object-c的实践之路逐步分享。

0 下载与安装

任何一门语言都有它的使用范围和生存土壤，编程语言也一样。如果想具有一个比较舒适的编程学习环境，将关注点更多的投入到语言的学习上，拥有一个稳定的系统配置是比较关键的。因此，假设具有一台安装有Mac OS X系统的稳定环境。

作为Apple的多年工具，Apple的XCode应该是学习object-c的比较好的选择，安装与配置XCode非常简单，[官方下载](https://developer.apple.com/xcode/)到本地，选择安装，等待安装结束。

注：本文所使用版本4.5.1

<!--break-->

1 检查与配置

检查安装是否正确，最好的方式就是建个项目瞧一下：

运行XCode，选择:

新建OS X--->Application--->Command Line Tool

--->next--->输入工程名HelloWorld

--->下一步--->创建；

创建成功之后进入视窗模式，点击右边的main.m，点击Run（或者使用快捷键Command+R，或者选择Product下面的run），都会告诉你build successed！

，并且在屏幕下方会输出Hello，World! 表示安装完全成功。

这时候，就可以选择对XCode做一些配置，暂时不设置也没关系，可以慢慢熟悉。

由于本人对命令行有一定喜爱，所以在XCode的偏好设置里--->download部分，选择安装了Command Line Tools，这个工具会自动安装很多命令行工具，如make工具等。

2 第一个程序

在第一步的检查过程中，其实已经运行了第一个程序了，那就是XCode自动生成的Hello World！，这差不多是所以编程语言的第一个示例程序了。生成的代码如下：

[code lang="c"]
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[])
{
@autoreleasepool {
// insert code here...
NSLog(@"Hello, World!");
}

return 0;
}
[/code]

抛开其余部分不管，其实就是一句简单的Hello World！输出。

3 分析与术语
有了上面这个自动生成的例子，就可以开始学习object-c中的一些术语与惯例了。



	
  * object-c文件以.m结尾，代表message，指的是object-c的一个主要特性，称之为.m文件；

	
  * c语言使用#include语句通知编译器应在头文件中查询定义。在object-c中也可以这样，但使用#import可保证头文件只被包含一次，而不论此命令实际出现了多少次。

	
  * 可以使用printf，但建议使用NSLog，因为它添加了特性，例如时间戳、日期戳和自动附加换行符等。

	
  * Cocoa对其所有函数、常量和类型名称都添加了“NS”前缀。

	
  * @符号是object-c在标准c语言基础上添加的特性之一，双引号的字符串前有一个@符号，表示引用的字符串应该作为Cocoa的NSString元素来处理。

	
  * NSString元素有很多打包的特性，Cocoa在需要字符串时可随时使用他们。下面是一些功能：

	
    * 告知其长度；

	
    * 将自身与其他字符串比较；

	
    * 将自身转换为整型值或浮点值；





这些术语与知识，差不多就可以对object-c的Hello World程序有了第一步了解了。

4 小结
本文主要介绍了object-c的环境配置，安装xcode及自己创建第一个object-c的项目。运行成功后，介绍了一些与该例子相关的知识。相信通过这些知识，对object-c有了个初步了解，接下来就可以进入到真正的object-c的旅程了。
