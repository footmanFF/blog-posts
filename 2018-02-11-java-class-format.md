---
layout: post
title: Java class文件结构
date: 2018-02-11
tags: java,class, jvm
---

### Java 文件

```java
package com.footmanff.jdktest.base;

public class TestClass {
    private int m;
    public int inc() {
        return m + 1;
    }
}

```

<!-- more -->

### javap

```
Classfile /private/tmp/TestClass.class
  Last modified Feb 11, 2018; size 407 bytes
  MD5 checksum 23d9ff52b20b01a74936eea0c866e6d7
  Compiled from "TestClass.java"
public class com.footmanff.jdktest.base.TestClass
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #4.#18         // java/lang/Object."<init>":()V
   #2 = Fieldref           #3.#19         // com/footmanff/jdktest/base/TestClass.m:I
   #3 = Class              #20            // com/footmanff/jdktest/base/TestClass
   #4 = Class              #21            // java/lang/Object
   #5 = Utf8               m
   #6 = Utf8               I
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable
  #11 = Utf8               LocalVariableTable
  #12 = Utf8               this
  #13 = Utf8               Lcom/footmanff/jdktest/base/TestClass;
  #14 = Utf8               inc
  #15 = Utf8               ()I
  #16 = Utf8               SourceFile
  #17 = Utf8               TestClass.java
  #18 = NameAndType        #7:#8          // "<init>":()V
  #19 = NameAndType        #5:#6          // m:I
  #20 = Utf8               com/footmanff/jdktest/base/TestClass
  #21 = Utf8               java/lang/Object
{
  public com.footmanff.jdktest.base.TestClass();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 6: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcom/footmanff/jdktest/base/TestClass;

  public int inc();
    descriptor: ()I
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=1, args_size=1
         0: aload_0
         1: getfield      #2                  // Field m:I
         4: iconst_1
         5: iadd
         6: ireturn
      LineNumberTable:
        line 11: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       7     0  this   Lcom/footmanff/jdktest/base/TestClass;
}
SourceFile: "TestClass.java"
```

### class 文件 16 进制输出

（jdk 1.8.0_131）：

```
    0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F
1.  ca fe ba be 00 00 00 34 00 16 0a 00 04 00 12 09 
2.  00 03 00 13 07 00 14 07 00 15 01 00 01 6d 01 00 
3.  01 49 01 00 06 3c 69 6e 69 74 3e 01 00 03 28 29 
4.  56 01 00 04 43 6f 64 65 01 00 0f 4c 69 6e 65 4e 
5.  75 6d 62 65 72 54 61 62 6c 65 01 00 12 4c 6f 63 
6.  61 6c 56 61 72 69 61 62 6c 65 54 61 62 6c 65 01 
7.  00 04 74 68 69 73 01 00 26 4c 63 6f 6d 2f 66 6f 
8.  6f 74 6d 61 6e 66 66 2f 6a 64 6b 74 65 73 74 2f 
9.  62 61 73 65 2f 54 65 73 74 43 6c 61 73 73 3b 01 
10. 00 03 69 6e 63 01 00 03 28 29 49 01 00 0a 53 6f 
11. 75 72 63 65 46 69 6c 65 01 00 0e 54 65 73 74 43 
12. 6c 61 73 73 2e 6a 61 76 61 0c 00 07 00 08 0c 00 
13. 05 00 06 01 00 24 63 6f 6d 2f 66 6f 6f 74 6d 61 
14. 6e 66 66 2f 6a 64 6b 74 65 73 74 2f 62 61 73 65 
15. 2f 54 65 73 74 43 6c 61 73 73 01 00 10 6a 61 76 
16. 61 2f 6c 61 6e 67 2f 4f 62 6a 65 63 74 00 21 00 
17. 03 00 04 00 00 00 01 00 02 00 05 00 06 00 00 00 
18. 02 00 01 00 07 00 08 00 01 00 09 00 00 00 2f 00 
19. 01 00 01 00 00 00 05 2a b7 00 01 b1 00 00 00 02 
20. 00 0a 00 00 00 06 00 01 00 00 00 06 00 0b 00 00 
21. 00 0c 00 01 00 00 00 05 00 0c 00 0d 00 00 00 01 
22. 00 0e 00 0f 00 01 00 09 00 00 00 31 00 02 00 01 
23. 00 00 00 07 2a b4 00 02 04 60 ac 00 00 00 02 00 
24. 0a 00 00 00 06 00 01 00 00 00 0b 00 0b 00 00 00 
25. 0c 00 01 00 00 00 07 00 0c 00 0d 00 00 00 01 00 
26. 10 00 00 00 02 00 11 
```

