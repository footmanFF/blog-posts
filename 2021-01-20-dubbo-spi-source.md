---
layout: post
title: dubbo ExtensionLoader解析
date: 2021-01-20
tags: dubbo,spi,ExtensionLoader
---

### 简介

```java
Codec2 codec2 = ExtensionLoader.getExtensionLoader(Codec2.class).getExtension(codecName)
```

SPI 注解，作为扩展接口 interface 都会标注 SPI 注解，value 标注的是默认使用的 provider：

```java
@Target({ElementType.TYPE})
public @interface SPI {
    /**
     * default extension name
     */
    String value() default "";
}
```

<!-- more -->

### 自定义 SPI 扩展

配置消费端过滤器：

````
- resources
		- META-INF
				- dubbo
						- org.apache.dubbo.rpc.Filter
````

```
DubboConsumerFilter = com.xx.xx.DubboConsumerFilter
```

application.properties

```
dubbo.consumer.filter = DubboConsumerFilter
```

### ExtensionLoader 类

```java
public class ExtensionLoader<T> {
  	// SPI接口和ExtensionLoader的缓存
	  private static final ConcurrentMap<Class<?>, ExtensionLoader<?>> EXTENSION_LOADERS = new ConcurrentHashMap<>();
		
  	// 缓存所有的扩展class和实例，这里的class是具体
  	private static final ConcurrentMap<Class<?>, Object> EXTENSION_INSTANCES = new ConcurrentHashMap<>();
}
```

### 获取一个扩展的执行流程

静态的 getExtensionLoader：

```java
  public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
        // ...
    		ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        if (loader == null) {
            EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
            loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        }
        return loader;
    }
```

实例方法 getExtension：

```java
    public T getExtension(String name) {
        // ..
      	final Holder<Object> holder = getOrCreateHolder(name);
        Object instance = holder.get();
        if (instance == null) {
            synchronized (holder) {
                instance = holder.get();
                if (instance == null) {
                    instance = createExtension(name);  // 关键，如何创建一个扩展
                    holder.set(instance);
                }
            }
        }
        return (T) instance;
    }
```

createExtension：

```java
    private T createExtension(String name) {
        Class<?> clazz = getExtensionClasses().get(name);
        // ..
        T instance = (T) EXTENSION_INSTANCES.get(clazz);
        if (instance == null) {
          	EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
          	instance = (T) EXTENSION_INSTANCES.get(clazz);
        }
        injectExtension(instance);
        // ..
        return instance;
    }
```

getExtensionClasses：

```java
    private Map<String, Class<?>> getExtensionClasses() {
        Map<String, Class<?>> classes = cachedClasses.get();
        if (classes == null) {
            synchronized (cachedClasses) {
                classes = cachedClasses.get();
                if (classes == null) {
                    classes = loadExtensionClasses();   // 加载SPI接口的扩展
                    cachedClasses.set(classes);
                }
            }
        }
        return classes;
    }
```

loadExtensionClasses：

```java
    private Map<String, Class<?>> loadExtensionClasses() {
        // ..
        Map<String, Class<?>> extensionClasses = new HashMap<>();
        loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY, type.getName());
        loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
        loadDirectory(extensionClasses, DUBBO_DIRECTORY, type.getName());
        loadDirectory(extensionClasses, DUBBO_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
        loadDirectory(extensionClasses, SERVICES_DIRECTORY, type.getName());
        loadDirectory(extensionClasses, SERVICES_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
        return extensionClasses;
    }
```

loadDirectory：

扫描某一个 resource 目录（dir），查找 type 类型的所有的扩展实现类。实际上扫描的是以 SPI 接口全限定名为名称的文件，文件中是 kv 结构，key 是扩展名，value 是扩展实现类的类全限定名。

```java
    /**
     * @param extensionClasses 扩展name和扩展class的映射
     * @param dir 扫描目录
     * @param type SPI扩展接口名
     */
		private void loadDirectory(Map<String, Class<?>> extensionClasses, String dir, String type) {
        String fileName = dir + type;
        Enumeration<java.net.URL> urls;
        ClassLoader classLoader = findClassLoader();
        if (classLoader != null) {
          urls = classLoader.getResources(fileName);
        } else {
          urls = ClassLoader.getSystemResources(fileName);
        }
        if (urls != null) {
          while (urls.hasMoreElements()) {
            java.net.URL resourceURL = urls.nextElement();
            loadResource(extensionClasses, classLoader, resourceURL);
          }
        }
    }
```

### Spring bean 注入

ExtensionLoader 有一个属性是 ExtensionFactory，通过这个 ExtensionFactory 完成了对 SPI 扩展的属性注入（依赖注入）。

ExtensionFactory 本身也是一个 SPI 接口。

```java
@SPI
public interface ExtensionFactory {
    // 通过类型，扩展名，拿一个实例
  	<T> T getExtension(Class<T> type, String name);
}
```

那么显然，可以通过 dubbo SPI 的方式，给 ExtensionFactory  提供实现类。

在 META-INF/dubbo/internal 下有个文件 org.apache.dubbo.common.extension.ExtensionFactory，这里标注了 ExtensionFactory  所有的官方提供的实现类：

