---
layout: post
title: Spring循环引用源码分析
date: 2018-02-09
tags: spring
---
Spring bean 有两种注入方式，构造器注入和 set 注入。后者经过测试循环引用是不会报异常的。前者不确定

### set 的方式

set 的方式默认是支持循环引用的，如果想设置成不支持，可以将 BeanFactory 实现类的 allowCircularReferences 设置成 falses。ApplicationContext 和 BeanFactory 基础的实现类都有这个参数可以设置。

spring 初始化 bean 的时候，是先构造一个空的对象，然后再根据需要去初始化属性指向的 bean。比如 B1 -> B2 为 B1 后属性引用 B2，同时 B2 -> B1。当去初始化 B2 的时候，在设置自己指向 B1 的属性时，是直接用 B1 还未完全构造完成的对象。

比如 B2 在需要去设置对 B1 的引用时，会调用 BeanFactory 的 getBean 方法。这个 getBean 方法的逻辑在最前端会去查一个缓存，这个缓存里放了先前已经开始初始化，但是还没有把属性设置完全的 bean 引用缓存。如果缓存非空，就直接用这个引用返回。如果缓存为空，就去调一个 ObjectFactory 对象的 getObject 方法作为 bean 引用返回。这个 ObjectFactory  是先前 B1 初始化时，被 new 出来以后，加进一个 ObjectFactory   容器的：

```java
// AbstractAutowireCapableBeanFactory.doCreateBean 542行
addSingletonFactory(beanName, new ObjectFactory<Object>() {
	public Object getObject() throws BeansException {
		return getEarlyBeanReference(beanName, mbd, bean);
	}
});
```

addSingletonFactory 其实很简单，只是将 beanName 和 ObjectFactory 对象设置进缓存（一个 ConcurrentHashMap）。这里 getEarlyBeanReference 的逻辑基本就是返回参数里的 bean。参数 bean 其实就是初始化的 B1 的对象实例。

getBean 获取缓存的逻辑：

```java
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