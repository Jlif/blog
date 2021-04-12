---
title: JVM相关总结（一）
abbrlink: '197e5454'
date: 2021-04-13 00:30:50
categories: JVM
---

# JVM相关总结（一）

## Java字节码

### 什么是字节码？

`Java bytecode` 由单字节(byte)的指令组成，理论上最多支持256（2^8=256）个操作码。实际上Java只使用了200左右的操作码，还有一些操作码则保留给调试操作。

根据指令的性质，主要分为四个大类：

1. 栈操作指令，包括与局部变量交互的指令
2. 程序流控制指令
3. 对象操作指令，包括方法调用指令
4. 算数运算以及类型转换指令
<!-- more -->
### 生成字节码

编译：`javac Hello.java`

查看字节码：`javap -c Hello`

进一步：`javap -c -verbose Hello`

### 字节码的运行时结构

JVM 是一台基于栈的计算机器。每个线程都有一个独属于自己的线程栈（JVM Stack），用于存储栈帧（Frame）。

每一次方法调用，JVM 都会自动创建一个栈帧。

栈帧由操作数栈，局部变量数组以及一个 Class 引用组成。Class 引用指向当前方法在运行时常量池中对应的 Class。

## JVM类加载器

### 类的生命周期

1. 加载（Loading）：找Class文件
2. 验证（Verification）：验证格式、依赖
3. 准备（Preparation）：静态字段、方法表
4. 解析（Resolution）：符号解析为引用
5. 初始化（Initialization）：构造器、静态变量赋值、静态代码块
6. 使用（Using）
7. 卸载（Unloading）

### 类的加载时机

1. 当虚拟机启动时，初始化用户指定的主类，就是启动执行的 main 方法所在的类；
2. 当遇到用以新建目标类实例的 new 指令时，初始化 new 指令的目标类，就是 new 一个类的时候要初始化；
3. 当遇到调用静态方法的指令时，初始化该静态方法所在的类；
4. 当遇到访问静态字段的指令时，初始化该静态字段所在的类；
5. 子类的初始化会触发父类的初始化；
6. 如果一个接口定义了 default 方法，那么直接实现或者间接实现该接口的类的初始化，会触发该接口的初始化；
7. 使用反射 API 对某个类进行反射调用时，初始化这个类，其实跟前面一样，反射调用要么是已经有实例了，要么是静态方法，都需要初始化；
8. 当初次调用 MethodHandle 实例时，初始化该 MethodHandle 指向的方法所在的类。

### 不会初始化（可能会加载）

1. 通过子类引用父类的静态字段，只会触发父类的初始化，而不会触发子类的初始化。
2. 定义对象数组，不会触发该类的初始化。
3. 常量在编译期间会存入调用类的常量池中，本质上并没有直接引用定义常量的类，不会触发定义常量所在的类。
4. 通过类名获取 Class 对象，不会触发类的初始化，Hello.class 不会让 Hello 类初始化。
5. 通过 Class.forName 加载指定类时，如果指定参数 initialize 为 false 时，也不会触发类初始化，其实这个参数是告诉虚拟机，是否要对类进行初始化。Class.forName（“jvm.Hello”）默认会加载 Hello 类。
6. 通过 ClassLoader 默认的 loadClass 方法，也不会触发初始化动作（加载了，但是不初始化）。

### 三类加载器

1. 启动类加载器（BootstrapClassLoader）
2. 扩展类加载器（ExtClassLoader）
3. 应用类加载器（AppClassLoader）

#### 加载器特点

1. 双亲委托
2. 负责依赖
3. 缓存加载

#### 为啥双亲委托？

1. 沙箱安全机制：防止核心库被随意篡改
2. 避免类的重复加载：当父加载器已经加载该类，不需要子加载器再加载一次

### 添加引用类的几种方式

1、放到 JDK 的 lib/ext 下，或者-Djava.ext.dirs
2、 java –cp/classpath 或者 class 文件放到当前路径
3、自定义 ClassLoader 加载
4、拿到当前执行类的 ClassLoader，反射调用 addUrl 方法添加 Jar 或路径(JDK9 无效)

## JVM内存模型

### JVM内存结构

每个线程都只能访问自己的线程栈。每个线程都不能访问（看不见）其他线程的局部变量。

所有原生类型的局部变量都存储在线程栈中，因此对其他线程是不可见的。

线程可以将一个原生变量值的副本传给另一个线程，但不能共享原生局部变量本身。

堆内存中包含了 Java 代码中创建的所有对象，不管是哪个线程创建的。 其中也涵盖了包装类型（例如 Byte，Integer，Long 等）。

不管是创建一个对象并将其赋值给局部变量， 还是赋值给另一个对象的成员变量， 创建的对象都会被保存到堆内存中。

如果是原生数据类型的局部变量，那么它的内容就全部保留在线程栈上。

如果是对象引用，则栈中的局部变量槽位中保存着对象的引用地址，而实际的对象内容保存在堆中。

对象的成员变量与对象本身一起存储在堆上，不管成员变量的类型是原生数值，还是 对象引用。

类的静态变量则和类定义一样都保存在堆中。

