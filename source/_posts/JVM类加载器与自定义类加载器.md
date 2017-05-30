---
title: JVM类加载器与自定义类加载器
date: 2017-05-07 18:07:13
categories: [JVM,资料集]
tags: [类加载器]
---
<Excerpt in index | 首页摘要>
JVM类加载器与自定义类加载器的目的。<!-- more -->
<The rest of contents | 余下全文>
# JVM类加载器
## 什么是类的加载
类的加载是指将类的.class文件中的二进制数据读入内存中，将其放在运行时数据区的方法区内。
**JDK 8移除了永久代，并在Native Heap创建了元空间(Metaspace)，类也被加载在元空间。**
class文件被加载近内存后，JVM会在Java Heap创建1个`java.lang.Class`对象，Class对象封装了类在方法区内的数据结构，并且向Java程序员提供了访问方法区内的数据结构的接口。
## 类的生命周期
类的生命周期包括：
- 加载：查找并加载类的二进制数据，在Java Heap中创建一个`java.lang.Class`对象
- 连接：连接包含验证、准备、初始化。
    * 验证：文件格式、元数据、字节码、符号引用。
    * 准备：为类的静态变量分配内存，并将其初始化为默认值。
    * 解析：把类中的符号引用转换为直接引用。
- 初始化：为类的静态变量赋予正确的初始值。
- 使用：new出对象在程序中被使用。
- 卸载：执行垃圾回收。
如图所示：
![类的生命周期](/resources/img/jvm/类的生命周期.png)

## 类加载器
- 启动类加载器(Bootstrap ClassLoader)：负责加载存放在JAVA_HOME/jre/lib下，或被`-Xbootclasspath`参数指定的路径中的，并且能被虚拟机识别的类库。
- 扩展类加载器(Extension ClassLoader)：该加载器由`sun.misc.Launcher$ExtClassLoader`实现，负载加载存放在JAVA_HOME/jre/lib/ext下，或被`java.ext.dirs`系统变量指定的路径中的所有类库。
- 应用程序类加载器(Application ClassLoader)：该类加载器由`sun.misc.Launcher$AppClassLoader`来实现，它负责加载用户类路径`ClassPath`所指定的类。
![类加载器](/resources/img/jvm/类加载器.png)

## 类加载机制
- 全盘负责：当一个类加载器负责加载某个Class时，该Class所依赖的和引用的其他Class也将由该类加载器负责载入，除非显示使用另外一个类加载器来载入。
- 父类委托(双亲委托)：先让父类加载器试图加载该类，只有在父类加载器无法加载该类时才尝试从自己的类路径中加载该类。
- 缓存机制：缓存机制将会保证所有加载过的Class都会被缓存，当程序中需要使用某个Class时，类加载器先从缓存区寻找该Class，只有缓存区不存在，系统才会读取该类对应的二进制数据，并将其转换成Class对象，存入缓存区。这就是为什么修改了Class后，必须重启JVM，程序的修改才会生效。

## 双亲委托模型
JVM规范推荐使用双亲委托的类加载机制，即使是重写类加载器也应遵循此规范（不强制）。
类加载器保证了类在JVM的唯一性。
符合以下3点的2个类才是同类型的类实例，否则类型强制转换的时候会抛出异常。
- 两个类来自同一个Class文件
- 两个类是由同一个虚拟机加载
- 两个类是由同一个类加载器加载
![双亲委派加载机制](/resources/img/jvm/双亲委派加载机制.png)

# Tomcat的类加载器
- 为什么Tomcat需要重写类加载器？
> 因为一个Tomcat容器可以部署多个应用，而每个应用引用的jar各不相同，重写自定义加载器可以区分不同的jar引用，防止jar包冲突。类加载器保证了类在JVM的唯一性。



# Spring Boot的类加载器
- 为什么Spring Boot需要重写类加载器？
> Spring Boot的FatJar的打包方式打破了Java加载jar包的约定，所以需要重写类加载器去加载class文件以及类库。（详细可以观察Spring Boot工程打包后的目录结构）

# 引用
[JVM（8）：JVM知识点总览-高级Java工程师面试必备](http://www.importnew.com/23792.html)
[JVM类加载机制详解（二）类加载器与双亲委派模型](http://blog.csdn.net/zhangliangzi/article/details/51338291)
