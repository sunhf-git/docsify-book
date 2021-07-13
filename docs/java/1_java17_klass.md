# java类在jvm中的存储介质klass

![java类在jvm中的存储介质klass](/images/code-1.jpg)

# 前言

**JVM几大模块**
- 类加载器子系统
- 内存模型
- 执行引擎
- 垃圾收集器
- JIT(热点代码缓存)

**klass模型是什么**
- java类在jvm中的存在形式
- java类-> c++的类 klass
- 非数组
   - InstanceKlass 普通的类在JVM中对应的c++类，存放类的元信息（变量、访问权限声明、方法、构造等声明,就是一个类结构中所有的信息），类被加载时存放在方法区
   - InstanceMirrorKlass 对应的是Class对象，当类被实例化时创建，存放在堆区
   - 面试题：class对象在在哪？类对象的定义又在哪？
- 数组
    - 基本类型数组
        - boolean、byte、char、short、int、float、long、double
        - 对应的Klass模型为TypeArrayKlass
    - 引用类型数组
        - 对应的Klass模型为ObjArrayKlass

**如何通过HSDB查看一个Java类对应的C++类**
- 非数组
    - 通过类向导
    - 通过类对象
- 数组
    - 数组是动态数据类型，是运行时创建的，类加载器中是不会有数组的元信息的。
    - 只能通过对象的class point指针在堆栈查找数组，堆栈中数组地址通常有一个 [I 标记



# Klass模型的继承结构

![Klass模型的继承结构](/images/java17/oop_klass.png)

**相关资料**
- [oop-Klass模型以及类加载原理](https://www.jianshu.com/p/ea491c150ebf)


## Java对象内存布局
![Java对象内存布局](/images/java17/java对象内存布局.png)

**相关资料**
- [Java对象内存布局](https://www.cnblogs.com/jajian/p/13681781.html)

## java17中的HSDB

**启动命令**
```shell
cd C:\Users\sunhf\Documents\jdk-jdk-17-25\build\windows-x86_64-server-release\jdk\bin
.\jhsdb hsdb
```

**相关资料**
- [hsdb工具查看运行时数据区](https://www.jianshu.com/p/f6f9b14d5f21)
- [jol工具分析Java对象大小](http://zhongmingmao.me/2016/07/01/jvm-jol-tutorial-1/)


# 其他资料
- [点击观看完整视频](https://www.bilibili.com/video/BV1PV411J7B1?p=2)