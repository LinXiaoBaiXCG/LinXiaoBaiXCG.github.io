---
title: Dubbo的高可用场景
date: 2019-09-03 21:50:00
tags: [分布式]
categories: 编程
---

# zookeeper宕机与dubbo直连
现象：zookeeper注册中心宕机，还可以消费dubbo暴露服务
原因：
健壮性
* 监控中心宕掉不影响使用，只是丢失部分采样数据
* 数据库宕掉后，注册中心仍能通过缓存提供服务列表查询，但不能注册新的服务
* 注册中心对等集群，任意一台宕掉后，将自动切换到另一台
* 注册中心全部宕掉后，服务提供者和服务消费者仍能通过本地缓存通讯
* 服务提供者无状态，任意一台宕掉后，不影响使用
* 服务提供者全部宕掉后，服务消费者应用将无法使用，并无限次重连等待服务提供者恢复

# 集群下dubbo负载均衡配置
在集群负载均衡时，Dubbo提供了多种均衡策略，默认为random随机调用
负载均衡策略
RandomLoadBalance
RoundRobinLoadBalance
LeastActiveLoadBalance
ConsistentHashLoadBalance
详解：[http://dubbo.apache.org/zh-cn/docs/source_code_guide/loadbalance.html](http://dubbo.apache.org/zh-cn/docs/source_code_guide/loadbalance.html)

# 服务降级
什么是服务降级
当服务器压力剧增的情况下，根据实际业务情况及流量，对一些服务和页面有策略的不出来或者换种简单的方式处理，从而释放服务器资源以保证核心交易正常运作或高效运作。

向注册中心写入动态配置覆盖规则：
```
RegistryFactory registryFactory = 
ExtensionLoader.getExtensionLoader(RegistryFactory.class).getAdaptiveExtension();
Registry registry = registryFactory.getRegistry(URL.valueOf("zookeeper://10.20.153.10:2181"));
registry.register(URL.valueOf("override://0.0.0.0/com.foo.BarService?category=configurators&dynamic=false&application=consumer_app&mock=force:return+null"));
```

mock=force:return+null 
表示消费方对该服务的方法调用都直接返回 null 值，不发起远程调用。用来屏蔽不重要服务不可用时对调用方的影响。
admin对消费者设置为屏蔽
mock=fail:return+null 
表示消费方对该服务的方法调用在失败后，再返回 null 值，不抛异常。用来容忍不重要服务不稳定时对调用方的影响。
admin对消费者设置为容错

dubbo 服务降级的真实含义：并不是对 provider 进行操作，而是告诉 consumer，调用服务时要做哪些动作。

# 集群容错模式
Dubbo 主要提供了这样几种容错方式：
* Failover Cluster - 失败自动切换
* Failfast Cluster - 快速失败
* Failsafe Cluster - 失败安全
* Failback Cluster - 失败自动恢复
* Forking Cluster - 并行调用多个服务提供者

可整合Hystrix进行服务熔断