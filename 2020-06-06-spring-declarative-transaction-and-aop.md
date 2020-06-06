---
layout: post
title: spring声明式事务如何织入事务逻辑
date: 2020-06-06
tags: spring
---

#### 问题

1. @Transactional 注解标注的申明式事务是用 AOP 实现的，那么事务逻辑是怎么织入的呢？
2. 织入的事务逻辑，是怎么处理连接的？底层的 dao 实现也需要连接，应该是用 ThreadLocal 做的，那么具体是怎么实现的？



```java
public abstract class AbstractBeanFactory extends FactoryBeanRegistrySupport implements ConfigurableBeanFactory {
  // ...
  sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
    @Override
    public Object getObject() throws BeansException {
      try {
        return createBean(beanName, mbd, args);
      }
      catch (BeansException ex) {
        // Explicitly remove instance from singleton cache: It might have been put there
        // eagerly by the creation process, to allow for circular reference resolution.
        // Also remove any beans that received a temporary reference to the bean.
        destroySingleton(beanName);
        throw ex;
      }
    }
  });
  // ...
}
```
<!-- more -->
AbstractAutowireCapableBeanFactory：

```java
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory
		implements AutowireCapableBeanFactory {
  protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)
    throws BeanCreationException {
  	// ...
    // Initialize the bean instance.
		Object exposedObject = bean;
		try {
			populateBean(beanName, mbd, instanceWrapper);
			if (exposedObject != null) {
				exposedObject = initializeBean(beanName, exposedObject, mbd);
			}
		}
    // ... 
  }
}
```

initializeBean：

```java
	protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
    // ...
    if (mbd == null || !mbd.isSynthetic()) {
			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
		}
		return wrappedBean;
  }
```

applyBeanPostProcessorsAfterInitialization：

```java
	@Override
	public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
			throws BeansException {

		Object result = existingBean;
		for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {
			result = beanProcessor.postProcessAfterInitialization(result, beanName);
			if (result == null) {
				return result;
			}
		}
		return result;
	}
```

getBeanPostProcessors() 中的 InfrastructureAdvisorAutoProxyCreator，在经过 postProcessAfterInitialization 以后 result 变成 cglib 包装的代理类：

