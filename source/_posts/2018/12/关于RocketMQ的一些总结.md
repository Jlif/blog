---
layout: "post"
title: "使用 RocketMQ 的一些总结"
date: "2018-12-19"
categories: 消息中间件
---

# 前言

最近开始做一个物联网项目，设备（终端）会向服务器发送心跳包以及其他数据，服务端的框架我使用了 netty 以及 spring-boot，考虑到后续随着设备数量的增多，现在的这种同步的模式，后续肯定会产生性能瓶颈。大概就 netty，数据库，设备这几部分，其实 netty 性能我是不担心的，但是数据一多，数据库肯定是会成为性能瓶颈。现在我把数据库这块，也就是业务这块，跟 netty 分离，netty 只处理终端上传的数据，处理完之后直接丢到 mq 里面。业务模块消费 mq 消息，将终端上传的数据持久化到数据库，供后续业务调用。
<!-- more -->

# 简介

RocketMQ 是一款阿里巴巴开源的消息中间件，基于 java 开发。以下是 Github 上关于 RocketMQ 的介绍：

RcoketMQ 是一款低延迟、高可靠、可伸缩、易于使用的消息中间件。具有以下特性：

- 支持发布/订阅（Pub/Sub）和点对点（P2P）消息模型
- 在一个队列中可靠的先进先出（FIFO）和严格的顺序传递
- 支持拉（pull）和推（push）两种消息模式
- 单一队列百万消息的堆积能力
- 支持多种消息协议，如 JMS、MQTT 等
- 分布式高可用的部署架构，满足至少一次消息传递语义
- 提供 docker 镜像用于隔离测试和云集群部署
- 提供配置、指标和监控等功能丰富的 Dashboard

# RocketMQ 的下载安装以及启动

## 官网下载以及安装 RocketMQ

在 linux 上使用下载命令，如：
    wget http://mirror.bit.edu.cn/apache/rocketmq/4.3.2/rocketmq-all-4.3.2-source-release.zip

下载压缩包，下载下来解压之后，使用 maven 编译

```shell
  > unzip rocketmq-all-4.3.2-source-release.zip
  > cd rocketmq-all-4.3.2/
  > mvn -Prelease-all -DskipTests clean install -U
  > cd distribution/target/apache-rocketmq
```

文件夹内就是我们需要的 mq 服务端了

## 启动 name server

```shell
> nohup sh bin/mqnamesrv &
> tail -f ~/logs/rocketmqlogs/namesrv.log
  The Name Server boot success...
```

## 启动 broker

可以添加自动创建 topic 选项，但是生产环境中不建议开启这个选项，还是手动创建 topic 比较好，topic 对性能影响挺大的

```shell
> nohup sh bin/mqbroker -c conf/broker.conf &
> tail -f ~/logs/rocketmqlogs/broker.log
  The broker[%s, 172.30.30.233:10911] boot success...
```

## 关闭相关服务

```shell
> sh bin/mqshutdown broker
The mqbroker(36695) is running...
Send shutdown request to mqbroker(36695) OK

> sh bin/mqshutdown namesrv
The mqnamesrv(36664) is running...
Send shutdown request to mqnamesrv(36664) OK
```

> Tips: 如果启动不成功，可以查看相关日志，看是否是环境缺失导致，或者是否没有创建 topic

## 图形化管理工具

github 上面 apache 官方维护着一个开源的项目 [rocketmq-externals](https://github.com/apache/rocketmq-externals)，里面的`rocketmq-console`，是一个 Java Web 项目，可以通过图形化的界面来监控 RocketMQ 的状态，还是挺好用的，可以手动创建队列什么的，推荐使用。

# RocketMQ 与项目的集成

## spring-boot-starter-rocketmq

apache 官方有一个 RocketMQ-Spring 项目 ==> [传送门](https://github.com/apache/rocketmq-spring/blob/master/README_zh_CN.md)

但是本项目使用的是 github 上面开源的另一个项目，买好车他们团队开源出来的 [rocketmq-spring-boot-starter](https://github.com/maihaoche/rocketmq-spring-boot-starter)

## 简单入门实例

### 添加 maven 依赖

```xml
<dependency>
    <groupId>com.maihaoche</groupId>
    <artifactId>spring-boot-starter-rocketmq</artifactId>
    <version>0.1.0</version>
</dependency>
```

### 添加配置

```yml
spring:
    rocketmq:
      name-server-address: 172.21.10.111:9876
      # 可选，如果无需发送消息则忽略该配置
      producer-group: local_pufang_producer
      # 发送超时配置毫秒数，可选，默认 3000
      send-msg-timeout: 5000
      # 追溯消息具体消费情况的开关，默认打开
      #trace-enabled: false
      # 是否启用 VIP 通道，默认打开
      #vip-channel-enabled: false
```

### 程序入口添加注解开启自动装配

在 springboot 应用主入口添加`@EnableMQConfiguration`注解开启自动装配：

```java
@SpringBootApplication
@EnableMQConfiguration
class DemoApplication {
}
```

### 构建消息体

通过我们提供的 Builder 类创建消息对象，详见 wiki

```java
MessageBuilder.of(new MSG_POJO()).topic("some-msg-topic").build();
```

### 创建发送方

详见 wiki：

```java
@MQProducer
public class DemoProducer extends AbstractMQProducer{
}
```

### 创建消费方

详见 wiki： 支持`springEL`风格配置项解析，如存在`suclogger-test-cluster`配置项，会优先将 topic 解析为配置项对应的值。

```java
@MQConsumer(topic = "${suclogger-test-cluster}", consumerGroup = "local_sucloger_dev")
public class DemoConsumer extends AbstractMQPushConsumer {

    @Override
    public boolean process(Object message, Map extMap) {
        // extMap 中包含 messageExt 中的属性和 message.properties 中的属性
        System.out.println(message);
        return true;
    }
}
```

### 发送消息

```java
// 注入发送者
@Autowired
private DemoProducer demoProducer;

...

// 发送
demoProducer.syncSend(msg)
```