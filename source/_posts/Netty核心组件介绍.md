---
title: Netty核心组件介绍
date: 2021-04-09 20:00:00
tags: [网络编程]
categories: 编程
---

>  理解Netty核心组件并使用netty

# Netty核心组件

## Channel

当客户端和服务端连接的时候会建立一个 Channel。Channel负责基本的 IO 操作，例如：bind（），connect（），read（），write（）等等。在Java的网络编程中，基础核心类是Socket，而Netty的Channel提供了一组API，极大地简化了直接与Socket进行操作的复杂性。

## EventLoop和EventLoopGoup

有了 Channel 连接服务，让信息之间可以流动。如果服务发出的消息称作“出站”消息，服务接受的消息称作“入站”消息。那么消息的“出站”/“入站”就会产生事件（Event）。

例如：连接已激活；数据读取；用户事件；异常事件；打开链接；关闭链接等等。

有了数据，数据的流动产生事件，那么就有一个机制去监控和协调事件。

这个机制（组件）就是 EventLoop。在 Netty 中每个 Channel 都会被分配到一个 EventLoop。一个 EventLoop 可以服务于多个 Channel。

每个 EventLoop 会占用一个 Thread，同时这个 Thread 会处理 EventLoop 上面发生的所有 IO 操作和事件（Netty 4.0）。

![](netty核心组件介绍\1.jpg)

理解了 EventLoop，再来说 EventLoopGroup 就容易了，EventLoopGroup 是用来生成 EventLoop 的。

EventLoopGroup 要做的就是创建一个新的Channel，并且给它分配一个 EventLoop。

![](netty核心组件介绍\2.png)

可以看到。

* 一个EventLoopGroup包含一个或多个EventLoop。
*  一个EventLoop在生命中周期绑定到一个Thread上。
*  EventLoop使用其对应的Thread处理IO事件。
*  一个Channel使用EventLoop进行注册。
*  一个EventLoop可被分配至一个或多个Channel。

## ChannelFuture

Netty中的所有IO操作都是异步的，不会立即返回，需要在稍后确定操作结果。因此Netty提供了ChannelFuture，其addListener方法可以注册一个ChannelFutureListener，当操作完成时，不管成功还是失败，均会被通知。ChannelFuture存储了之后执行的操作的结果并且无法预测操作何时被执行，提交至Channel的操作按照被唤醒的顺序被执行。

```java
ChannelFuture f = b.connect(host, port).addListener(future -> {
    if (future.isSuccess()) {
        System.out.println("连接成功!");
    } else {
        System.err.println("连接失败!");
    }
}).sync();

    //通过ChannelFuture 的 channel() 方法获取连接相关联的Channel 
    Channel channel = f.channel();
    
    //通过 ChannelFuture 接口的 sync()方法让异步的操作编程同步的。
    //bind()是异步的，但是，你可以通过 `sync()`方法将其变为同步。
    ChannelFuture f = b.bind(port).sync();
```

## ChannelHandler、ChannelPipeline和ChannelHandlerContext

如果说 EventLoop 是事件的通知者，那么 ChannelHandler 就是事件的处理者。

从应用开发者看来，ChannelHandler是最重要的组件，其中存放用来处理进站和出站数据的用户逻辑。

ChannelHandler的方法被网络事件触发，ChannelHandler可以用于几乎任何类型的操作，如将数据从一种格式转换为另一种格式或处理抛出的异常。例如，其子接口ChannelInboundHandler，接受进站的事件和数据以便被用户定义的逻辑处理，或者当响应所连接的客户端时刷新ChannelInboundHandler的数据。

假设每次请求都会触发事件，而由 ChannelHandler 来处理这些事件，这个事件的处理顺序是由 ChannelPipeline 来决定的。

ChannelPipeline 为 ChannelHandler 链提供了容器。到 Channel 被创建的时候，会被 Netty 框架自动分配到 ChannelPipeline 上。

ChannelPipeline 保证 ChannelHandler 按照一定顺序处理事件，当事件触发以后，会将数据通过 ChannelPipeline 按照一定的顺序通过 ChannelHandler。

说白了，ChannelPipeline 是负责“排队”的。这里的“排队”是处理事件的顺序。

同时，ChannelPipeline 也可以添加或者删除 ChannelHandler，管理整个队列。

![](netty核心组件介绍\3.jpg)

如上图，ChannelPipeline 使 ChannelHandler 按照先后顺序排列，信息按照箭头所示方向流动并且被 ChannelHandler 处理。

说完了 ChannelPipeline 和 ChannelHandler，前者管理后者的排列顺序。那么它们之间的关联就由 ChannelHandlerContext 来表示了。

每当有 ChannelHandler 添加到 ChannelPipeline 时，同时会创建 ChannelHandlerContext 。

ChannelHandlerContext 的主要功能是管理 ChannelHandler 和 ChannelPipeline 的交互。

ChannelHandlerContext 参数贯穿 ChannelPipeline，将信息传递给每个 ChannelHandler，是个合格的“通讯员”。

![](netty核心组件介绍\7.png)

## Bytebuf（字节容器）

网络通信最终都是通过字节流进行传输的。 ByteBuf 就是 Netty 提供的一个字节容器，其内部是一个字节数组。 当我们通过 Netty 传输数据的时候，就是通过 ByteBuf 进行的。

我们可以将 ByteBuf 看作是 Netty 对 Java NIO 提供了 ByteBuffer 字节容器的封装和抽象。

有很多小伙伴可能就要问了 ： 为什么不直接使用 Java NIO 提供的 ByteBuffer 呢？

因为 ByteBuffer 这个类使用起来过于复杂和繁琐。

## Bootstrap 和 ServerBootstrap（启动引导类）

Bootstrap 的作用就是将 Netty 核心组件配置到程序中，并且让他们运行起来。

客户端引导 Bootstrap，主要有两个方法 bind（） 和 connect（）。Bootstrap 通过 bind（） 方法创建一个 Channel。

在 bind（） 之后，通过调用 connect（） 方法来创建 Channel 连接。



服务端引导 ServerBootstrap，与客户端不同的是在 Bind（） 方法之后会创建一个 ServerChannel，它不仅会创建新的 Channel 还会管理已经存在的 Channel。

服务端和客户端的引导存在两个区别：

- ServerBootstrap（服务端引导）绑定一个端口，用来监听客户端的连接请求。而 Bootstrap（客户端引导）只要知道服务端 IP 和 Port 建立连接就可以了。
- Bootstrap（客户端引导）需要一个 EventLoopGroup，但是 ServerBootstrap（服务端引导）则需要两个 EventLoopGroup。因为服务器需要两组不同的 Channel。第一组 ServerChannel 自身监听本地端口的套接字。第二组用来监听客户端请求的套接字。

## 几个核心组件的关系

![](netty核心组件介绍\4.jpg)