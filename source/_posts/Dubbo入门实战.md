---
title: Dubbo入门实战
date: 2019-04-19 13:18:20
tags: [分布式]
categories: 编程
---

## Dubbo简介

Dubbo是阿里巴巴SOA服务化治理方案的核心框架，每天为2,000+个服务提供3,000,000,000+次访问量支持，并被广泛应用于阿里巴巴集团的各成员站点。

Dubbo是一个分布式服务框架，致力于提供高性能和透明化的RPC远程服务调用方案，以及SOA服务治理方案。

文档：[http://dubbo.apache.org/zh-cn/index.html](http://dubbo.apache.org/zh-cn/index.html)

## RPC简介

RPC（Remote Procedure Call）—远程过程调用，它是一种通过网络从远程计算机程序上请求服务，而不需要了解底层网络技术的协议。RPC协议假定某些传输协议的存在，如TCP或UDP，为通信程序之间携带信息数据。在OSI网络通信模型中，RPC跨越了传输层和应用层。RPC使得开发包括网络分布式多程序在内的应用程序更加容易。
RPC采用客户机/服务器模式。请求程序就是一个客户机，而服务提供程序就是一个服务器。首先，客户机调用进程发送一个有进程参数的调用信息到服务进程，然后等待应答信息。在服务器端，进程保持睡眠状态直到调用信息到达为止。当一个调用信息到达，服务器获得进程参数，计算结果，发送答复信息，然后等待下一个调用信息，最后，客户端调用进程接收答复信息，获得进程结果，然后调用执行继续进行。
有多种 RPC模式和执行。最初由 Sun 公司提出。IETF ONC 宪章重新修订了 Sun 版本，使得 ONC RPC 协议成为 IETF 标准协议。现在使用最普遍的模式和执行是开放式软件基础的分布式计算环境（DCE）。

## Dubbo的作用

- 透明化的远程方法调用，就像调用本地方法一样调用远程方法，只需简单配置，没有任何API侵入。
- 软负载均衡及容错机制，可在内网替代F5等硬件负载均衡器，降低成本，减少单点。
- 服务自动注册与发现，不再需要写死服务提供方地址，注册中心基于接口名查询服务提供者的IP地址，并且能够平滑添加或删除服务提供者。

## Dubbo的流程
![/Dubbo入门实战/9005929-b630c970a0b2e8ae.jpg](/Dubbo入门实战/9005929-b630c970a0b2e8ae.jpg)

如图：

0：start。服务容器container启动，加载提供者provider。

1：registry。服务提供者向dubbo的注册中心注册自己提供的服务。

2：subscribe注册。消费者服务启动的时候向注册中心订阅自己需要的服务。

3：notify通知。注册中心将消费者所需要的provider地址返回给consumer，如果有变更，注册中心将会基于长连接将变更数据推送给消费者。

4：invoke调用。服务消费者，从提供者地址列表中，基于软负载均衡算法，选一台提供者进行调用，如果调用失败，再选另一台调用。

5：count。provider和consumer在内存中累计的调用次数和调用时间定时每分钟发送给监控中心。

## Zookeeper介绍

本Demo中的Dubbo注册中心采用的是Zookeeper。为什么采用Zookeeper呢？

Zookeeper是一个分布式的服务框架，是树型的目录服务的数据存储，能做到集群管理数据 ，这里能很好的作为Dubbo服务的注册中心。

Dubbo能与Zookeeper做到集群部署，当提供者出现断电等异常停机时，Zookeeper注册中心能自动删除提供者信息，当提供者重启时，能自动恢复注册数据，以及订阅请求。

## Dubbo实战

![](/Dubbo入门实战/微信截图_20190419135018.png)

服务提供者模块：dubbo-provider

消费者模块：dubbo-consumer

调用的服务接口共同的Api：dubbo-api

### 结构：
![](/Dubbo入门实战/微信截图_20190419135540.png)
dubbo-provider模块下的application.properties配置如下：

```
spring.application.name = dubbo-provider
server.port = 9090

#指定当前服务/应用的名字（同样的服务名字相同，不要和别的服务同名）
dubbo.application.name = dubbo-provider

demo.service.version = 1.0.0

dubbo.protocol.name = dubbo
dubbo.protocol.port = 20880

#指定注册中心的位置
dubbo.registry.address = zookeeper://localhost:2181

#统一设置服务提供方的规则
dubbo.provider.timeout = 1000
```

dubbo-consumer模块下的application.properties配置如下：

```
spring.application.name = dubbo-consumer
server.port = 9091

#指定当前服务/应用的名字（同样的服务名字相同，不要和别的服务同名）
dubbo.application.name = dubbo-consumer

demo.service.version = 1.0.0

dubbo.protocol.name = dubbo
dubbo.protocol.port = 20880

#指定注册中心的位置
dubbo.registry.address = zookeeper://localhost:2181

#统一设置服务提供方的规则
dubbo.consumer.timeout = 5000
```

### 运行项目

启动DubboProviderApplication和DubboConsumerApplication

访问：[http://localhost:9091/sayHello/HelloWorld](http://localhost:9091/sayHello/HelloWorld)
![](/Dubbo入门实战/微信截图_20190419140359.png)

### github地址
https://github.com/LinXiaoBaiXCG/dubbo-demo
  

  