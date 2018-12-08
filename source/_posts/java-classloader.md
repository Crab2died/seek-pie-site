---
layout: post
title:  "Java Classloader"
date:   2018-06-25 09:15:27
author: Crab2Died
categories: java
tags: 
  - Java 
  - Classloader
---

## 1. 类加载时机
### 1.1 - 生命周期
```
                      +-----------------------连接(Linking)---------------------------+
    加载(Loading) ——> |  验证(Verification) ——> 准备(Preparation) ——> 解析(Resolution) |
                      +---------------------------------------------------------------+
                                                                            ↓
                                卸载(Unloading) <—— 使用(Using) <—— 初始化(Initialization)
```

### 1.2 - 立即初始化(主动引用)  
   1. 遇到`new`、 `getstatic`、 `putstatic`或`invokestatic`这4条字节码指令时
   2. 使用`java.lang.reflect`包的方法对类进行反射调用的时候,如果类没有进行过初始化,则需要先触发其初始化.
   3. 当初始化一个类的时候,如果发现其父类还没有进行过初始化,则需要先触发其父类的初始化.
   4. 当虚拟机启动时,用户需要指定一个要执行的主类(包含`main()`方法的那个类),虚拟机会先初始化这个主类.
   5. 当使用JDK 1.7的动态语言支持时,如果一个`java.lang.invoke.MethodHandle`实例最后的解析结果**REF_getStatic**、
      **REF_putStatic**、**REF_invokeStatic**的方法句柄,并且这个方法句柄所对应的类没有进行过初始化,则需要先触发其初始化.
      
### 1.3 - 被动加载
   1. 子类调用父类静态方法,子类不会被初始化
   2. 引用类型定义不初始化加载
   3. 常量调用不会初始化类(编译期已进入常量池)
   
### 1.4 - 接口初始化
   &emsp;&emsp;当一个类在初始化时,要求其父类全部都已经初始化过了,但是一个接口在初始化时,并不要求其父接口全部都完成了初始化,
   只有在真正使用到父接口的时候(如引用接口中定义的常量)才会初始化.

## 2. 类加载过程
### 2.2 - 加载
   1. 通过一个类的全限定名来获取定义此类的二进制字节流(这一条玩出了很多技术,如:jsp(从其他文件中生成),Applet(从网络中获取),Proxy(运行时计算)).
   2. 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构.
   3. 在内存中生成一个代表这个类的`java.lang.Class`对象,作为方法区这个类的各种数据的访问入口.  
   **_
   注:
   加载阶段完成后,虚拟机外部的二进制字节流就按照虚拟机所需的格式存储在方法区之中,方法区中的数据存储格式由虚拟机实现自行定义,
   虚拟机规范未规定此区域的具体数据结构. 然后在内存中实例化一个`java.lang.Class`类的对象(并没有明确规定是在Java堆中,
   对于HotSpot虚拟机而言,Class对象比较特殊,它虽然是对象,但是存放在方法区里面),
   这个对象将作为程序访问方法区中的这些类型数据的外部接口.
   _**

### 2.3 - 验证
   &emsp;&emsp;确保Class文件的字节流中包含的信息符合当前虚拟机的要求,并且不会危害虚拟机自身的安全.
   `-Xverify:none`
   参数来关闭大部分的类验证措施,以缩短虚拟机类加载的时间.  
   1.  文件格式验证
        - 魔数**0xCAFEBABE**验证
        - 主、次版本号验证
        - 常量池常量验证
        - UTF-8编码验证
        - ...
   2.  元数据验证
        - 这个类是否有父类
        - 该类是否继承了不能继承的类(如:`final类`)
        - ...
   3.  字节码验证  
       第三阶段是整个验证过程中最复杂的一个阶段,主要目的是通过数据流和控制流分析,确定程序语义是合法的、 符合逻辑的.
        - 保证跳转指令不会跳转到方法体以外的字节码指令上.
        - 保证方法体中的类型转换是有效的
        - ...
   4.  符号引用验证
        - 符号引用中通过字符串描述的全限定名是否能找到对应的类.
        - 在指定类中是否存在符合方法的字段描述符以及简单名称所描述的方法和字段.
        - 符号引用中的类、 字段、 方法的访问性(`private`、 `protected`、 `public`、 `default`)是否可被当前类访问
          (不通过将抛一个IncompatibleClassChangeError异常子类).
        - ...

