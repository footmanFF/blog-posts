---
layout: post
title: Spring循环引用源码分析
date: 2018-02-09
tags: spring
---
Spring bean 有两种注入方式，构造器注入和 set 注入。后者经过测试循环引用是不会报异常的，前者会报异常。

### set 的方式

set 的方式默认是支持循环引用的，如果想设置成不支持，可以将 BeanFactory 实现类的 allowCircularReferences 设置成 falses。ApplicationContext 和 BeanFactory 基础的实现类都有这个参数可以设置。

spring 初始化 bean 的时候，是先构造一个空的对象，然后再根据需要去初始化属性指向的 bean。比如 B1 -> B2 为 B1 后属性引用 B2，同时 B2 -> B1。当去初始化 B2 的时候，在设置自己指向 B1 的属性时，是直接用 B1 还未完全构造完成的对象。

比如 B2 在需要去设置对 B1 的引用时，会调用 BeanFactory 的 getBean 方法。这个 getBean 方法的逻辑在最前端会去查一个缓存，这个缓存里放了先前已经开始初始化，但是还没有把属性设置完全的 bean 引用缓存。如果缓存非空，就直接用这个引用返回。如果缓存为空，就去调一个 ObjectFactory 对象的 getObject 方法作为 bean 引用返回。这个 ObjectFactory  是先前 B1 初始化时，被 new 出来以后，加进一个 ObjectFactory   容器的：

```java
// AbstractAutowireCapableBeanFactory.doCreateBean
addSingletonFactory(beanName, new ObjectFactory<Object>() {
	public Object getObject() throws BeansException {
		return getEarlyBeanReference(beanName, mbd, bean);
	}
});
```

<!-- more -->

addSingletonFactory 其实很简单，只是将 beanName 和 ObjectFactory 对象设置进缓存（一个 ConcurrentHashMap）。这里 getEarlyBeanReference 的逻辑基本就是返回参数里的 bean。参数 bean 其实就是初始化的 B1 的对象实例。

getBean 获取缓存的逻辑：

```java
// DefaultSingletonBeanRegistry
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
	Object singletonObject = this.singletonObjects.get(beanName);
	if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
		synchronized (this.singletonObjects) {
			singletonObject = this.earlySingletonObjects.get(beanName);
			if (singletonObject == null && allowEarlyReference) {
				ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
				if (singletonFactory != null) {
                    // 关键的在这里
					singletonObject = singletonFactory.getObject(); 
					this.earlySingletonObjects.put(beanName, singletonObject);
					this.singletonFactories.remove(beanName);
				}
			}
		}
	}
	return (singletonObject != NULL_OBJECT ? singletonObject : null);
}
```

显然是最终还是调了 singletonFactory.getObject()。

### 构造器的方式

会报异常：

```
BeanCurrentlyInCreationException: Error creating bean with name 'beanHolder': Requested bean is currently in creation: Is there an unresolvable circular reference
```

整体的 bean 构造方式和 set 方式的差不多，只是构造器的方式在初始化 bean 的时候需要使用指定的构造器，set 的方式使用没有参数的构造器。这种情况下，B1 被构造之前必须要将 B2 构造好，不然 B1 无法实例化（区别于 set 的方式）。B2 如果又依赖 B1 的话，spring 就无法继续运行了，因为连 B1 的实例都没有。

spring 是通过将当前正在初始化，但是没有初始化完全的 bean 的 name 放进一个 Set 里去，出现循环引用的时候回重复去 getBean 相同 name 的 bean，就像上面的 B1 一样。然后就无法通过下面的检查，就抛出异常了：

```java
// DefaultSingletonBeanRegistry
protected void beforeSingletonCreation(String beanName) {
	if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.add(beanName)) {
		throw new BeanCurrentlyInCreationException(beanName);
	}
}
```

