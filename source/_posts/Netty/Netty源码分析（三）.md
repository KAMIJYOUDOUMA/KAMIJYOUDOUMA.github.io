---
title: Netty源码分析（三）：客户端启动
categories: Netty
tags:
  - Netty
  - 源码分析
comments: true
abbrlink: 40002
date: 2019-03-27
---

# Bootstrap
`Bootstrap`主要包含两个部分，一个是服务器地址的解析器组`AddressResolverGroup`，另一个是用来工作的`EventLoopGroup`。
`EventLoopGroup`负责出人`EventLoop`，`AddressResolverGroup`负责给`EventLoop`解析服务器地址。
# 客户端连接远程服务器
连接远程服务器，会先check引导类（Bootstrap）的group有没有设置以及生成channel的工厂，之后再调用doResolveAndConnect方法。
```
    private ChannelFuture doResolveAndConnect(final SocketAddress remoteAddress, final SocketAddress localAddress) {
        // 初始化并注册一个channel，并将chanelFuture返回
        final ChannelFuture regFuture = initAndRegister();
        // 得到实际的channel（初始化和注册的动作可能尚未完成）
        final Channel channel = regFuture.channel();
        // 当到这chanel相关处理已经完成时
        if (regFuture.isDone()) {
            // 连接失败直接返回
            if (!regFuture.isSuccess()) {
                return regFuture;
            }
            // 解析服务器地址并完成连接动作
            return doResolveAndConnect0(channel, remoteAddress, localAddress, channel.newPromise());
        } else {
            // 注册一般到这就已经完成，这里以防万一
            final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
            // 添加一个监听器
            regFuture.addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    // Directly obtain the cause and do a null check so we only need one volatile read in case of a
                    // failure.
                    Throwable cause = future.cause();
                    if (cause != null) {
                        promise.setFailure(cause);
                    } else {
                        // 修改注册状态为成功（当注册成功时不在使用全局的executor，使用channel自己的，详见 https://github.com/netty/netty/issues/2586）
                        promise.registered();
                        // 进行相关的绑定操作
                        doResolveAndConnect0(channel, remoteAddress, localAddress, promise);
                    }
                }
            });
            return promise;
        }
    }
```
和`ServerBootstrap`一样，`Bootstrap`也要先去初始化和注册channel，注册的方法和`ServerBootstrap`相同，初始化channel的方法有所区别。
## 初始化channel
```
    void init(Channel channel) throws Exception {
        ChannelPipeline p = channel.pipeline();
        p.addLast(config.handler());
        // 获取channel的可选项Map
        final Map<ChannelOption<?>, Object> options = options0();
        synchronized (options) {
            setChannelOptions(channel, options, logger);
        }
        // 获取channel的属性Map
        final Map<AttributeKey<?>, Object> attrs = attrs0();
        synchronized (attrs) {
            for (Entry<AttributeKey<?>, Object> e : attrs.entrySet()) {
                channel.attr((AttributeKey<Object>) e.getKey()).set(e.getValue());
            }
        }
    }
```
客户端channel的初始化和服务端相比要简单很多，只需要设置一些相关信息即可。

channel处理好之后，客户端就会去连接服务器。
## 连接服务器
```
    private ChannelFuture doResolveAndConnect0(final Channel channel, SocketAddress remoteAddress,
                                               final SocketAddress localAddress, final ChannelPromise promise) {
        try {
            final EventLoop eventLoop = channel.eventLoop();
            final AddressResolver<SocketAddress> resolver = this.resolver.getResolver(eventLoop);
            // 解析器不知道怎么处理这个服务器地址或者已经处理过了
            if (!resolver.isSupported(remoteAddress) || resolver.isResolved(remoteAddress)) {
                doConnect(remoteAddress, localAddress, promise);
                return promise;
            }
            // 解析服务器地址
            final Future<SocketAddress> resolveFuture = resolver.resolve(remoteAddress);
            // 解析完成时
            if (resolveFuture.isDone()) {
                final Throwable resolveFailureCause = resolveFuture.cause();
                if (resolveFailureCause != null) {
                    // 解析失败直接关闭channel
                    channel.close();
                    promise.setFailure(resolveFailureCause);
                } else {
                    // 解析成功开始连接
                    doConnect(resolveFuture.getNow(), localAddress, promise);
                }
                return promise;
            }
            // 解析没完成时等待解析完成
            resolveFuture.addListener(new FutureListener<SocketAddress>() {
                @Override
                public void operationComplete(Future<SocketAddress> future) throws Exception {
                    if (future.cause() != null) {
                        // 解析失败直接关闭channel
                        channel.close();
                        promise.setFailure(future.cause());
                    } else {
                        // 解析成功开始连接
                        doConnect(future.getNow(), localAddress, promise);
                    }
                }
            });
        } catch (Throwable cause) {
            promise.tryFailure(cause);
        }
        return promise;
    }
```
当服务器地址是IP的时候，直接连接，如果是域名之类的，会先解析出服务器的IP地址，然后再进行连接。域名解析直接使用的java.net.InetAddress的getByName方法，而连接的方法调用的是sun.nio.ch.SocketChannelImpl的connect方法。

文中帖的代码注释全在：https://github.com/KAMIJYOUDOUMA/nettyForAnalysis.git ， 有兴趣的童鞋可以关注一下。


 