### 2-4 - 准备
   &emsp;&emsp;准备阶段是正式为类变量分配内存并设置类变量初始值的阶段,这些变量所使用的内存都将在方法区中进行分配.
   注:内存分配仅包括类变量(static修饰),不包括实例变量;初始值通常是指零值(如:`public static int v = 123;` 初始值v为0而非123,
   但如`public static final int v = 123;`时将被初始化为123)

### 2-5 - 解析
   &emsp;&emsp;解析阶段是虚拟机将常量池内的符号引用替换为直接引用的过程
   - 符号引用(Symbolic References)
   - 直接引用(Direct References)
   - 虚拟机对除invokedynamic(动态调用点限定符)指令外的操作进行缓存
   - 解析动作主要针对类或接口、 字段、 类方法、 接口方法、 方法类型、 方法句柄和调用点限定符7类符号引用进行,
     分别对应于常量池的**CONSTANT_Class_info**、**CONSTANT_Fieldref_info**、 **CONSTANT_Methodref_info**、
     **CONSTANT_InterfaceMethodref_info**、 **CONSTANT_MethodType_info**、**CONSTANT_MethodHandle_info**
     和**CONSTANT_InvokeDynamic_info** 7种常量类型
     * 1)类或接口解析
     * 2)字段解析
     * 3)类方法解析
     * 4)接口方法解析

### 2-6 - 初始化
   &emsp;&emsp;类加载的最后一步,真正执行java的字节码`.<clinit>()`方法保证在子类执行之前父类的`<clinit>()`已执行完毕.虚拟机中第一个被执
   行的`<clinit>()`方法的类肯定是`java.lang.Object.<clinit>()`虚拟机保证只执行一次,多线程时只会有一条线程执行,其他线程阻塞等待.

### 2-7 - 卸载

## 3. 类加载器
   &emsp;&emsp;**描述**  
   &emsp;&emsp;通过一个类的全限定名来获取描述此类的二进制字节流
   应用:类层次划分、热部署、OSGi(面向Java的动态模型系统)、代码加密等

### 3-1 - 类与类加载器
   &emsp;&emsp;相同的类被不同的类加载器加载他们必不相等.

### 3-2 - 双亲委派模型(Parents Delegation Model)
   &emsp;&emsp;**类加载器**  
   1. 启动类加载器(Bootstrap ClassLoader)C++实现、虚拟机一部分
      - 又称 引导类加载器
      - 按名识别,如:rt.jar
      - 负责加载&lt;JAVA_HOME&gt;\lib目录或被参数`-Xbootclasspth`指定的目录
   2. 其他类加载器, java实现,独立与虚拟机外部,全部继承抽象类java.lang.ClassLoader
      - 扩展类加载器(Extension ClassLoader)
        * 由sun.misc.Launcher$ExtClassLoader类实现
        * 负责加载&lt;JAVA_HOME&gt;\lib\ext目录或参数`-Djava.ext.dirs`所指定目录下的类
      - 应用程序类加载器(Application ClassLoader)
        * 又称 系统类加载器
        * 由sun.misc.Launcher$AppClassLoader实现
        * 它负责加载用户类路径(ClassPath)上所指定的类库,开发者可以直接使用这个类加载器,是程序中默认的类加载器  

   双亲委派模型图
   ```
                                                     自定义类加载器(CustomClassLoader)
                                                    //
    启动类加载器 <= 扩展类加载器 <= 应用程序加载器 <==
                                                    \\
                                                     自定义类加载器(CustomClassLoader)
   ```
   &emsp;&emsp;双亲委派模型的工作过程是:如果一个类加载器收到了类加载的请求,它首先不会自己去尝试加载这个类,而
   是把这个请求委派给父类加载器去完成,每一个层次的类加载器都是如此,因此所有的加载请求最终都应该传送到顶层的启动类加载器中,
   只有当父加载器反馈自己无法完成这个加载请求(它的搜索范围中没有找到所需的类)时,子加载器才会尝试自己去加载

### 3-3 - 破坏双亲委派模型

## 4. 问题
### 4-1 - 为什么要用双亲委派类加载
   &emsp;&emsp;双亲委派模型可以防止内存中出现多分同样的字节码，如果没有双亲委派模型的话如果用户编写了一个`java.lang.Object`的同
   名类放在classpath中，多个类加载器去加载这个类到内存中系统会出现多个不同的Object类，那么类之间的比较结果及类的唯一性将无法
   保证，而且如果不使用这种模型将给虚拟机带来安全隐患。