---
layout: post
category : technology
tags : [Jmeter , Chrome , doc ]
title: JMeter 进行压力测试
date : 2015-06-21 13:44:20
---


##测试脚本录制
-------------

使用测试脚本可以记录用户的操作。实际进行测试时，我们通过重复执行该脚本模拟用户的压力。

1. 打开 [JMeter](http://jmeter.apache.org/)

2. 在 WorkBench 中，建立一个 HTTP(S) Test Script Recorder

3. 在 Test plan content > Grouping 中，选择 Put each group in a new controller
 
4. 点击 Start 按钮，这样会启动一个用于录制脚本的代理服务器，默认端口是 8080

5. 打开 Chrome Developer tools，打开准备测试的页面

6. 在 [Proxy SwitchyOmega](https://github.com/FelisCatus/SwitchyOmega) 中选择 JMeter 的代理服务器

7. 清空缓存并重新加载页面
  
8. 执行要测试的动作

9. 当 Chrome 完成全部加载后，停止代理服务器

这样测试脚本就录制完毕了。之后可以把录制好的 Controller 放到 Thread Group 之中进行压力测试。


<!--more-->

##设置压力
-------------

压力来源于用户的重复访问行为。我们用线程模拟来自用户的网络请求。

在 Test Plan 中建立线程组，用于控制线程的数量。


##选取关注的报告
-------------

我们主要关注

* Graph Results
* Summary Report

##执行测试
-------------

点击绿色播放按钮开始测试。

在测试之前，根据需要清理之前测试的结果。点击齿轮扫帚进行清理。