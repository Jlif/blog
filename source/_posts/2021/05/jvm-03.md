---
title: jvm-03
categories: JVM
abbrlink: f7703578
date: 2021-05-02 00:29:21
---

# JVM 相关总结（三）

## GC 日志解读与分析

### 示例代码

```java

import java.util.Random;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.LongAdder;

/*
演示 GC 日志生成与解读
*/
public class GCLogAnalysis {
    private static Random random = new Random();
<!-- more -->
    public static void main(String[] args) {
        // 当前毫秒时间戳
        long startMillis = System.currentTimeMillis();
        // 持续运行毫秒数；可根据需要进行修改
        long timeoutMillis = TimeUnit.SECONDS.toMillis(1);
        // 结束时间戳
        long endMillis = startMillis + timeoutMillis;
        LongAdder counter = new LongAdder();
        System.out.println("正在执行。..");
        // 缓存一部分对象；进入老年代
        int cacheSize = 2000;
        Object[] cachedGarbage = new Object[cacheSize];
        // 在此时间范围内，持续循环
        while (System.currentTimeMillis() < endMillis) {
            // 生成垃圾对象
            Object garbage = generateGarbage(100 * 1024);
            counter.increment();
            int randomIndex = random.nextInt(2 * cacheSize);
            if (randomIndex < cacheSize) {
                cachedGarbage[randomIndex] = garbage;
            }
        }
        System.out.println("执行结束！共生成对象次数：" + counter.longValue());
    }

    // 生成对象
    private static Object generateGarbage(int max) {
        int randomSize = random.nextInt(max);
        int type = randomSize % 4;
        Object result = null;
        switch (type) {
            case 0:
                result = new int[randomSize];
                break;
            case 1:
                result = new byte[randomSize];
                break;
            case 2:
                result = new double[randomSize];
                break;
            default:
                StringBuilder builder = new StringBuilder();
                String randomString = "randomString-Anything";
                while (builder.length() < randomSize) {
                    builder.append(randomString);
                    builder.append(max);
                    builder.append(randomSize);
                }
                result = builder.toString();
                break;
        }
        return result;
    }
}
```

### 串行/并行/G1 案例演练

#### 串行

> java -XX:+UseSerialGC -Xloggc:gc.log -Xms128m -Xmx128m -XX:+PrintGCDetails -XX:+PrintGCDateStamps GCLogAnalysis

> java -XX:+UseSerialGC -Xloggc:gc.log -Xms512m -Xmx512m -XX:+PrintGCDetails -XX:+PrintGCDateStamps GCLogAnalysis

> java -XX:+UseSerialGC -Xloggc:gc.log -Xms4g -Xmx4g -XX:+PrintGCDetails -XX:+PrintGCDateStamps GCLogAnalysis

#### 并行

> java -XX:+UseParallelGC -Xloggc:gc.log -Xms128m -Xmx128m -XX:+PrintGCDetails -XX:+PrintGCDateStamps GCLogAnalysis

> java -XX:+UseParallelGC -Xloggc:gc.log -Xms512m -Xmx512m -XX:+PrintGCDetails -XX:+PrintGCDateStamps GCLogAnalysis

> java -XX:+UseParallelGC -Xloggc:gc.log -Xms4g -Xmx4g -XX:+PrintGCDetails -XX:+PrintGCDateStamps GCLogAnalysis

#### CMS

> java -XX:+UseConcMarkSweepGC -Xloggc:gc.log -Xms128m -Xmx128m -XX:+PrintGCDetails -XX:+PrintGCDateStamps GCLogAnalysis

> java -XX:+UseConcMarkSweepGC -Xloggc:gc.log -Xms512m -Xmx512m -XX:+PrintGCDetails -XX:+PrintGCDateStamps GCLogAnalysis

> java -XX:+UseConcMarkSweepGC -Xloggc:gc.log -Xms4g -Xmx4g -XX:+PrintGCDetails -XX:+PrintGCDateStamps GCLogAnalysis

#### G1

> java -XX:+UseG1GC -Xloggc:gc.log -Xms128m -Xmx128m -XX:+PrintGCDetails -XX:+PrintGCDateStamps GCLogAnalysis

> java -XX:+UseG1GC -Xloggc:gc.log -Xms512m -Xmx512m -XX:+PrintGCDetails -XX:+PrintGCDateStamps GCLogAnalysis

