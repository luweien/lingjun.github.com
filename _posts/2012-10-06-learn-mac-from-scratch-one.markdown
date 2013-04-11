---
author: shupeng.lisp
comments: true
date: 2012-10-06 21:24:16
layout: post
slug: learn_mac_from_scratch_one
title: Mac新手入门配置之道
wordpress_id: 72
categories: [配置]
tags:
- mac
- 新手配置
---

Mac以其良好的用户体验为广大果粉所称颂，而对于初次接触mac系统的来说，前面几步该做什么呢？下面就以一个新手的角度来分享下吧。

本机系统Mac OS X 10.8.2
本文包括内容：


#### 0 心态




#### 1 用户




#### 2 启用功能键(F1-F12)




#### 3 终端配置




#### 4 HOME、END、DELETE、PAGEUP、PAGEDOWN配置




#### 5 Safari全屏




#### 6 触摸板使用与设置




#### 7 结语


 <!--break-->

#### 0 心态


心态要好，要平和。要非常好，要非常平和。
在我们接触到一个新东西时，总会有各种不适应，这时候，我们就需要将心态调整好,相信问题一定是可以解决的，慢慢来。


#### 1 用户


任何一个操作系统，对用户的管理都是很关键的，同样，我们对一个系统的用户也需要做一定的配置。



	
  * 配置用户


打开系统偏好设置--->System下的User&Group--->有增加(+)，删除(-)，配置(齿轮)三个选项，可以做你想要的配置。

如果是灰色的，先把最下面那把锁解开，修改完毕之后锁上。

	
  * 配置主机名


如果对Unix系统有所了解的话，这里说的就是hostname，可以在终端下输入hostname查看。要更改主机名，只需要在终端下输入下述命令：

[code lang="shell"]＃ sudo scutil --set HostName 新的主机名[/code]


#### 2 启用功能键(F1-F12)


默认状态下，功能键是关闭的，按F1-F12显示的是诸如亮度、声音等功能，而常用的如F2表示重命名、F5表示刷新等都没有开启，这时候只需要稍微更改下配置就可以使用了。具体方法：

系统偏好设置--->键盘（Keyboard）选项卡--->Use all F1, F2, etc. keys as standard function keys，那个方框打勾就行，即将F1到F12作为基本功能键。


#### 3 终端配置


作为Unix系统，终端的使用是必不可少的，但我们在终端输入ls的时候，发现目录和文件是没有颜色区分表示的，这显得不是很友好，试着输入ls -G，发现有了区别。

每次这样输入会显得非常麻烦，好在终端具有别名（alias），可以将此写入到配置文件，就一劳永逸了：



	
  * 在~/.bash_profile（如果没有则创建一个）中增加一行


[code lang="shell"]alias ls="ls -G"[/code]

``,退出终端再进入可以发现生效了；



	
  * 上面方案可以显示，但不能自己配置颜色，如果期望自己定义颜色的话，可以如下设置：
在~/.bash_profile中增加如下两行,上面那个别名需要注释或者删掉；


[code lang="shell"]export CLICOLOR=1
export LSCOLORS=gxfxaxdxcxegedabagacad[/code]

``退出终端再进入。

~/.bash_ alias ls=”ls -G”是给”ls -G”起了一个别名，当执行ls时，就相当于执行了ls -G。CLICOLOR是用来设置是否进行颜色的显示。CLI是Command Line Interface的缩写。

LSCOLORS是用来设置当CLICOLOR被启用后，各种文件类型的颜色。LSCOLORS的值中每两个字母为一组，分别设置某个文件类型的文字颜色和背景颜色。


#### 4 HOME、END、DELETE、PAGEUP、PAGEDOWN配置



Mac的键盘没有Home, End, Page UP, Page DOWN这几个键？但可以通过用Fn键来组合得到同样的功能：



	
  * Home键=Fn+左方向

	
  * End键=Fn+右方向

	
  * PageUP=Fn+上方向

	
  * PageDOWN=Fn+下方向

	
  * 向前Delete=Fn+delete键


苹果笔记本Fn键功能很有用


##### 终端的HOME、END配置


做了上述配置之后，发现终端里Home、End还是没有预期效果，需要做如下调整：



	
  * 打开 Terminal => Preferences => Keyboard,

	
  * 在列表里找到 home, 双击编辑, 在弹出的窗口里把 Action 改成 “Send string to shell:”, 然后把光标移底下的文本框中, 按下 CTRL + A, 点 OK 确定.

	
  *  在列表里找到 end, 用同样的方法修改, 不过在文本框里按 CTRL+E, , 点 OK 确定.


现在在命令行里试试看, 确认一下 Home End 键的行为是不是变成跳到行首和行尾了。


#### 5 Safari全屏


在其它系统下很习惯全屏浏览器，而在Mac默认不是全屏的，要想先期做适应，可以试着用一下：

control+command+f


#### 6 触摸板使用与设置


在系统偏好设置--->TrackPad中观察并学习，我没有更改，只是把Point&Click中的第一项也勾选上了，这样就不需要按下去了，在触摸屏上碰一下就好了。


#### 7 结语


好了，基本配置就到这里了，经过这这几步配置，基本的操作已经OK了，下一步将进入常用软件安装与配置了。希望在Mac之路上体验越来越愉快。