### JVM内存整体结构

如图所示：

![JVM内存结构](https://img.jiangchen.tech/JVM%E5%86%85%E5%AD%98%E7%BB%93%E6%9E%84.png)

#### Tip：最大堆内存设置

最大堆内存设置一般为宿主机或者容器可用内存的60-70%为好，因为还有非堆、堆外内存等其他需要使用内存的区域，不然可能导致OOM等问题。

### CPU与内存行为

- CPU乱序执行
- volatile 关键字
- 原子性操作
- 内存屏障

#### 什么是 JMM？

JMM规范明确定义了不同的线程之间，通过哪些方式，在什么时候可以看见其它线程保存到共享变量中的值；以及在必要时，如何对共享变量的访问进行同步。这样的好处是屏蔽各种硬件平台和操作系统之间的内存访问差异，实现了 Java 并发程序真正的跨平台。

## JVM 启动参数

### JVM 启动参数简介

以 - 开头为标准参数，所有的 JVM 都要实现这些参数，并且向后兼容。（如 -server）

-D 设置系统属性（如 -Dfile.encoding=UTF-8）

-X 非标准参数，基本都是传给 JVM 的，默认 JVM 实现这些参数的功能，但是并不保证所有 JVM 实现都满足，切不保证向后兼容。可以使用`java -X`命令来查看当前 JVM 支持的非标准参数。（如 -Xmx8g）

–XX：开头为非稳定参数, 专门用于控制 JVM 的行为，跟具体的 JVM 实现有关，随时可能会在下个版本取消。

-XX：+-Flags 形式, +- 是对布尔值进行开关。（如 -XX:+UseG1GC，加减开关，+表示打开，-表示关闭）

-XX：key=value 形式, 指定某个选项的值。（如 -XX:MaxPermSize=256m）

### JVM 启动参数类型

- 系统属性参数
- 运行模式参数
- 堆内存设置参数
- GC 设置参数
- 分析诊断参数
- JavaAgent 参数

## JDK 内置命令行工具

### JVM 命令行工具

- java：Java 应用的启动程序
- javac：JDK 内置的编译工具
- javap：反编译 class 文件的工具
- javadoc：根据 Java 代码和标准注释,自动生成相关的API说明文档
- javah：JNI 开发时, 根据 java 代码生成需要的 .h文件
- extcheck：检查某个 jar 文件和运行时扩展 jar 有没有版本冲突，很少使用
- jdb：Java Debugger ; 可以调试本地和远端程序, 属于 JPDA 中的一个 demo 实现, 供其他调试器参考。开发时很少使用
- jdeps：探测 class 或 jar 包需要的依赖
- jar：打包工具，可以将文件和目录打包成为 .jar 文件；.jar 文件本质上就是 zip 文件, 只是后缀不同。使用时按顺序对应好选项和参数即可
- keytool：安全证书和密钥的管理工具（支持生成、导入、导出等操作）
- jarsigner：JAR 文件签名和验证工具
- policytool：实际上这是一款图形界面工具, 管理本机的 Java 安全策略
- jps/jinfo：查看 java 进程
- jstat：查看 JVM 内部 gc 相关信息
- jmap：查看 heap 或类占用空间统计
- jstack：查看线程信息
- jcmd：执行 JVM 相关分析命令（整合命令）
- jrunscript/jjs：执行 js 命令

#### jstat

```shell
> jstat -options
-class 类加载(Class loader)信息统计
-compiler JIT 即时编译器相关的统计信息
-gc GC 相关的堆内存信息，用法: jstat -gc -h 10 -t 864 1s 20
-gccapacity 各个内存池分代空间的容量
-gccause 看上次 GC, 本次 GC（如果正在 GC中）的原因, 其他输出和 -gcutil 选项一致
-gcnew 年轻代的统计信息（New = Young = Eden + S0 + S1）
-gcnewcapacity 年轻代空间大小统计
-gcold 老年代和元数据区的行为统计
-gcoldcapacity old 空间大小统计
-gcmetacapacity meta 区大小统计
-gcutil GC 相关区域的使用率（utilization）统计
-printcompilation 打印 JVM 编译统计信息
```

如：

```shell
jstat -gcutil pid 1000 1000
```

每隔1秒，打印一千次

#### jmap

常用选项就 3 个:

-heap 打印堆内存（/内存池）的配置和使用信息

-histo 看哪些类占用的空间最多, 直方图

-dump:format=b,file=xxxx.hprof Dump 堆内存

如：

```shell
jmap -heap pid
jmap -histo pid
jmap -dump:format=b,file=3826.hprof 3826
```

#### jstack

-F 强制执行 thread dump. 可在 Java 进程卡死（hung 住）时使用, 此选项可能需要系统权限。

-m 混合模式(mixed mode),将 Java 帧和 native 帧一起输出, 此选项可能需要系统权限。

-l 长列表模式. 将线程相关的 locks 信息一起输出，比如持有的锁，等待的锁。

如：

```shell
jstack pid -l
```

### JVM 图形化工具

- jconsole
- jvisualvm
- VisualGC
- jmc