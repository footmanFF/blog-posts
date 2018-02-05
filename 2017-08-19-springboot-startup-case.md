---
layout: post
title: SpringBoot无法启动问题排查
date: 2017-08-19
tags: Java,
---

## 背景

在维护的项目使用了 SpringBoot，常规本地运行都是直接用 main 方法去跑。有一个需求需要在 Spring 容器启动前增加一些操作，因为测试和线上环境其实都是打 war 包去部署到 tomcat 中去的，和 main 方法稍微有些不同。所以就去尝试本地也用 tomcat war 包部署的方式去启动，但是碰到了一个麻烦的问题。在运行的时候会运行很长时间，而且进程占满了一个 cpu。

<!-- more -->

## 解决

包依赖包的日志级别调整的 debug

```Xml
<logger name="org" level="debug" />
```

然后在跑，能看到非常多的 debug 信息，启动以后很长的一段时间 debug 信息都在疯狂的刷，找到一些错误信息（这里出现了明显的不可逆错误居然没有往外抛异常也是醉了，还在让程序瞎跑）

```
Error parsing Mapper XML. Cause: java.lang.IllegalArgumentException: Result Maps collection already contains value for com.weidai.cf.rdc.dao.ApiEventConfigMapper.BaseResultMap
```

这个是 MyBatis 在解析 Mapper 配置的时候报错。另外还看到这个信息。

```
Find JAR URL: jar:file:/Users/zhangli/Work/idea_workspace/******/rule-data-center-webapp/target/*****/WEB-INF/lib/******-dao-2.0.3-SNAPSHOT.jar!/com/******/dao/handler
```

之前有从 SpringBoot 的启动的地方一步一步 debug 进去看过，最终发现其实是在 scan compotent 的地方运行进去就出不来了。这条信息说明是扫描到了匹配的 jar 包并准备处理。但是这个 jar 包的版本是旧的，记得已经把 SNAPSHOT 去掉了，那为什么还能扫描到这个版本？然后去检查部署的 war 包。

```
*****-dao-2.0.3-SNAPSHOT.jar
*****-dao-2.0.4.jar
```

在 IDEA 的 target 目录下的 lib 文件夹能找到这两个 jar 包。问题很明显是重复的 jar 包导致的 MyBatis 初始化失败，然而 MyBatis  初始化出现错误居然不往外抛异常也是醉了，还什么 error 信息也不打印。解决办法很简单，mvn clean，然后重新打包就可以了。

但是为什么会出现多个版本包呢？去重试在 IDEA 内切分支到旧的分支里去，在旧分支打包，最终发现打包以后，target 的 lib 包内能重现出重复的 jar 包。所以 IDEA 在自动打包时候，是不会自己去做 clean 的（应该有什么地方进行配置）。

但是为什么用 main 方法执行就又可以呢？那是因为 main 执行 classpath 不是使用的打包结果，当切换分支以后，IDEA 回去刷新依赖，这个时候是不会出现重复的 jar 的，从 IDEA 的 Maven Projects 视图的 dependencies 标签里就能看到。

## 总结

框架运行处问题的时候去打开 debug 日志很有用。