---
layout: post
title: Spring Boot 的 @Resource 注解不生效问题解决
date: 2018-08-25
---

### 背景

本来想搭一下 Spring-Boot 组合 Spock 做单元测试，弄了一个很小的 demo 项目测试，结果等到运行的时候 @Resource 注释不生效，bean name 也没错。反而 @Autowired 是能成功注入的，这个真的是奇了怪了。项目大概是这样的：

```java
@SpringBootConfiguration
public class App {
    public static void main(String[] args) {
        ConfigurableApplicationContext applicationContext = SpringApplication.run(App.class, args);
        A a = applicationContext.getBean(A.class);
        B b = a.getB();
        System.out.println(b);
    }
}
```

##### 类 A

```java
@Component
public class A {
    @Resource
    private B b;
    public B getB() {
        return b;
    }
}
```

##### 类 B

```java
@Service
public class B {
}
```

现象是一运行 A 对象内的 B 属性注入不进来，通过 applicationContext.gerBean 是可以正常获取 B 类的 bean，这 TM 就奇了怪了。然后就去 debug 源码。

最初是在 Spring 扫描包路径获得 BeanDefinition 的地方找是不是有什么问题，是不是这个时候获得的 BeanDefinition 不对，遗漏了 A 类的属性。找了半天，发现 @Resource 注释的属性是在 CommonAnnotationBeanPostProcessor 类内部扫描获得的：

```java
public class CommonAnnotationBeanPostProcessor {
    	@Override
	public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
		super.postProcessMergedBeanDefinition(beanDefinition, beanType, beanName);
		if (beanType != null) {
			InjectionMetadata metadata = findResourceMetadata(beanName, beanType, null);
			metadata.checkConfigMembers(beanDefinition);
		}
	}
    
    	@Override
	public PropertyValues postProcessPropertyValues(
			PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeansException {

		InjectionMetadata metadata = findResourceMetadata(beanName, bean.getClass(), pvs);
		try {
			metadata.inject(bean, beanName, pvs);
		}
		catch (Throwable ex) {
			throw new BeanCreationException(beanName, "Injection of resource dependencies failed", ex);
		}
		return pvs;
	}
}
```

##### 方法 postProcessMergedBeanDefinition

这个方法在创建 bean 的时候会回调，调用路径是 

- AbstractAutowireCapableBeanFactory.doCreateBean
- AbstractAutowireCapableBeanFactory.applyMergedBeanDefinitionPostProcessors
- CommonAnnotationBeanPostProcessor.postProcessMergedBeanDefinition

CommonAnnotationBeanPostProcessor 的抽象是 BeanPostProcessor，Spring 初始化的维护在 AbstractBeanFactory 的属性中，等到创建 bean 需要回调这些 processor 的时候就直接拿来用。

他的 findResourceMetadata 方法会去扫描当前处理 bean 的属性，如果返现有被 @Resource 注释的属性，就会提取出来创建成一个对象，InjectionMetadata 类。然后通过这个类的内部的方法 checkConfigMembers 来，将 @Resource 注释的属性封装成 InjectedElement 对象，并维护到 InjectionMetadata 的 checkedElements 属性内（类型为 Set<InjectedElement>）。这个 checkedElements 是在下一个 postProcessPropertyValues 方法中被用到的。

##### 方法 postProcessPropertyValues

这个方法的调用链如下：

- AbstractAutowireCapableBeanFactory.doCreateBean

- AbstractAutowireCapableBeanFactory.populateBean
  这个方法直接看循环引用的时候看到过，是创建 bean 时，属性有对其他 bean 依赖的情况会用到。

  ```
  /**
  * Populate the bean instance in the given BeanWrapper with the property values
  * from the bean definition.
  * @param beanName the name of the bean
  * @param mbd the bean definition for the bean
  * @param bw BeanWrapper with bean instance
  */
  protected void populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw) {}
  ```

- CommonAnnotationBeanPostProcessor.postProcessPropertyValues

