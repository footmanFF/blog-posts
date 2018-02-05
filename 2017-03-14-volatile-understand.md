---
layout: post
title: volatile理解
date: 2017-03-14
tags: Java
---
### 看一段代码
```java
class VolatileExample {
  int a = 0;
  volatile boolean flag = false;
  public void writer() {
      a = 1;                   //1
      flag = true;               //2
  }
  public void reader() {
      if (flag) {                //3
          int i =  a;           //4
      }
  }
}
```
volatile的作用是让一个变量是多线程可见的，并且对他的 get 和 set 是原子的。着重考虑前一点，后一点好理解。

<!-- more -->


那么有个问题。在上面的代码里，如果线程A执行writer，线程B并行执行read，变量flag因为有volatile所以是多线程可见的，但是变量a是可见的吗？有没有可能，reader方法最终i被设置为0，因为没有a变量不是「可见的」？。

这个文章有讲这个问题 <http://www.infoq.com/cn/articles/java-memory-model-4>

>当写一个volatile变量时，JMM会把该线程对应的本地内存中的共享变量刷新到主内存。
>当读一个volatile变量时，JMM会把该线程对应的本地内存置为无效。线程接下来将从主内存中读取共享变量。

如上说法，那么应该是线程A在写flag时，同时让当前线程调用的方法所能使用到的变量（局部变量、类变量）都刷到主内存。当B读flag时，同时让当前线程调用的方法所能使用到的变量（局部变量、类变量）在本地内存失效，重新去主内存中读取。
以上就等价于让所有的变量都变成了多线程可见的，就如volatile是多线程可见的一样。

>下面对volatile写和volatile读的内存语义做个总结：
>    1. 线程A写一个volatile变量，实质上是线程A向接下来将要读这个volatile变量的某个线程发出了（其对共享变量所在修改的）消息。
>    2. 线程B读一个volatile变量，实质上是线程B接收了之前某个线程发出的（在写这个volatile变量之前对共享变量所做修改的）消息。
>    3. 线程A写一个volatile变量，随后线程B读这个volatile变量，这个过程实质上是线程A通过主内存向线程B发送消息。

看起来就是那么回事，volatile的作用的可见性更体现在：volatile的写和读取发出了一个「通知-接受」机制，通知其他线程是否需要重新reload当前线程的本地内存。

另外一点，volatile 变量是不能做递增的：

![](http://note-1255449501.file.myqcloud.com/2017-12-15-061815.png)