---
layout: post
title: state machine一些概念理解
date: 2019-04-14
tags: state machine
---

#### 状态机是怎么定义，或者说是怎么抽象的？

- state 状态
- hierarchical state 子状态，具层级关系的状态
- region 域，区域
- transition 过渡，分三种类型，external、internal、local
- guard 守卫，相当于一种拦截，在某一个操作前设置一个 guard，用 guard 来校验到底是否执行后面的操作
- action 一次业务逻辑执行，执行参数 StateContext
- StateContext 状态上下文

<!-- more -->

#### state 的一些概念

**普通状态：**一个不带有子状态的状态。

**组合状态：**比如状态 A 下有嵌套的状态 A1，假如当前状态处在这个子状态 A1 上，那么 A 和 A1 和一起可以看做是一个子状态机，这个子状态机的状态就称为「组合状态」，这个例子的组合状态就是 「A - A1」。

> [这个资料里有一段描述](https://www.timsrc.com/article/22.html)：
> 如果我们从另外一个角度看待状态的内部，我们会发现，状态之中还会嵌套状态，这种被嵌套的状态被称为`子状态`，而嵌套子状态的状态称为`组合状态`

**子状态：**状态可以嵌套，嵌套在其他状态下的状态就称为「子状态」。

#### transition 的三种类型 

**external:** 外部的转换，「外」是相当于 source state 说的，一次转换的 target state 和 source target 不一样，状态流转到新的状态，其中历经了 exit 和 entry。

**internal:** Internal transition is used when action needs to be executed without causing a state transition. With internal transition source and target state is always a same and it is identical with self-transition in the absence of state entry and exit actions.

内部的转换，「内」是相当于 source state 说的，Internal transition 的 source state 和 target state 一样，按上面最后一句的描述，是不会经过 exit 和 entry，那么也应该不会执行 exit action 和 entry action。

**local:** Local transition doesn’t cause exit and entry to source state if target state is a substate of a source state. Other way around, local transition doesn’t cause exit and entry to target state if target is a superstate of a source state.

#### 一个概念：正交

正交主要用来形容 region (orthogonal region)。当进入一个组合状态以后，组合状态下有多个「区域」，每个区域定义了一些状态和转换，并且这些个区域内的状态流转都互相不影响时，那么这些区域就是正交的。相当于每个区域内的状态流转并行运行。但是，无论如何并行运行，从组合状态出来的时候必须汇聚成一个转换，即从组合状态出来的时候，只允许有一个 target。

> [这个资料里有一段描述](https://www.timsrc.com/article/22.html)：
> 在一个组合状态中有两个正交区域，当对象进入组合状态中，控制流分岔为两个并发流，等到这些并发流都达到最终状态或者有一个离开组合状态的显示的转移，那么这些并发流就重新汇合为一个流。所以，一个对象在一个时间可能同时处于多个正交子状态，而只能处于一个非正交子状态；即正交子状态是并发的，非正交子状态是顺序的。

spring-statemachine 正交区域的配置：[11.4 Configuring Regions](https://docs.spring.io/spring-statemachine/docs/2.0.3.RELEASE/reference/htmlsingle/#configuring-regions)。

```java
states
    .withStates()
    .initial(States2.S1)
    .state(States2.S2)
    .and()
    	.withStates()
    	.parent(States2.S2)
    	.initial(States2.S2I)
    	.state(States2.S21)
    	.end(States2.S2F)
    .and()
    	.withStates()
    	.parent(States2.S2)
    	.initial(States2.S3I)
    	.state(States2.S31)
    	.end(States2.S3F);
```

对同一个状态创建两个区域，每个区域都有相同的初始状态，那么这两个区域就成为「正交」的了。

#### 资料

https://docs.spring.io/spring-statemachine/docs/2.0.3.RELEASE/reference/htmlsingle/
https://www.ibm.com/support/knowledgecenter/SS8PJ7_8.5.1/com.ibm.xtools.modeler.doc/topics/cstate.html
https://www.timsrc.com/article/22.html