> java -XX:+UseG1GC -Xloggc:gc.log -Xms4g -Xmx4g -XX:+PrintGCDetails -XX:+PrintGCDateStamps GCLogAnalysis

#### 汇总表格

| 堆内存大小 | Serial GC | Parallel GC | CMS GC | G1 GC |
| :--- | :--- | :--- | :--- | :--- |
| 128M | OOM（26 次 GC，20 次 Full GC） | OOM（27 次 GC，17 次 Full GC） |  OOM（24 次 GC，13 次 Full GC） | OOM（34 次 GC，11 次 Full GC） |
| 512M | 6837（13 次 GC，4 次 Full GC） | 6933（28 次 GC，4 次 Full GC） |  9023（29 次 GC，12 次 Full GC）  | 9232（47 次 GC，21 次 Full GC） |
| 4G | 8505（2 次 GC,0 次 Full GC） | 10228（2 次 GC,0 次 Full GC） |  11363（11 次 GC，0 次 Full GC）  | 11809（14 次 GC，0 次 Full GC） |

#### 总结

总体来看，当堆内存越大，GC 越不频繁，性能越好。当然，因为示例代码存在内存泄露的问题，只要时间够久，也总会 OOM 的。

至于并行 GC 在小内存的时候并没有表现出我想象中的性能优势，我的猜想是因为内存泄漏，每次 GC 其实都占用了不少时间，但其实越往后，能够回收的内存越少。因为 CMS 和 G1 对暂停时间有一定的控制，除非回收次数显著高于并行 GC，不然在示例代码中并行 GC 的优势并不明显。还有就是示例代码执行时间不够长，只有一秒钟，如果时间足够长，并行 GC 在吞吐量方面应该是有优势的。

在生产中，一般堆比较小的时候还是采用并行 GC 回收器，大内存的时候采用 G1 回收器，如果对延迟有特殊的要求，可以考虑 ZGC。

## JVM 线程堆栈数据分析

### JVM 内部线程种类

• VM 线程：单例的 VMThread 对象，负责执行 VM 操作，下文将对此进行讨论；

• 定时任务线程：单例的 WatcherThread 对象， 模拟在 VM 中执行定时操作的计时器中断；

• GC 线程：垃圾收集器中，用于支持并行和并发垃圾回收的线程；

• 编译器线程： 将字节码编译为本地机器代码；

• 信号分发线程：等待进程指示的信号，并将其分配给 Java 级别的信号处理方法。

### 安全点

1. 方法代码中被植入的安全点检测入口；

2. 线程处于安全点状态：线程暂停执行，这个时候线程栈不再发生改变；

3. JVM 的安全点状态：所有线程都处于安全点状态。

### JVM 支持多种方式来进行线程转储

1. JDK 工具，包括：jstack 工具，jcmd 工具，jconsole, jvisualvm, Java Mission Control 等；

2. Shell 命令或者系统控制台，比如 Linux 的 kill -3,Windows 的 Ctrl + Break 等；

3. JMX 技术， 主要是使用 ThreadMxBean。

## 内存分析与相关工具

### 思考

一个对象具有 100 个属性，与 100 个对象每个具有 1 个属性，哪个占用的内存空间更大？

类比一百个箱子，每个箱子装一本书；一个箱子，装一百本书。

### 一个 Java 对象占用多少内存？

可以使用 Instrumentation.getObjectSize() 方法来估算一个对象占用的内存空间。

JOL (Java Object Layout) 可以用来查看对象内存布局。

### 对象头和对象引用

在 64 位 JVM 中，对象头占据的空间是 12-byte(=96bit=64+32)，但是以 8 字节对齐，所以一个空类的实例至少占用 16 字节。

在 32 位 JVM 中，对象头占 8 个字节，以 4 的倍数对齐 (32=4*8)。

所以 new 出来很多简单对象，甚至是 new Object()，都会占用不少内容哈。

通常在 32 位 JVM，以及内存小于 -Xmx32G 的 64 位 JVM 上（默认开启指针压缩），一个引用占的内存默认是 4 个字节。

因此，64 位 JVM 一般需要多消耗堆内存。

#### 包装类型

包装类型比原生数据类型消耗的内存要多：

