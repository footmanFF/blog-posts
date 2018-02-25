---
layout: post
title: IDEA使用心得
date: 2018-02-25
tags: IDEA
---

记录一些 IDEA 使用中的各种小问题

### debug step in 失效

红框内的打钩去掉就OK了，看源码的时候勾上了真实蛋疼

![](http://note-1255449501.file.myqcloud.com/2018-02-25-070051.png)

另外，如果 debug 的代码是一个非 Main 的线程，但是如果 debug 到一半 Main 方法结束，即 JVM 运行结束，会导致 step in 按下以后直接结束掉。这个时候像个办法让 Main 方法永远不结束就行了。 

<!-- more -->