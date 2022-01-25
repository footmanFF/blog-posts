---
layout: post
title: IDEA心得不定期更新
date: 2018-02-25
tags: IDEA
---

记录一些 IDEA 使用中的各种小问题

### 代码格式化

alt + command + L

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

### 避免在格式化代码的时候引入 import 的通配符

![2018-08-03 at 1.38 PM](http://note-1255449501.file.myqcloud.com/2018-08-03-053908.png)

### 关闭拼写校验

![2018-08-04 at 4.36 PM](http://note-1255449501.file.myqcloud.com/2018-08-04-083725.png)

### 重新执行上一次的 debug

先 Command + 5 然后按 Command + R。

### 缩进

```java
step2.setCondition(CombineAndExecuteCondition.of(
    applyAmountRefundableCondition,       // 4个缩进
));

step2.setCondition(CombineAndExecuteCondition.of(
        applyAmountRefundableCondition,   // 8个缩进
));
```

控制这个缩进的配置：

![](http://note-1255449501.file.myqcloud.com/2019-08-15-025949.png)

### 链式调用的缩进控制

比如：

```
super.getFoo().foo().getBar().bar();
```

想缩进成这样：

```
super.getFoo()
     .foo()
     .getBar()
     .bar();
```

![](http://note-1255449501.file.myqcloud.com/2019-09-02-102134.png)

### 如何删除项目

鼠标停在一个项目上，然后按键盘「DELETE」

![](http://note-1255449501.file.myqcloud.com/2019-09-05-060832.png)

### 强制刷 IDEA 本身的各种索引和缓存

有的时候可能全局查找类就是找不到，但是全局查找文件的是可以找到的，提示：

```
Cannot resolve symbol
```

这种情况可能是因为索引没有维护好，可以试试重建 IDEA 本身的索引：

![](http://note-1255449501.file.myqcloud.com/2019-09-05-061634.png)

### CMD + C 复制的时候仅仅复制文本，而不带格式

调整快捷键即可，带样式的复制是有专门的功能的。

![2021-10-03 at 10.06 AM](http://note-1255449501.file.myqcloud.com/2021-10-03-020712.png)