他会提取上一个方法获得的 InjectedElement，分别调用他们的 inject 方法：

```java
/**
 * Either this or {@link #getResourceToInject} needs to be overridden.
*/
protected void inject(Object target, String requestingBeanName, PropertyValues pvs) throws Throwable {
    if (this.isField) {
        Field field = (Field) this.member;
        ReflectionUtils.makeAccessible(field);
        field.set(target, getResourceToInject(target, requestingBeanName));
    } else {
        if (checkPropertySkipping(pvs)) {
            return;
        }
        try {
            Method method = (Method) this.member;
            ReflectionUtils.makeAccessible(method);
            method.invoke(target, getResourceToInject(target, requestingBeanName));
        }
        catch (InvocationTargetException ex) {
            throw ex.getTargetException();
        }
    }
}
```

执行的逻辑比较清晰，拿到反射获得的 Field，利用反射将属性设置为可访问，并去创建属性所依赖的 bean。调用 getResourceToInject 方法去注入创建这个 bean。后续的过程不在本次讨论范围内，有兴趣的可以看看。

##### 那么为什么 @Resource 注释的属性没有注入进 bean 呢

原因很简单，AbstractBeanFactory 在初始化的时候，会在内部维护一个 BeanPostProcessor 的列表，用于在需要的时候拿出来使用，执行回调：

```java
public abstract class AbstractBeanFactory extends FactoryBeanRegistrySupport implements ConfigurableBeanFactory {
	/** BeanPostProcessors to apply in createBean */
	private final List<BeanPostProcessor> beanPostProcessors = new ArrayList<BeanPostProcessor>();
}
```

但是 debug 的时候压根没有看到 CommonAnnotationBeanPostProcessor，没有这个 processor 自然就无法处理 @Resource 注释了，而且也是不会报错的。

##### 那么为什么没有 CommonAnnotationBeanPostProcessor 这个 processor 呢？

这些 processor 是在容器初始化的时候创建的，AbstractApplicationContext 的 refresh 方法看过 Spring 启动过程的应该会比较眼熟，这个方法中的 registerBeanPostProcessors 步骤会去做这个事情：

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
    // Prepare this context for refreshing.
    prepareRefresh();
    // Tell the subclass to refresh the internal bean factory.
    ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
    // Prepare the bean factory for use in this context.
    prepareBeanFactory(beanFactory);
    // Allows post-processing of the bean factory in context subclasses.
    postProcessBeanFactory(beanFactory);
    // Invoke factory processors registered as beans in the context.
    invokeBeanFactoryPostProcessors(beanFactory);
    // Register bean processors that intercept bean creation.
    registerBeanPostProcessors(beanFactory);
    // Initialize message source for this context.
    initMessageSource();
    // Initialize event multicaster for this context.
    initApplicationEventMulticaster();
    // Initialize other special beans in specific context subclasses.
    onRefresh();
    // Check for listener beans and register them.
    registerListeners();
    // Instantiate all remaining (non-lazy-init) singletons.
    finishBeanFactoryInitialization(beanFactory);
    // Last step: publish corresponding event.
    finishRefresh();
}
```

这个方法内部的逻辑比较简单，去看了基本都能明白，最关键的是这一行：

```java
String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);
```

getBeanNamesForType 方会从容器中所有注册的 BeanDefinition 中寻找是 processor 的 bean，获得他们的 bean name 然后返回。后续的代码再通过这些 name 去拿到真实的 bean。

从其他能处理 @Resource 属性的项目 debug 的时候，发现这些 name 是这样的：

![2018-08-25 at 4.59 PM](/var/folders/fd/ptrbg3sx0cv0k988y2qdqxnm0000gn/T/se.razola.Glui2/B8D69E1D-E11C-48E1-8FC7-27054EFD1A3D-477-00004DDB369CF63A/2018-08-25 at 4.59 PM.png)

其中的 org.springframework.context.annotation.internalCommonAnnotationProcessor 就是重点，我的 dmoe 中这个 bean name 没有返回。

##### 那么为什么 internalCommonAnnotationProcessor 没有被注册进容器呢？

看起来应该是一个默认一定会有的 bean，然后 Shift + Command + F 搜这个类，全局只存在一定地方有这个完整的字符：

```java
public class AnnotationConfigUtils {
	/**
	 * The bean name of the internally managed JSR-250 annotation processor.
	 */
	public static final String COMMON_ANNOTATION_PROCESSOR_BEAN_NAME =
			"org.springframework.context.annotation.internalCommonAnnotationProcessor";
}
```

接着继续搜 COMMON_ANNOTATION_PROCESSOR_BEAN_NAME，看什么地方有用到：

```java
private static final boolean jsr250Present =
    ClassUtils.isPresent("javax.annotation.Resource", AnnotationConfigUtils.class.getClassLoader());
