---
layout: post
title: Spring IOC循环引用源码分析
date: 2018-02-09
tags: spring
---
Spring bean 有两种注入方式，构造器注入和 set 注入。后者经过测试循环引用是不会报异常的，前者会报异常。

### set 的方式

set 的方式默认是支持循环引用的，如果想设置成不支持，可以将 BeanFactory 实现类的 allowCircularReferences 设置成 falses。ApplicationContext 和 BeanFactory 基础的实现类都有这个参数可以设置。

spring 初始化 bean 的时候，是先构造一个空的对象，然后再根据需要去初始化属性指向的 bean。比如 B1 -> B2 为 B1 后属性引用 B2，同时 B2 -> B1。当去初始化 B2 的时候，在设置自己指向 B1 的属性时，是直接用 B1 还未完全构造完成的对象。

比如 B2 在需要去设置对 B1 的引用时，会调用 BeanFactory 的 getBean 方法。这个 getBean 方法的逻辑在最前端会去查一个缓存，这个缓存里放了先前已经开始初始化，但是还没有把属性设置完全的 bean 引用缓存。如果缓存非空，就直接用这个引用返回。如果缓存为空，就去调一个 ObjectFactory 对象的 getObject 方法作为 bean 引用返回。这个 ObjectFactory 是先前 B1 初始化时，被 new 出来以后，加进一个 ObjectFactory 容器的：

```java
// AbstractAutowireCapableBeanFactory.doCreateBean
addSingletonFactory(beanName, new ObjectFactory<Object>() {
    public Object getObject() throws BeansException {
        return getEarlyBeanReference(beanName, mbd, bean);
    }
});
```

<!-- more -->

addSingletonFactory 其实很简单，只是将 beanName 和 ObjectFactory 对象设置进缓存（一个 ConcurrentHashMap）。这里 getEarlyBeanReference 的逻辑基本就是返回参数里的 bean。参数 bean 其实就是初始化的 B1 的对象实例，这个 B1 实例这个时候还是处在初始化中的。

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

#### 2021-01-05 补充

到上述描述的为止，是能解决循环引用问题的。核心的原理其实就是给 B2 注入的 B1 是一个初始化中的对象引用。但是引入另外一个机制会出现问题。

BeanPostProcessor 的 postProcessAfterInitialization，执行的时间点是在 populateBean bean 以后。按照上面的例子，B2 在设置了 B1 的引用以后（这个引用是 B1 的原始对象），如果后面 B1 后面有一个 BeanPostProcessor  的 postProcessAfterInitialization 将 B1 的 bean 对象实例完全替换为另一个对象，那么就有问题了。注入给 B2 的是原始的 B1 对象，但是其实因为 BeanPostProcessor 的存在又把实际的 B1 的 bean 对象换成了另一个对象，此时就有问题了。

BeanPostProcessor 的 postProcessAfterInitialization 方法是可以替换 bean 对象的：

```java
public interface BeanPostProcessor {
  Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;
}
```

initializeBean 的逻辑可以看出，bpp 的 postProcessAfterInitialization 的返回，可以替换原始的 bean 对象。

```java
	protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
		// ...
		if (mbd == null || !mbd.isSynthetic()) {
			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
		}
		return wrappedBean;
	}
```

上述这个问题，典型的场景就是 AOP。AOP 是通过 BeanPostProcessor  的 postProcessAfterInitialization  来完成对象的代理的。

那么问题来了，这个问题如何解决？

先看 ObjectFactory 的 getObject 方法内的 getEarlyBeanReference 方法调用，getEarlyBeanReference  这个方法的逻辑：

```java
	protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
		Object exposedObject = bean;
		if (bean != null && !mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
					SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
					exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);
					if (exposedObject == null) {
						return null;
					}
				}
			}
		}
		return exposedObject;
	}
```

在 BeanPostProcessor 列表中，有一种 BeanPostProcessor 是 SmartInstantiationAwareBeanPostProcessor。如果存在就回去调用他的 getEarlyBeanReference 方法作为 bean 对象返回。

SmartInstantiationAwareBeanPostProcessor 的子类中有一个是 AbstractAutoProxyCreator。巧的是，AOP 的 BeanPostProcessor 都继承自这个抽象类。比如以前研究过的申明式事务的 bpp InfrastructureAdvisorAutoProxyCreator 就是继承自这个类。

getEarlyBeanReference 的逻辑：

```java
	@Override
	public Object getEarlyBeanReference(Object bean, String beanName) throws BeansException {
		Object cacheKey = getCacheKey(bean.getClass(), beanName);
		this.earlyProxyReferences.put(cacheKey, bean);
		return wrapIfNecessary(bean, beanName, cacheKey);
	}
```

wrapIfNecessary：

```java
	protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
		// ...
		// Create proxy if we have advice.
		Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
		if (specificInterceptors != DO_NOT_PROXY) {
			this.advisedBeans.put(cacheKey, Boolean.TRUE);
			Object proxy = createProxy(bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
			this.proxyTypes.put(cacheKey, proxy.getClass());
			return proxy;
		}
		// ...
		return bean;
	}
```

可以看到 wrapIfNecessary 做的逻辑是生成一个 aop 的代理对象并返回。那么意味着，ObjectFactory 的 getObject 方法的返回就已经是一个 aop 代理对象了。那么回到上面的例子，B2 bean 对 B1 的引用，此时已经是一个经过 aop 包装的代理对象，避免的上面说的问题。

此处还有一个问题，既然 aop 的代理已经提前在 getEarlyBeanReference 中做掉了，那么 B1 属性在属性构造好以后，进入真正的 aop 的 bpp 以后，是不是会重复 aop 的代理逻辑？

答案是不会，aop 的 bpp 的处理逻辑中会判断，当前的 B1 是否已经代理过了，显然这个时候已经代理过，那就不会再重复执行了。判断的依据是 earlyProxyReferences 中是否存在。

#### todo 

补充对 singletonObjects、earlySingletonObjects、singletonFactories、earlyProxyReferences 的解释。

### 构造器的方式

会报异常：

```
BeanCurrentlyInCreationException: Error creating bean with name 'beanHolder': Requested bean is currently in creation: Is there an unresolvable circular reference
```

整体的 bean 构造方式和 set 方式的差不多，只是构造器的方式在初始化 bean 的时候需要使用指定的构造器，set 的方式使用没有参数的构造器。这种情况下，B1 被构造之前必须要将 B2 构造好，不然 B1 无法实例化（区别于 set 的方式）。B2 如果又依赖 B1 的话，spring 就无法继续运行了，因为连 B1 的实例都没有。

spring 是通过将当前正在初始化，但是没有初始化完全的 bean 的 name 放进一个 Set 里去，出现循环引用的时候会重复去 getBean 相同 name 的 bean，就像上面的 B1 一样。然后就无法通过下面的检查，就抛出异常了：

```java
// DefaultSingletonBeanRegistry
protected void beforeSingletonCreation(String beanName) {
    if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.add(beanName)) {
        throw new BeanCurrentlyInCreationException(beanName);
    }
}
```