比如第 1 行第 1 列的值为 ca，就标记为 <1, 0>。

以上一个字符表示一个 16 进制数，两个 16 进制数其实代表的其实就是一个字节。每个空格分割的都是两个字节（1 字节 8 位）。然后逐个字节去分析 class 文件格式。

u1 代表 1 个字节，依次类推，u4 代表 4 字节

### class 文件格式

![](http://note-1255449501.file.myqcloud.com/2018-02-11-064150.png)

#### 魔数

第 1 行，cafe babe

#### 次版本号

第 1 行，0000，10 进制为 0

#### 主版本号

第 1 行，0034，10 进制为 52。结合次版本号，最终 class 版本号是 52.0

#### 常量池常量数量

第 1 行，0016，10 进制为 22。

#### 常量池表（constant_pool_count & constant_pool）

![2018-02-11 at 3.09 PM](/var/folders/fd/ptrbg3sx0cv0k988y2qdqxnm0000gn/T/se.razola.Glui2/220A95B8-E822-4BBC-A22F-28BE92891D60-769-00005B6A513F5A86/2018-02-11 at 3.09 PM.png)

常量池表的索引从 1 开始，比如 constant_pool_count 为 22，索引表的下标范围是 1 - 21。0 是预留出来的（TODO）

从第 1 行 0a00 开始，0a 代表，依次：

| 索引 | 类型                                 | tag       |                                                              |                                        |
| ---- | ------------------------------------ | --------- | ------------------------------------------------------------ | -------------------------------------- |
| 1    | constant_methodref_info              | 0A        | 00 04：<br />00 12：                                         |                                        |
| 2    | constant_fieldref_info               | 09        | 00 03：<br />00 13：                                         |                                        |
| 3    | constant_class_info                  | 07        | 00 14：                                                      |                                        |
| 4    | constant_class_info                  | 07        | 00 15：                                                      |                                        |
| 5    | constant_utf8_info                   | 01        | 00 01：字节长度 length<br />6D：length 个字节，这里是 1 字节，utf8字符为「m」 | m                                      |
| 6    | constant_utf8_info                   | 01        | 00 01：<br />49：length 个字节，这里是 1 字节，utf8字符为「I」 | I                                      |
| 7    | constant_utf8_info                   | 01        | 00 06：<br />3c：<<br />69：i<br />6e：n <br />69：i<br />74：t <br />3e：> | <init>                                 |
| 8    | constant_utf8_info                   | 01        | 00 03：<br />28：(<br />29：)<br />56：V                     | ()V                                    |
| 9    | constant_utf8_info                   | 01        | 0004<br />43<br />6f<br />64<br />65                         | Code                                   |
| 10   | constant_utf8_info                   | 01        | 00 0f<br />4c<br />69<br />6e<br />65<br />4e<br />75<br />6d<br />62<br />65<br />72<br />54<br />61<br />62<br />6c<br />65 | LineNumberTable                        |
| 11   | constant_utf8_infoconstant_utf8_info | 01        | 00 12：18 个字节<br />4c<br />6f<br />63<br />61<br />6c<br />56<br />61<br />72<br />69<br />61<br />62<br />6c<br />65<br />54<br />61<br />62<br />6c<br />65 | LocalVariableTable                     |
| 12   | constant_utf8_info                   | 01        | 00 04<br /><br />74 68 69 73                                 |                                        |
| 13   | constant_utf8_info                   | 01        | 00 26：38<br />4c 63 6f 6d 2f 66 6f<br />6f 74 6d 61 6e 66 66 2f 6a 64 6b 74 65 73 74 2f<br />62 61 73 65 2f 54 65 73 74 43 6c 61 73 73 3b | Lcom/footmanff/jdktest/base/TestClass; |
| 14   | constant_utf8_info                   | 01<9， f> | 00 03<br />69 6e 63                                          | inc                                    |
| 15   | constant_utf8_info                   | 01        | 00 03<br />28 29 49                                          | ()I                                    |
| 16   | constant_utf8_info                   | 01        | 00 0a：10<br />53 6f<br />75 72 63 65 46 69 6c 65            | SourceFile                             |
| 17   | constant_utf8_info                   | 01        | 00 0e：14<br />54 65 73 74 43<br />6c 61 73 73 2e 6a 61 76 61 | TestClass.java                         |
| 18   | constant_nameAndType_info            | 0c        | 00 07<br />00 08                                             |                                        |
| 19   | constant_nameAndType_info            | 0c        | 00 05<br />00 06                                             |                                        |
| 20   | constant_utf8_info                   | 01        | 00 24：36<br /><br />63 6f 6d 2f 66 6f 6f 74 6d 61<br />6e 66 66 2f 6a 64 6b 74 65 73 74 2f 62 61 73 65<br />2f 54 65 73 74 43 6c 61 73 73 | com/footmanff/jdktest/base/TestClass   |
| 21   | constant_utf8_info                   | 01        | 00 10：16<br />6a 61 76<br />61 2f 6c 61 6e 67 2f 4f 62 6a 65 63 74 | java/lang/Object                       |

#### access_flags

00 21

#### this_class

00 03

#### super_class

00 04

#### interfaces_count

00 00<17, 4>

#### interfaces

因为 interfaces_count 为 0，所以这里一个都没有

#### fields_count

00 01

#### fields

| access_flag        | name_index                 | descriptor_index                                             | attribute_count        | attributes |
| ------------------ | -------------------------- | ------------------------------------------------------------ | ---------------------- | ---------- |
| 00 02<br />private | 00 05<br />常量池索引 5：m | 00 06<br />常量池索引 6：I<br />I （基本类型 int 用大写的 i 代表） | 00 00<br />属性数量为0 | 无         |

#### method_count

00 02

#### methods

| access_flag | name_index                      | descriptor_index             | attribute_count     | attributes                                                   |
| ----------- | ------------------------------- | ---------------------------- | ------------------- | ------------------------------------------------------------ |
| 00 01       | 00 07<br />常量池索引 7：<init> | 00 08<br />常量池索引 8：()V | 00 01<br />1 个属性 | 00 09：常量池索引 9 -> Code<br />00 00 00 2f：47 字节的方法代码 |
|             |                                 |                              |                     |                                                              |

> <clinit>、<init> TODO

##### code ( init )

code 内存的是方法的字节码，从 <18, 9> 到 <21, 7>，连 attribute_name_index 和 attribute_length 一共 47 字节，所以字节码部分一共 41 字节，如下：

```
    0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F
18.                            00 09 00 00 00 2f 00 
19. 01 00 01 00 00 00 05 2a b7 00 01 b1 00 00 00 02 
20. 00 0a 00 00 00 06 00 01 00 00 00 06 00 0b 00 00 
21. 00 0c 00 01 00 00 00 05
```

属性表结构：

![](http://note-1255449501.file.myqcloud.com/2018-02-12-075049.png)

Code 属性：

![](http://note-1255449501.file.myqcloud.com/2018-02-12-074430.png)

##### max_stack

是操作数栈的最大深度 00 01

##### max_locals

局部变量表需要空间，单位 slot，00 01

##### code_length

字节码的字节长度 00 00 00 05

##### code

字节码的二进制流，每个指令是一个字节，2a b7 00 01 b1

##### exception_table_length

异常表长度

##### exception_table

异常表

##### attributes_count

##### attributes
