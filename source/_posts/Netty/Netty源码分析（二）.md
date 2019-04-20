---
title: Netty源码分析（二）：服务端启动
categories: Netty
tags:
  - Netty
  - 源码分析
comments: true
abbrlink: 40001
date: 2019-03-25
---

>上一篇粗略的介绍了一下netty，本篇将详细介绍Netty的服务器的启动过程。
# ServerBootstrap
看过上篇事例的人，可以知道`ServerBootstrap`是Netty服务端启动中扮演着一个重要的角色。
它是Netty提供的一个服务端引导类，继承自`AbstractBootstrap`。  
`ServerBootstrap`主要包括两部分：`bossGroup`和`workerGroup`。其中`bossGroup`主要用于绑定端口，接收来自客户端的请求，接收到请求之后，就会把这些请求交给`workGroup`去处理。就像现实中的老板和员工一样，自己开个公司（绑定端口），到外面接活（接收请求），使唤员工干活（让worker去处理）。
# 端口绑定
端口绑定之前，会先check引导类（ServerBootstrap）的bossGroup和workerGroup有没有设置，之后再调用doBind。
```
    private ChannelFuture doBind(final SocketAddress localAddress) {
        // 初始化并注册一个channel，并将chanelFuture返回
        final ChannelFuture regFuture = initAndRegister();
        // 得到实际的channel（初始化和注册的动作可能尚未完成）
        final Channel channel = regFuture.channel();
        // 发生异常时，直接返回
        if (regFuture.cause() != null) {
            return regFuture;
        }
        // 当到这chanel相关处理已经完成时
        if (regFuture.isDone()) {
            // 到这可以确定channel已经注册成功
            ChannelPromise promise = channel.newPromise();
            // 进行相关的绑定操作
            doBind0(regFuture, channel, localAddress, promise);
            return promise;
        } else {
            // 注册一般到这就已经完成，到以防万一
            final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
            // 添加一个监听器
            regFuture.addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    Throwable cause = future.cause();
                    if (cause != null) {
                        promise.setFailure(cause);
                    } else {
                        // 修改注册状态为成功（当注册成功时不在使用全局的executor，使用channel自己的，详见 https://github.com/netty/netty/issues/2586）
                        promise.registered();
                        // 进行相关的绑定操作
                        doBind0(regFuture, channel, localAddress, promise);
                    }
                }
            });
            return promise;
        }
    }
```
上面的代码主要有两部分：初始化并注册一个channel和绑定端口。
## 初始化并注册一个channel
```
    final ChannelFuture initAndRegister() {
        Channel channel = null;
        try {
            // 生产各新channel
            channel = channelFactory.newChannel();
            // 初始化channel
            init(channel);
        } catch (Throwable t) {
            if (channel != null) {
                // 注册失败时强制关闭
                channel.unsafe().closeForcibly();
                // 由于channel尚未注册好，强制使用GlobalEventExecutor
                return new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE).setFailure(t);
            }
            return new DefaultChannelPromise(new FailedChannel(), GlobalEventExecutor.INSTANCE).setFailure(t);
        }
        // 注册channel
        ChannelFuture regFuture = config().group().register(channel);
        if (regFuture.cause() != null) {
            if (channel.isRegistered()) {
                channel.close();
            } else {
                channel.unsafe().closeForcibly();
            }
        }
        return regFuture;
    }
```
channel的初始化方法：
```
    void init(Channel channel) throws Exception {
        // 获取bossChannel的可选项Map
        final Map<ChannelOption<?>, Object> options = options0();
        synchronized (options) {
            setChannelOptions(channel, options, logger);
        }
        // 获取bossChannel的属性Map
        final Map<AttributeKey<?>, Object> attrs = attrs0();
        synchronized (attrs) {
            for (Entry<AttributeKey<?>, Object> e : attrs.entrySet()) {
                @SuppressWarnings("unchecked")
                AttributeKey<Object> key = (AttributeKey<Object>) e.getKey();
                channel.attr(key).set(e.getValue());
            }
        }
        ChannelPipeline p = channel.pipeline();
        // 设置worker的相关属性
        final EventLoopGroup currentChildGroup = childGroup;
        final ChannelHandler currentChildHandler = childHandler;
        final Entry<ChannelOption<?>, Object>[] currentChildOptions;
        final Entry<AttributeKey<?>, Object>[] currentChildAttrs;
        synchronized (childOptions) {
            currentChildOptions = childOptions.entrySet().toArray(newOptionArray(childOptions.size()));
        }
        synchronized (childAttrs) {
            currentChildAttrs = childAttrs.entrySet().toArray(newAttrArray(childAttrs.size()));
        }
        p.addLast(new ChannelInitializer<Channel>() {
            @Override
            public void initChannel(final Channel ch) throws Exception {
                final ChannelPipeline pipeline = ch.pipeline();
                // 添加handler到pipeline
                ChannelHandler handler = config.handler();
                if (handler != null) {
                    pipeline.addLast(handler);
                }
                // 通过EventLoop将ServerBootstrapAcceptor到pipeline中，保证它是最后一个handler
                ch.eventLoop().execute(new Runnable() {
                    @Override
                    public void run() {
                        pipeline.addLast(new ServerBootstrapAcceptor(
                                ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                    }
                });
            }
        });
    }
```
channel的注册方法，最终是调用doRegister，不同的channel有所不同，下面以Nio为例：
```
    protected void doRegister() throws Exception {
        boolean selected = false;
        for (; ; ) {
            try {
                // 直接调用java的提供的Channel的注册方法
                selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
                return;
            } catch (CancelledKeyException e) {
                if (!selected) {
                    eventLoop().selectNow();
                    selected = true;
                } else {
                    throw e;
                }
            }
        }
    }
```
## 绑定端口
最终调用的是NioServerSocketChannel的doBind方法。
```
    protected void doBind(SocketAddress localAddress) throws Exception {
        if (PlatformDependent.javaVersion() >= 7) {
            javaChannel().bind(localAddress, config.getBacklog());
        } else {
            javaChannel().socket().bind(localAddress, config.getBacklog());
        }
    }
```
到这就完成了netty服务端的整个启动过程。

文中帖的代码注释全在：https://github.com/KAMIJYOUDOUMA/nettyForAnalysis.git ， 有兴趣的童鞋可以关注一下。

 