/**
  * Register all relevant annotation post processors in the given registry.
  * @param registry the registry to operate on
  * @param source the configuration source element (already extracted)
  * that this registration was triggered from. May be {@code null}.
  * @return a Set of BeanDefinitionHolders, containing all bean definitions
  * that have actually been registered by this call
  */
public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
    BeanDefinitionRegistry registry, Object source) {
    // Check for JSR-250 support, and if present add the CommonAnnotationBeanPostProcessor.
    if (jsr250Present && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, COMMON_ANNOTATION_PROCESSOR_BEAN_NAME));
    }
}
```

这个 if 前后省略了一些逻辑。到这里就很明显了，Spring 启动时，会用去加载 javax.annotation.Resource 类，如果加载不到，那么也就不会去注册 internalCommonAnnotationProcessor 了。

加载 Resource 类的逻辑还有个点是，没加载到类是不会有日志信息的，警告也不会：

```java
public static Class<?> forName(String name, ClassLoader classLoader) throws ClassNotFoundException, LinkageError {
    ClassLoader clToUse = classLoader;
    if (clToUse == null) {
        clToUse = getDefaultClassLoader();
    }
    try {
        return (clToUse != null ? clToUse.loadClass(name) : Class.forName(name));
    }
    catch (ClassNotFoundException ex) {
        int lastDotIndex = name.lastIndexOf(PACKAGE_SEPARATOR);
        if (lastDotIndex != -1) {
            String innerClassName =
                name.substring(0, lastDotIndex) + INNER_CLASS_SEPARATOR + name.substring(lastDotIndex + 1);
            try {
                return (clToUse != null ? clToUse.loadClass(innerClassName) : Class.forName(innerClassName));
            }
            catch (ClassNotFoundException ex2) {
                // Swallow - let original exception get through
                // 这里异常直接忽略了
            }
        }
        throw ex;
    }
}
```

自己手动去掉 printStackTrace 会报这个：

```
java.lang.ClassNotFoundException: javax.annotation.Resource
	at java.base/jdk.internal.loader.BuiltinClassLoader.loadClass(BuiltinClassLoader.java:582)
	at java.base/jdk.internal.loader.ClassLoaders$AppClassLoader.loadClass(ClassLoaders.java:185)
	at java.base/java.lang.ClassLoader.loadClass(ClassLoader.java:496)
	at org.springframework.util.ClassUtils.forName(ClassUtils.java:250)
	at org.springframework.util.ClassUtils.isPresent(ClassUtils.java:327)
	at com.footmanff.utsample.App.main(App.java:46)
```

原因很明显了，因为创建项目的时候手残选了 jdk 9，jdk 9 开始把 jdk 拆成不同模块了，IDEA 默认的运行是只用 java.base 模块的，而这个模块下是没有 javax.annotation.Resource 类的，他在 java.xml.ws.annotation 模块下：

![2018-08-25 at 5.08 PM](http://note-1255449501.file.myqcloud.com/2018-08-25-090823.png)

到此就解决了。