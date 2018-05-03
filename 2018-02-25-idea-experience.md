---
layout: post
title: IDEA心得不定期更新
date: 2018-02-25
tags: IDEA
---

记录一些 IDEA 使用中的各种小问题

### debug step in 失效

红框内的打钩去掉就OK了，看源码的时候勾上了真是蛋疼

![](http://note-1255449501.file.myqcloud.com/2018-02-25-070051.png)

另外，如果 debug 的代码是一个非 Main 的线程，但是如果 debug 到一半 Main 方法结束，即 JVM 运行结束，会导致 step in 按下以后直接结束掉。这个时候像个办法让 Main 方法永远不结束就行了。 

<!-- more -->

### Maven 模式运行

Run - Edit Configrations - 新增一个Maven的运行方式。

![](http://note-1255449501.file.myqcloud.com/2018-03-22-080451.png)

### Maven 自动下载源码

![](http://note-1255449501.file.myqcloud.com/2018-04-26-021913.png)