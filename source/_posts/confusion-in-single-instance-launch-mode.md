---
title: 记singleInstance的一个坑
date: 2016-11-06 03:17:40
tags: [Android]
---

之前曾经写过一个Demo，遇到了一个奇怪的行为，今晚查找launchMode相关资料的时候竟然以外得到了原因，急忙记录下来。

先说说行为。这个Demo有3个Activity：Main、A、B。Main的launchMode设置为`singleInstance`，上有两个按钮可以分别启动A和B。

除此以外，还有一个Widget，点击之后会进入Main。

接下来我的操作：打开Main，点击进入A，按Home，点击Widget回到Main，点击进入B，按Home，点击Widget回到Main，点击试图进入A，谁知打开来竟然是B！

今晚无意中找到这样一个帖子：[android - How to start an activity from a singleInstance activity? - Stack Overflow](http://stackoverflow.com/questions/5804966/how-to-start-an-activity-from-a-singleinstance-activity)，发现了他摘录的官方文档，终于明白：

原来，singleInstance的Activity是它所在的唯一一个Activity，因此从这里打开的Activity会在新的Task中打开，**就像使用了FLAG_ACTIVITY_NEW_TASK一样**。如果去翻翻这个flag就会发现，它会查找是否被启动的已经存在，如果存在，它会切换到这个task但是并不会自动pop上层的activity！

继续翻阅就会发现，想要无条件创建task的话，需要增加`FLAG_ACTIVITY_MULTIPLE_TASK`标记，经过测试，的确可以了。

但是后来想想，这样似乎也有问题，按照文档：

>  Used in conjunction with `FLAG_ACTIVITY_NEW_TASK` to disable the behavior of bringing an existing task to the foreground. When set, a new task is *always* started to host the Activity for the Intent, regardless of whether there is already an existing task running the same thing.

也就是说，使用这个会不停地创建新的task。

我循环进入Main - 点击进入A - Home - 点击Widget进入Main，这样几次之后，用`dumpsys`查看了app的所有activity，果然已经有五六个A的task了……

最后该思考一下，对于一个需要返回Main的Widget或Notification，应该如何设计跳转呢？