Integer：占用 16 字节 (8+4=12+补齐），因为 int 部分占 4 个字节。所以使用 Integer 比原生类型 int 要多消耗 300% 的内存。

Long：一般占用 16 个字节 (8+8=16)，当然，对象的实际大小由底层平台的内存对齐确定，具体由特定 CPU 平台的 JVM 实现决定。看起来一个 Long 类型的对象，比起原生类型 long 多占用了 8 个字节（也多消耗了 100%）。

#### 多维数组

在二维数组`int[dim1][dim2]` 中，每个嵌套的数组 int[dim2] 都是一个单独的 Object，会额外占用 16 字节的空间。当数组维度更大时，这种开销特别明显。

`int[128][2]` 实例占用 3600 字节。而 int[256] 实例则只占用 1040 字节。里面的有效存储空间是一样的，3600 比起 1040 多了 246%的额外开销。在极端情况下，`byte[256][1]`，额外开销的比例是 19 倍！

#### String

String 对象的空间随着内部字符数组的增长而增长。当然，String 类的对象有 24 个字节的额外开销。

对于 10 字符以内的非空 String，增加的开销比起有效载荷（每个字符 2 字节+ 4 个字节的 length），多占用了 100% 到 400% 的内存。

#### 对齐是绕不过去的问题

我们可能会认为，一个 X 类的实例占用 17 字节的空间。但是由于需要对齐 (padding)，JVM 分配的内存是 8 字节的整数倍，所以占用的空间不是 17 字节，而是 24 字节。

### 常见异常

#### OutOfMemoryError: Java heap space

创建新的对象时，堆内存中的空间不足以存放新创建的对象

产生的原因，很多时候就类似于将 XXL 号的对象，往 S 号的 Java heap space 里面塞。其实清楚了原因，问题就很容易解决了：只要增加堆内存的大小，程序就能正常运行。

另外还有一些情况是由代码问题导致的：

• 超出预期的访问量/数据量：应用系统设计时，一般是有“容量”定义的，部署这么多机器，用来处理一定流量的数据/业务。如果访问量突然飙升，超过预期的阈值，类似于时间坐标系中针尖形状的图谱。那么在峰值所在的时间段，程序很可能就会卡死、并触发`java.lang.OutOfMemoryError: Java heap space`错误。

• 内存泄露 (Memory leak)：这也是一种经常出现的情形。由于代码中的某些隐蔽错误，导致系统占用的内存越来越多。如果某个方法/某段代码存在内存泄漏，每执行一次，就会（有更多的垃圾对象）占用更多的内存。随着运行时间的推移，泄漏的对象耗光了堆中的所有内存，那么`java.lang.OutOfMemoryError: Java heap space` 错误就爆发了。

#### OutOfMemoryError: PermGen space/OutOfMemoryError: Metaspace

java.lang.OutOfMemoryError: PermGen space 的主要原因，是加载到内存中的

class 数量太多或体积太大，超过了 PermGen 区的大小。

解决办法：增大 PermGen/Metaspace

-XX:MaxPermSize=512m

-XX:MaxMetaspaceSize=512m

高版本 JVM 也可以：

-XX:+CMSClassUnloadingEnabled

#### OutOfMemoryError: Unable to create new native thread

java.lang.OutOfMemoryError: Unable to create new native thread 错误是程序创建的线程数量已达到上限值的异常信息。

解决思路：

1. 调整系统参数 ulimit -a，echo 120000 > /proc/sys/kernel/threads-max

2. 降低 xss 等参数

3. 调整代码，改变线程创建和使用方式

### 内存 dump 分析工具

- Eclipse MAT
- jhat

## JVM 问题分析调优经验

### 1. 高分配速率

分配速率 (Allocation rate) 表示单位时间内分配的内存量。通常使用 MB/sec 作为单位。上一次垃圾收集之后，与下一次 GC 开始之前的年轻代使用量，两者的差值除以时间，就是分配速率。

分配速率过高就会严重影响程序的性能，在 JVM 中可能会导致巨大的 GC 开销。

正常系统：分配速率较低 ~ 回收速率 -> 健康

内存泄漏：分配速率持续大于回收速率 -> OOM

性能劣化：分配速率较高 ~ 回收速率 -> 亚健康

在某些情况下，只要增加年轻代的大小，即可降低分配速率过高所造成的影响。增加年轻代空间并不会降低分配速率，但是会减少 GC 的频率。如果每次 GC 后只有少量对象存活，minor GC 的暂停时间就不会明显增加。

### 2. 过早提升

提升速率（promotion rate）用于衡量单位时间内从年轻代提升到老年代的数据量。一般使用 MB/sec 作为单位，和分配速率类似。

JVM 会将长时间存活的对象从年轻代提升到老年代。根据分代假设，可能存在一种情况，老年代中不仅有存活时间长的对象，也可能有存活时间短的对象。这就是过早提升：对象存活时间还不够长的时候就被提升到了老年代。

major GC 不是为频繁回收而设计的，但 major GC 现在也要清理这些生命短暂的对象，就会导致 GC 暂停时间过长。这会严重影响系统的吞吐量。

GC 之前和之后的年轻代使用量以及堆内存使用量。这样就可以通过差值算出老年代的使用量。

和分配速率一样，提升速率也会影响 GC 暂停的频率。但分配速率主要影响 minor GC，而提升速率则影响 major GC 的频率。

有大量的对象提升，自然很快将老年代填满。老年代填充的越快，则 major GC 事件的频率就会越高。

一般来说过早提升的症状表现为以下形式：

1. 短时间内频繁地执行 full GC

2. 每次 full GC 后老年代的使用率都很低，在 10-20%或以下

3. 提升速率接近于分配速率

解决这类问题，需要让年轻代存放得下暂存的数据，有两种简单的方法：

一是增加年轻代的大小，设置 JVM 启动参数，类似这样：-Xmx64m -XX:NewSize=32m，程序在执行时，Full GC 的次数自然会减少很多，只会对 minor GC 的持续时间产生影响。

二是减少每次批处理的数量，也能得到类似的结果。

至于选用哪个方案，要根据业务需求决定。在某些情况下，业务逻辑不允许减少批处理的数量，那就只能增加堆内存，或者重新指定年轻代的大小。如果都不可行，就只能优化数据结构，减少内存消耗。

但总体目标依然是一致的：让临时数据能够在年轻代存放得下。

## GC 疑难情况问题分析

1、查询业务日志，可以发现这类问题：请求压力大，波峰，遭遇降级，熔断等等，基础服务、外部 API 依赖。

2、查看系统资源和监控信息：

硬件信息、操作系统平台、系统架构；

排查 CPU 负载、内存不足，磁盘使用量、硬件故障、磁盘分区用满、IO 等待、IO 密集、丢数据、并发竞争等情况；

排查网络：流量打满，响应超时，无响应，DNS 问题，网络抖动，防火墙问题，物理故障，网络参数调整、超时、连接数。

3、查看性能指标，包括实时监控、历史数据。可以发现假死，卡顿、响应变慢等现象；

排查数据库，并发连接数、慢查询、索引、磁盘空间使用量、内存使用量、网络带宽、死锁、TPS、查询

数据量、redo 日志、undo、binlog 日志、代理、工具 BUG。可以考虑的优化包括：集群、主备、只读实例、分片、分区；

大数据，中间件，JVM 参数。

4、排查系统日志，比如重启、崩溃、Kill。

5、APM，比如发现有些链路请求变慢等等。

6、排查应用系统

排查配置文件：启动参数配置、Spring 配置、JVM 监控参数、数据库参数、Log 参数、APM 配置、内存问题，比如是否存在内存泄漏，内存溢出、批处理导致的内存放大、GC 问题等等；

GC 问题，确定 GC 算法、确定 GC 的 KPI，GC 总耗时、GC 最大暂停时间、分析 GC 日志和监控指标：内存分配速度，分代提升速度，内存使用率等数据。适当时修改内存配置；

排查线程，理解线程状态、并发线程数，线程 Dump，锁资源、锁等待，死锁；

排查代码，比如安全漏洞、低效代码、算法优化、存储优化、架构调整、重构、解决业务代码 BUG、第三方库、XSS、CORS、正则；

单元测试：覆盖率、边界值、Mock 测试、集成测试。

7、排除资源竞争、坏邻居效应

8、疑难问题排查分析手段

DUMP 线程/内存；

抽样分析/调整代码、异步化、削峰填谷。