![2020-06-02 at 9.04 PM](http://note-1255449501.file.myqcloud.com/2020-06-02-130446.png)

InfrastructureAdvisorAutoProxyCreator：

```java
public class InfrastructureAdvisorAutoProxyCreator extends AbstractAdvisorAutoProxyCreator {
  	public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
    	//.. 
      return wrapIfNecessary(bean, beanName, cacheKey);
    }
}
```

```java
	protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
		Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
    Object proxy = createProxy(bean.getClass(), beanName, 
                               specificInterceptors, new SingletonTargetSource(bean));
    return proxy;
	}
```

```java
	// 生路部分代码
	protected Object createProxy(
			Class<?> beanClass, String beanName, 
    Object[] specificInterceptors, TargetSource targetSource) {
  
    ProxyFactory proxyFactory = new ProxyFactory();
		proxyFactory.copyFrom(this);

    evaluateProxyInterfaces(beanClass, proxyFactory);
		// BeanFactoryTransactionAttributeSourceAdvisor
		Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
		proxyFactory.addAdvisors(advisors);
		proxyFactory.setTargetSource(targetSource);
		customizeProxyFactory(proxyFactory);

		proxyFactory.setFrozen(this.freezeProxy);
		proxyFactory.setPreFiltered(true);

		return proxyFactory.getProxy(getProxyClassLoader());
  }
```

ProxyFactory：

```java
public class ProxyFactory extends ProxyCreatorSupport {
  public Object getProxy(ClassLoader classLoader) {
		return createAopProxy().getProxy(classLoader);
	}
  protected final synchronized AopProxy createAopProxy() {
		// ..
		return getAopProxyFactory().createAopProxy(this);
	}
}
```

DefaultAopProxyFactory：

```java
public class DefaultAopProxyFactory implements AopProxyFactory, Serializable {
  @Override
	public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
    // config.isProxyTargetClass() == true
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class<?> targetClass = config.getTargetClass();
			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
				return new JdkDynamicAopProxy(config);
			}
			return new ObjenesisCglibAopProxy(config);
		}
		else {
			return new JdkDynamicAopProxy(config);
		}
	}
}
```

ObjenesisCglibAopProxy：

```java
class ObjenesisCglibAopProxy extends CglibAopProxy {
  // 继承自CglibAopProxy的方法
  @Override
	public Object getProxy(ClassLoader classLoader) {
    Class<?> rootClass = this.advised.getTargetClass();
    Enhancer enhancer = createEnhancer();
    // ..
    enhancer.setClassLoader(classLoader);
    enhancer.setSuperclass(proxySuperClass);
    enhancer.setInterfaces(AopProxyUtils.completeProxiedInterfaces(this.advised));
    enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
    enhancer.setStrategy(new ClassLoaderAwareUndeclaredThrowableStrategy(classLoader));
    
		// [0]: CglibAopProxy$DynamicAdvisedInterceptor
    Callback[] callbacks = getCallbacks(rootClass);
    Class<?>[] types = new Class<?>[callbacks.length];
    for (int x = 0; x < types.length; x++) {
      types[x] = callbacks[x].getClass();
    }
    // fixedInterceptorMap only populated at this point, after getCallbacks call above
    enhancer.setCallbackFilter(new ProxyCallbackFilter(
      this.advised.getConfigurationOnlyCopy(), this.fixedInterceptorMap, this.fixedInterceptorOffset));
    enhancer.setCallbackTypes(types);

    // Generate the proxy class and create a proxy instance.
    return createProxyClassAndInstance(enhancer, callbacks);
  }
}
```

```java
	@Override
	protected Object createProxyClassAndInstance(Enhancer enhancer, Callback[] callbacks) {
    // (1)
    Class<?> proxyClass = enhancer.createClass();
    // (2)
    Object proxyInstance = objenesis.newInstance(proxyClass, enhancer.getUseCache());
    // (3)
  	((Factory) proxyInstance).setCallbacks(callbacks);
    return proxyInstance;
  }
```

（1）：enhancer.createClass 是 cglib 的方法，这里是获得增强对象的 Class。

（2）：通过反射执行构造器，获得代理对象。

到此处 aop 代理对象已经生成，那么逻辑是在什么地方织入的呢？cglib 相对于 spring 是一个外部包，他的基础使用姿势 [Cglib及其基本使用](https://www.cnblogs.com/xrq730/p/6661692.html) 。

cglib 可以在方法外层进行拦截，构造代理对象的时候给 cglib 提供一个方法拦截逻辑实现即可。这个方法拦截的由接口 MethodInterceptor 定义：

```java
public interface MethodInterceptor extends Callback {
    public Object intercept(Object obj, java.lang.reflect.Method method, Object[] args,
                               MethodProxy proxy) throws Throwable;
}
```

cglib 最最基本的例子类似这样：

```java
Enhancer enhancer = new Enhancer();
enhancer.setSuperclass(SomeClass.class);
enhancer.setCallback(methodInterceptorImpl);
Object proxy = enhancer.create();
```

对照上述 spring 创建代理的逻辑，设置 MethodInterceptor 是在 ObjenesisCglibAopProxy 中获取到：

```java
class ObjenesisCglibAopProxy extends CglibAopProxy {
  // 继承自CglibAopProxy的方法
  @Override
	public Object getProxy(ClassLoader classLoader) {
		//..
    Callback[] callbacks = getCallbacks(rootClass);
    //..
  }
}
```

但是最终被使用是在 createProxyClassAndInstance，上述的（3）位置。

（3）：把得到的代理对象强转成 net.sf.cglib.proxy.Factory，并调用他的 setCallbacks 方法。那么这个 Factory 接口是咋回事呢？

```java
/**
 * All enhanced instances returned by the Enhancer class implement this interface.
 * Using this interface for new instances is faster than going through the Enhancer
 * interface or using reflection. In addition, to intercept methods called during
 * object construction you must use these methods instead of reflection.
 *
 * 所有的 cglib 增强的对象实现这个接口。
 */
public interface Factory {
  // ...
}
```

至此，织入逻辑的点找到了。收纳注入逻辑类是 DynamicAdvisedInterceptor：

```java
class CglibAopProxy implements AopProxy, Serializable {
  private static class DynamicAdvisedInterceptor implements MethodInterceptor, Serializable {
    @Override
		public Object intercept(Object proxy, Method method, 
                            Object[] args, MethodProxy methodProxy) throws Throwable {
      // ..
      List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
			Object retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
      // ..
      return retVal;
    }
  }
}
```

DynamicAdvisedInterceptor 继承自 cglib 的 callback 接口，即上述（3）处的 callbacks （数组）中的一个。

```java
public class ReflectiveMethodInvocation implements ProxyMethodInvocation, Cloneable {
  @Override
	public Object proceed() throws Throwable {
		//..
		Object interceptorOrInterceptionAdvice =
				this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
		if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
			InterceptorAndDynamicMethodMatcher dm =
					(InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
			if (dm.methodMatcher.matches(this.method, this.targetClass, this.arguments)) {
				// (1)
        return dm.interceptor.invoke(this);
			}
			else {
				return proceed();
			}
		}
    //..
	}
}
```

（1）： proceed 方法会调用一个执行链，链上是一些方法的拦截器，此处拦截器只有一个，是 TransactionInterceptor。在其上做了事务相关的逻辑：

```java
public class TransactionInterceptor extends TransactionAspectSupport 
  implements MethodInterceptor, Serializable {
  	@Override
	public Object invoke(final MethodInvocation invocation) throws Throwable {
		Class<?> targetClass = (invocation.getThis() != null ?  : null);
		return invokeWithinTransaction(invocation.getMethod(), targetClass, new InvocationCallback() {
			@Override
			public Object proceedWithInvocation() throws Throwable {
				return invocation.proceed();
			}
		});
	}
}
```

invokeWithinTransaction：

```java
public abstract class TransactionAspectSupport implements BeanFactoryAware, InitializingBean {
  protected Object invokeWithinTransaction(Method method, Class<?> targetClass, 
                                           final InvocationCallback invocation) throws Throwable {
    //..
    // (1)
    TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
    Object retVal = null;
    try {
      // This is an around advice: Invoke the next interceptor in the chain.
      // This will normally result in a target object being invoked.
      retVal = invocation.proceedWithInvocation();
    }
    catch (Throwable ex) {
      // target invocation exception
      completeTransactionAfterThrowing(txInfo, ex);
      throw ex;
    }
    finally {
      cleanupTransactionInfo(txInfo);
    }
    commitTransactionAfterReturning(txInfo);
    return retVal;
    //..
  }
}
```

获取实际数据连接的地方在 （1）：

```java
	protected TransactionInfo createTransactionIfNecessary(
			PlatformTransactionManager tm, TransactionAttribute txAttr, final String joinpointIdentification) {
  	//..
    TransactionStatus status = tm.getTransaction(txAttr);
    return prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
  }
```

接着是 PlatformTransactionManager.getTransaction：

```java
public abstract class AbstractPlatformTransactionManager 
  implements PlatformTransactionManager, Serializable {
  @Override
	public final TransactionStatus getTransaction(TransactionDefinition definition)
    throws TransactionException {
  	Object transaction = doGetTransaction();
    
    if (isExistingTransaction(transaction)) {
      return xx;
    }
    
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
				definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
				definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED){
      
      boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
      DefaultTransactionStatus status = newTransactionStatus(
						definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
      
      // 
      doBegin(transaction, definition);
      prepareSynchronization(status, definition);
      return status;
    }
    // ..
  }
}
```

doBegin 方法，省略的部分逻辑：

```java
	@Override
	protected void doBegin(Object transaction, TransactionDefinition definition) {
  	DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
		Connection con = null;
    if (txObject.getConnectionHolder() == null ||
        txObject.getConnectionHolder().isSynchronizedWithTransaction()) {
      // (1)
      Connection newCon = this.dataSource.getConnection();
      txObject.setConnectionHolder(new ConnectionHolder(newCon), true);
    }
    txObject.getConnectionHolder().setSynchronizedWithTransaction(true);
    con = txObject.getConnectionHolder().getConnection();

    Integer previousIsolationLevel = DataSourceUtils.prepareConnectionForTransaction(con, definition);
    txObject.setPreviousIsolationLevel(previousIsolationLevel);

    // Switch to manual commit if necessary. This is very expensive in some JDBC drivers,
    // so we don't want to do it unnecessarily (for example if we've explicitly
    // configured the connection pool to set it already).
    if (con.getAutoCommit()) {
      txObject.setMustRestoreAutoCommit(true);
      con.setAutoCommit(false);
    }
    txObject.getConnectionHolder().setTransactionActive(true);

    int timeout = determineTimeout(definition);
    if (timeout != TransactionDefinition.TIMEOUT_DEFAULT) {
      txObject.getConnectionHolder().setTimeoutInSeconds(timeout);
    }

    // Bind the session holder to the thread.
    if (txObject.isNewConnectionHolder()) {
      // (2)
      TransactionSynchronizationManager.bindResource(getDataSource(), txObject.getConnectionHolder());
    }
    
    // 异常处理
  }
```

（1）：从 DataSource 获取了连接。

（2）：设置将连接设置进 TransactionSynchronizationManager 的 ThreadLocal 内。