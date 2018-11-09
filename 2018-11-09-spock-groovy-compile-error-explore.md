---
layout: post
title: spock接入编译报错问题
date: 2018-11-09
tags: spock, groovy
---

最近在用 spock，但是搭环境时候碰到一个依赖问题，这里记录下解决过程：

为了使用最新版的特性，引入的 spock 的核心依赖是这样的：

```xml
            <dependency>
                <groupId>org.spockframework</groupId>
                <artifactId>spock-core</artifactId>
                <version>1.2-groovy-2.5</version>
            </dependency>

            <dependency>
                <groupId>org.spockframework</groupId>
                <artifactId>spock-spring</artifactId>
                <version>1.2-groovy-2.5</version>
                <scope>test</scope>
            </dependency>
```

<!-- more -->

使用的 Spring-boot 版本是这样的：

```xml
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.5.13.RELEASE</version>
```

这个时候，编译会报错，错误信息是这样的：

![2018-11-09 at 11.13 AM](http://note-1255449501.file.myqcloud.com/2018-11-09-031350.png)

org/codehaus/groovy/ast/MethodCallTransformation 这个类没有，网上找了一圈，这个类来自这个依赖：

```xml
            <dependency>
                <groupId>org.codehaus.groovy</groupId>
                <artifactId>groovy</artifactId>
                <version>2.5.0</version>
            </dependency>
```

版本至少是 2.5 以上会包含 MethodCallTransformation 这个类。但是我的例子虽然引入了 spock 是 1.2-groovy-2.5 的版本，但是自动依赖拉取的 groovy 相关报的版本确是 2.4.15 的：

```
[INFO] +- org.spockframework:spock-core:jar:1.2-groovy-2.5:test
[INFO] |  +- org.codehaus.groovy:groovy:jar:2.4.15:test
[INFO] |  +- org.codehaus.groovy:groovy-json:jar:2.4.15:test
[INFO] |  +- org.codehaus.groovy:groovy-nio:jar:2.4.15:test
[INFO] |  +- org.codehaus.groovy:groovy-macro:jar:2.5.2:test
[INFO] |  +- org.codehaus.groovy:groovy-templates:jar:2.4.15:test
[INFO] |  +- org.codehaus.groovy:groovy-test:jar:2.4.15:test
[INFO] |  +- org.codehaus.groovy:groovy-sql:jar:2.4.15:test
[INFO] |  \- org.codehaus.groovy:groovy-xml:jar:2.4.15:test
```

这里就是关键点了，为什么我引用了「spock-core:jar:1.2-groovy-2.5」，但是还是给我自动依赖了 2.4.X 版本的 groovy，网上找了圈资料，这个 github 的 issure 和这个问题一样：https://github.com/spring-projects/spring-boot/issues/13444

核心原因是 1.X 版本的设定了 groovy.version 版本为 2.4.15，这个可以通过点开 spring-boot-starter-parent 的父 pom 确认：

spring-boot-dependencies-1.5.13.RELEASE.pom

```xml
<properties>
    <groovy.version>2.4.15</groovy.version>
</properties>

<dependency>
    <groupId>org.codehaus.groovy</groupId>
    <artifactId>groovy</artifactId>
    <version>${groovy.version}</version>
</dependency>
```

相当于父级 pom 指定了特定依赖的版本，解决的办法很简单，自己在本项目内指定出来就行了，maven 的执行方式是这样：

```xml
    <properties>
        <groovy.version>2.5.3</groovy.version>
    </properties>
```

至此，问题解决。

另外，如果去将 spring-boot 升到 2.X ，那么默认的 groovy 版本就是 2.5 以上的，这也是为什么自己搭的 demo 项目没有这个编译报错，而公司项目却有问题，原因就是自己搭的 demo 用的 spring-boot 是 2.X 版本的。