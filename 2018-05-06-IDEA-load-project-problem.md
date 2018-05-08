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

com.seventh7.mybatis.ref.CmProject 初始化耗了 75 秒。去把 MyBatis 插件禁用以后重新测试：

```
2018-05-06 15:40:13,639 [ 138286]   INFO - ellij.project.impl.ProjectImpl - 150 project components initialized in 999 ms
```

<!-- more -->

禁用以后明显解决了，这个插件记得是之前安装的破解的 MyBatis 插件。重新去 idea 插件仓库里搜，现在有了一个 Free MyBatis Plugin，不用再找破解版的了：

![](http://note-1255449501.file.myqcloud.com/2018-05-06-075848.png) 

后来又出现了新的问题，启动的时候有两次 SocketException: Operation timed out。观察 INFO 日志的时候，这两个异常日志一出现，其他的启动日志就立刻打出来，IDEA 也顺利打开了：

```
2018-05-07 09:30:56,988 [ 116724]   INFO - cloudConfig.CloudConfigManager - Operation timed out (Read failed)
java.net.SocketException: Operation timed out (Read failed)
	at java.net.SocketInputStream.socketRead0(Native Method)
	at java.net.SocketInputStream.socketRead(SocketInputStream.java:116)
	at java.net.SocketInputStream.read(SocketInputStream.java:171)
	at java.net.SocketInputStream.read(SocketInputStream.java:141)
	at sun.security.ssl.InputRecord.readFully(InputRecord.java:465)
	at sun.security.ssl.InputRecord.read(InputRecord.java:503)
	at sun.security.ssl.SSLSocketImpl.readRecord(SSLSocketImpl.java:983)
	at sun.security.ssl.SSLSocketImpl.performInitialHandshake(SSLSocketImpl.java:1385)
	at sun.security.ssl.SSLSocketImpl.startHandshake(SSLSocketImpl.java:1413)
	at sun.security.ssl.SSLSocketImpl.startHandshake(SSLSocketImpl.java:1397)
	at sun.net.www.protocol.https.HttpsClient.afterConnect(HttpsClient.java:559)
	at sun.net.www.protocol.https.AbstractDelegateHttpsURLConnection.connect(AbstractDelegateHttpsURLConnection.java:185)
	at sun.net.www.protocol.http.HttpURLConnection.getInputStream0(HttpURLConnection.java:1546)
	at sun.net.www.protocol.http.HttpURLConnection.getInputStream(HttpURLConnection.java:1474)
	at java.net.HttpURLConnection.getResponseCode(HttpURLConnection.java:480)
	at sun.net.www.protocol.https.HttpsURLConnectionImpl.getResponseCode(HttpsURLConnectionImpl.java:338)
	at com.intellij.util.io.HttpRequests.openConnection(HttpRequests.java:513)
	at com.intellij.util.io.HttpRequests.access$300(HttpRequests.java:63)
	at com.intellij.util.io.HttpRequests$RequestImpl.getConnection(HttpRequests.java:292)
	at com.intellij.ide.plugins.RepositoryHelper.lambda$loadPlugins$1(RepositoryHelper.java:156)
	at com.intellij.util.io.HttpRequests.lambda$doProcess$0(HttpRequests.java:418)
	at com.intellij.util.net.ssl.CertificateManager.runWithUntrustedCertificateStrategy(CertificateManager.java:335)
	at com.intellij.util.io.HttpRequests.doProcess(HttpRequests.java:418)
	at com.intellij.util.io.HttpRequests.process(HttpRequests.java:398)
	at com.intellij.util.io.HttpRequests.access$100(HttpRequests.java:63)
	at com.intellij.util.io.HttpRequests$RequestBuilderImpl.connect(HttpRequests.java:266)
	at com.intellij.ide.plugins.RepositoryHelper.loadPlugins(RepositoryHelper.java:151)
	at com.intellij.ide.plugins.RepositoryHelper.loadPlugins(RepositoryHelper.java:106)
	at com.intellij.ide.plugins.RepositoryHelper.loadPlugins(RepositoryHelper.java:98)
	at com.intellij.ide.plugins.RepositoryHelper.loadPlugins(RepositoryHelper.java:93)
	at com.intellij.cloudConfig.CloudConfigManager.lambda$getRepositoryPlugins$31(CloudConfigManager.java:1334)
	at com.intellij.openapi.application.impl.ApplicationImpl$2.call(ApplicationImpl.java:328)
	at java.util.concurrent.FutureTask.run(FutureTask.java:266)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
	at java.lang.Thread.run(Thread.java:745)
```

![](http://note-1255449501.file.myqcloud.com/2018-05-07-015157.png)

抓包看了下，两个超时请求分别请求的是 googleapi 和 client.google.com，连接不上。