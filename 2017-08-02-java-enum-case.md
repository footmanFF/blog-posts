---
layout: post
title: 使用Java枚举碰到的一个问题
date: 2017-08-02
tags: Java, enum
---

## enum 的一个问题

有这样一个场景，服务端序列化一个枚举类并传递给客户端，开始是没有问题的。但是当服务端增加一个枚举值而没有通知客户端的时候，客户端反序列化就会报异常了。

<!-- more -->

```j
Caused by: java.io.InvalidObjectException: enum constant PKG_RESULT does not exist in class com.weidai.cf.rdc.common.enums.UserInfoFieldEnum
	at java.io.ObjectInputStream.readEnum(ObjectInputStream.java:1972)
	at java.io.ObjectInputStream.readObject0(ObjectInputStream.java:1532)
	at java.io.ObjectInputStream.readObject(ObjectInputStream.java:422)
```

原因很简单，客户端的枚举 class 没有对应的枚举值，自然无法反序列化，那么就造成了问题。

这篇文章和下面的评论基本说清楚了问题：

<http://brian.pontarelli.com/2006/01/17/versioning-and-serialization-of-enums/>

## readResolve

readResolve 会在 readObject 从二进制流中读出对象以后被调用，通过在对象中实现这个类，可以替换或者加工 readObject 返回的类。

看这里：

<https://stackoverflow.com/questions/1168348/java-serialization-readobject-vs-readresolve>

那是不是能通过 readResolve 来解决呢？答案是不行。因为 Java 枚举对象的序列化和反序列化是独立处理的，readResolve 即使被定义在枚举类中，也会被忽略不执行的。

ObjectInputStream 的 JDK 注释：

> Enum constants are deserialized differently than ordinary serializable or externalizable objects. The serialized form of an enum constant consists solely of its name; field values of the constant are not transmitted. To deserialize an enum constant, ObjectInputStream reads the constant name from the stream; the deserialized constant is then obtained by calling the static method `Enum.valueOf(Class, String)` with the enum constant's base type and the received constant name as arguments. Like other serializable or externalizable objects, enum constants can function as the targets of back references appearing subsequently in the serialization stream. The process by which enum constants are deserialized cannot be customized: any class-specific readObject, readObjectNoData, and readResolve methods defined by enum types are ignored during deserialization. Similarly, any serialPersistentFields or serialVersionUID field declarations are also ignored--all enum types have a fixed serialVersionUID of 0L.

Java 这么做是有原因的，当一个枚举被反序列化的时候，如果用默认的反序列化规则，那么会创建一个新的实例，然而枚举又要求自己是单例的，那 Java 只能用特殊的反序列化机制去保证这个单例。

<https://stackoverflow.com/questions/34589325/why-default-serialization-is-prevented-in-enum-class>

## 解决

但是又想在 RPC 中用枚举，怎么解决呢？一种办法是用类来模拟枚举：

<https://stackoverflow.com/questions/3520462/simulate-a-class-of-type-enum>

```java
// The typesafe enum pattern
public class Suit {
    private final String name;
    private Suit(String name) { this.name = name; }
    public String toString() { return name; }
    public static final Suit CLUBS = new Suit("clubs");
    public static final Suit DIAMONDS = new Suit("diamonds");
    public static final Suit HEARTS = new Suit("hearts");
    public static final Suit SPADES = new Suit("spades");
}
```

## 总结

因为枚举在 RPC 的时候会有反序列化问题，那么当枚举表达的类别很有可能会增加的时候，就别用枚举了，用 int 或 String 等等。当表达性别这种不太会增加或者减少的类别时，用枚举才比较合适。当然如果不在 RPC 环境下，用枚举会比 int 好。