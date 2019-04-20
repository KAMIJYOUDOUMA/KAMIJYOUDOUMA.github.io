---
title: Netty源码分析（一）：Netty总览
categories: Netty
tags:
  - Netty
  - 源码分析
comments: true
abbrlink: 40000
date: 2019-03-23
---

>作为当前最流行的网络通信框架，Netty在互联网领域大放异彩，本系列将详细介绍Netty(4.1.22.Final)。
# 代码事例
## 服务端
```
public final class EchoServer {
    // 从启动参数判断是否使用ssl
    static final boolean SSL = System.getProperty("ssl") != null;
    // 获取端口（默认8007）
    static final int PORT = Integer.parseInt(System.getProperty("port", "8007"));

    public static void main(String[] args) throws Exception {
        // 配置SSL
        final SslContext sslCtx;
        if (SSL) {
            SelfSignedCertificate ssc = new SelfSignedCertificate();
            sslCtx = SslContextBuilder.forServer(ssc.certificate(), ssc.privateKey()).build();
        } else {
            sslCtx = null;
        }
        // 配置服务器
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .option(ChannelOption.SO_BACKLOG, 100)
                    .handler(new LoggingHandler(LogLevel.INFO))
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        public void initChannel(SocketChannel ch) throws Exception {
                            ChannelPipeline p = ch.pipeline();
                            if (sslCtx != null) {
                                p.addLast(sslCtx.newHandler(ch.alloc()));
                            }
                            // 添加一个自定义的handler
                            p.addLast(new EchoServerHandler());
                        }
                    });
            // 启动服务器
            ChannelFuture f = b.bind(PORT).sync();
            // 等待至连接断开
            f.channel().closeFuture().sync();
        } finally {
            // 关闭服务器资源
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}
```
## 客户端
```
public final class EchoClient {
    // 获取是否使用ssl
    static final boolean SSL = System.getProperty("ssl") != null;
    // 获取服配置的服务器地址(默认本地)
    static final String HOST = System.getProperty("host", "127.0.0.1");
    // 获取端口（默认8007）
    static final int PORT = Integer.parseInt(System.getProperty("port", "8007"));
    // 首次发送消息的大小
    static final int SIZE = Integer.parseInt(System.getProperty("size", "256"));

    public static void main(String[] args) throws Exception {
        // 配置SSL
        final SslContext sslCtx;
        if (SSL) {
            sslCtx = SslContextBuilder.forClient()
                    .trustManager(InsecureTrustManagerFactory.INSTANCE).build();
        } else {
            sslCtx = null;
        }
        // 配置客户端
        EventLoopGroup group = new NioEventLoopGroup();
        try {
            Bootstrap b = new Bootstrap();
            b.group(group)
                    .channel(NioSocketChannel.class)
                    // 禁用Nagle算法
                    .option(ChannelOption.TCP_NODELAY, true)
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        public void initChannel(SocketChannel ch) throws Exception {
                            ChannelPipeline p = ch.pipeline();
                            if (sslCtx != null) {
                                p.addLast(sslCtx.newHandler(ch.alloc(), HOST, PORT));
                            }
                            //p.addLast(new LoggingHandler(LogLevel.INFO));
                            p.addLast(new EchoClientHandler());
                        }
                    });
            // 启动客户端
            ChannelFuture f = b.connect(HOST, PORT).sync();
            //等待至连接断开
            f.channel().closeFuture().sync();
        } finally {
            // 关闭客户端资源
            group.shutdownGracefully();
        }
    }
}
```
# 运行流程
## 服务器
![](https://user-gold-cdn.xitu.io/2019/3/23/169a9f9d2b9be73d?w=1412&h=780&f=png&s=127632)
## 客户端
![](https://user-gold-cdn.xitu.io/2019/3/23/169a9fa486131392?w=983&h=472&f=png&s=41707)
# 总结
本篇篇幅较短，只是简单贴了一段netty-example下面的代码，并梳理了一下netty的流程。接下来的几篇会详细介绍netty服务端和客户端启动时做了哪些事情。
我目前正在尝试在给netty添加注释：https://github.com/KAMIJYOUDOUMA/nettyForAnalysis.git  ，有兴趣的童鞋可以关注一下。

 