```
adaptive=org.apache.dubbo.common.extension.factory.AdaptiveExtensionFactory
spi=org.apache.dubbo.common.extension.factory.SpiExtensionFactory
spring=org.apache.dubbo.config.spring.extension.SpringExtensionFactory
```

objectFactory 的初始化：

```java
public class ExtensionLoader<T> {
		private final ExtensionFactory objectFactory;
		// 构造器内，初始化了objectFactory
  	private ExtensionLoader(Class<?> type) {
        this.type = type;
      	if (type == ExtensionFactory.class) {
          	objectFactory = null;
        } else {
            objectFactory = ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
        }
    }
}
```

```java
    private T injectExtension(T instance) {
        for (Method method : instance.getClass().getMethods()) {
            // ..
            Class<?> pt = method.getParameterTypes()[0];
            // ..
            String property = getSetterProperty(method);
            Object object = objectFactory.getExtension(pt, property);
            if (object != null) {
	              method.invoke(instance, object);
            }
        }
        return instance;
    }
```

### ExtensionFactory 默认是如何加载的？

getAdaptiveExtension：

```java
    // 精简以后
		public T getAdaptiveExtension() {
        // ..
        Object instance = createAdaptiveExtension();
        // ..
      	return (T) instance;
    }
```

createAdaptiveExtension：

```java
   	// 精简以后
		private T createAdaptiveExtension() {
      	return injectExtension((T) getAdaptiveExtensionClass().newInstance());
    }
```

getAdaptiveExtensionClass：

```java
    private Class<?> getAdaptiveExtensionClass() {
        getExtensionClasses();
        if (cachedAdaptiveClass != null) {
            // debug到这里返回
          	return cachedAdaptiveClass;
        }
        return cachedAdaptiveClass = createAdaptiveExtensionClass();
    }
```

debug 到这里，可以看到 getAdaptiveExtension 的返回最终来自 cachedAdaptiveClass，并且这里 cachedAdaptiveClass 提前已经被设置好了，那么在什么时候设置的呢？

```
// ExtensionLoader 的 cachedAdaptiveClass
private volatile Class<?> cachedAdaptiveClass = null;
```

通过 debug 发现，在 ExtensionLoader 从 resource 目录的 META-INF/dubbo/xx 中查找扩展类时，会扫描标注了 @Adaptive 注解的扩展类，刚好 ExtensionFactory 的某一个扩展类标注了这个注解，这个类是 AdaptiveExtensionFactory：

```java
    private void loadClass(Map<String, Class<?>> extensionClasses, java.net.URL resourceURL, Class<?> clazz, String name) throws NoSuchMethodException {
        // ...
       	// 这里
        if (clazz.isAnnotationPresent(Adaptive.class)) {
            cacheAdaptiveClass(clazz);
        } else if (isWrapperClass(clazz)) {
            cacheWrapperClass(clazz);
        } else {
						// ..
        }
    }
```

```java
    private void cacheAdaptiveClass(Class<?> clazz) {
        if (cachedAdaptiveClass == null) {
            cachedAdaptiveClass = clazz;
        }
      	// ..
    }
```

再看 AdaptiveExtensionFactory：

```java
@Adaptive
public class AdaptiveExtensionFactory implements ExtensionFactory {

    private final List<ExtensionFactory> factories;

    public AdaptiveExtensionFactory() {
        ExtensionLoader<ExtensionFactory> loader = ExtensionLoader.getExtensionLoader(ExtensionFactory.class);
        List<ExtensionFactory> list = new ArrayList<ExtensionFactory>();
        for (String name : loader.getSupportedExtensions()) {
            list.add(loader.getExtension(name));
        }
        factories = Collections.unmodifiableList(list);
    }

    @Override
    public <T> T getExtension(Class<T> type, String name) {
        for (ExtensionFactory factory : factories) {
            T extension = factory.getExtension(type, name);
            if (extension != null) {
                return extension;
            }
        }
        return null;
    }

}
```

AdaptiveExtensionFactory 的 getExtension 方法会尝试使用所有的 ExtensionFactory 扩展去获取 SPI 扩展，至此就解答了ExtensionFactory  的默认加载问题。

### 运行时指定 SPI 接口实现类的魔法

上面讲了一种通过显示创建的方式获取 adaptiveExtension，从代码看还有一种方式是通过运行时动态生成适配器类

```java
    private Class<?> createAdaptiveExtensionClass() {
        String code = new AdaptiveClassCodeGenerator(type, cachedDefaultName).generate();
        ClassLoader classLoader = findClassLoader();
        org.apache.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(org.apache.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
        return compiler.compile(code, classLoader);
    }
```

AdaptiveClassCodeGenerator 的 generate 方法用字符串拼接出一个 SPI 接口的子类，通过 debug 获取其中一个子类：

