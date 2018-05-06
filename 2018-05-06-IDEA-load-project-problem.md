---
layout: post
title: IDEA load project 慢问题排查
date: 2018-05-06
tags: IDEA
---

去打开 IDEA 的 DEBUG 日志，日志地址在 help - Show log in finder 里，日志配置文件在 /Applications/IntelliJ IDEA.app/Contents/bin/log.xml。启动时的日志有一段明显很值得怀疑：

```
2018-05-06 15:34:45,576 [  93768]   INFO - ij.components.ComponentManager - com.seventh7.mybatis.ref.CmProject initialized in 75086 ms
2018-05-06 15:34:45,588 [  93780]   INFO - ellij.project.impl.ProjectImpl - 152 project components initialized in 75849 ms
```

com.seventh7.mybatis.ref.CmProject 初始化耗了 75 秒。去把 mybatis 插件禁用以后重新测试：

```
2018-05-06 15:40:13,639 [ 138286]   INFO - ellij.project.impl.ProjectImpl - 150 project components initialized in 999 ms
```

<!-- more -->

禁用以后明显解决了，这个插件记得是之前安装的破解的 mybatis 插件。重新去 idea 插件仓库里搜，现在又了一个 Free MyBatis Plugin，不用再找破解版的了：

![](http://note-1255449501.file.myqcloud.com/2018-05-06-075848.png) 