```java
package org.apache.dubbo.rpc;

import org.apache.dubbo.common.extension.ExtensionLoader;

public class Protocol$Adaptive implements org.apache.dubbo.rpc.Protocol {
    public void destroy() {
        throw new UnsupportedOperationException("The method public abstract void org.apache.dubbo.rpc.Protocol.destroy() of interface org.apache.dubbo.rpc.Protocol is not adaptive method!");
    }

    public int getDefaultPort() {
        throw new UnsupportedOperationException("The method public abstract int org.apache.dubbo.rpc.Protocol.getDefaultPort() of interface org.apache.dubbo.rpc.Protocol is not adaptive method!");
    }

    public org.apache.dubbo.rpc.Exporter export(
        org.apache.dubbo.rpc.Invoker arg0) throws org.apache.dubbo.rpc.RpcException {
        if (arg0 == null)
            throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument == null");
        if (arg0.getUrl() == null)
            throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument getUrl() == null");
        org.apache.dubbo.common.URL url = arg0.getUrl();
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        if (extName == null)
            throw new IllegalStateException("Failed to get extension (org.apache.dubbo.rpc.Protocol) name from url (" + url.toString() + ") use keys([protocol])");
        org.apache.dubbo.rpc.Protocol extension = (org.apache.dubbo.rpc.Protocol) ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.Protocol.class).getExtension(extName);
        return extension.export(arg0);
    }

    public org.apache.dubbo.rpc.Invoker refer(java.lang.Class arg0,
        org.apache.dubbo.common.URL arg1) throws org.apache.dubbo.rpc.RpcException {
        if (arg1 == null)
            throw new IllegalArgumentException("url == null");
        org.apache.dubbo.common.URL url = arg1;
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        if (extName == null)
            throw new IllegalStateException("Failed to get extension (org.apache.dubbo.rpc.Protocol) name from url (" + url.toString() + ") use keys([protocol])");
        org.apache.dubbo.rpc.Protocol extension = (org.apache.dubbo.rpc.Protocol) ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.Protocol.class).getExtension(extName);
        return extension.refer(arg0, arg1);
    }
}
```

这个是 Protocol 接口的实现，并且是运行时动态生成的。比如其中的 export 方法：

```java
URL url = arg0.getUrl();
String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
Protocol extension = (Protocol) ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(extName);
extension.export(arg0);
```

他的逻辑是通过 Invoker 获取到一个 protocal 扩展的 name，并从 ExtensionLoader 中运行时拿到一个 Protocol 的实例，再执行他的 export。

这里相当于是运行时动态根据 url 中的参数去选择使用 SPI 的实现，这里就是「运行时指定 SPI 接口实现类的魔法」。

### 同一个 SPI 接口可以实现 AOP

如果一个 SPI 接口有多个实现，每个实现的外层想包装复用的逻辑，有一种优雅的方法可以做到：

在 META-INF/dubbo 下配置扩展实现时，可以增加一类 Wrapper 扩展实现类。如果一个实现类有一个只有个一个参数的构造器，并且构造器参数类型就是 SPI 接口的类型，那么 dubbo SPI 机制会自动去用 Wrapper 类作为真实 SPI 接口实现类的代理类。如果 Wrapper 类在真实实现类的完成包装逻辑，那么就实现了 AOP 的目的了，只是这里的代理是静态的，不是动态的。

并且 Wrapper 类可以指定多个，Wrapper 类会层层封装在真实 SPI 接口实现类的外层。

创建 SPI 接口扩展实现类的逻辑：

```java
    @SuppressWarnings("unchecked")
    private T createExtension(String name) {
        Class<?> clazz = getExtensionClasses().get(name);
        T instance = (T) EXTENSION_INSTANCES.get(clazz);
        if (instance == null) {
            EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        }
        injectExtension(instance);
        // 关键点在这里
        Set<Class<?>> wrapperClasses = cachedWrapperClasses;
        if (CollectionUtils.isNotEmpty(wrapperClasses)) {
            for (Class<?> wrapperClass : wrapperClasses) {
                // 代理生成
            	  instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
            }
        }
        return instance;
    }
```

cachedWrapperClasses 类是所有在扫包的时候发现的 wapper 类的 class。最终在初始化 extension 的时候回通过 wapper 类的构造器初始化一个代理类，代理原始的 extension 对象。并且因为会在一个循环中层层包装代理类，有可能对一个 extension  对象包装多层。

扫描逻辑：

```java
    private void loadClass(Map<String, Class<?>> extensionClasses, java.net.URL resourceURL, Class<?> clazz, String name) throws NoSuchMethodException {
        // ...
      	if (clazz.isAnnotationPresent(Adaptive.class)) {
            cacheAdaptiveClass(clazz);
        } else if (isWrapperClass(clazz)) {
            cacheWrapperClass(clazz);   // 此处维护了 cachedWrapperClasses
        } else {
						// ...
        }
    }
```

isWrapperClass：

isWrapperClass 判断了「只有个一个参数的构造器，并且构造器参数类型就是 SPI 接口的类型」。

```java
    private boolean isWrapperClass(Class<?> clazz) {
        try {
            clazz.getConstructor(type);
            return true;
        } catch (NoSuchMethodException e) {
            return false;
        }
    }
```

### 关于 Activate 注解

https://blog.csdn.net/qq_35190492/article/details/108256452