---
title: Netty源码分析（四）：EventLoopGroup
categories: Netty
tags:
  - Netty
  - 源码分析
comments: true
abbrlink: 40003
date: 2019-04-07
---

>无论服务端或客户端启动时都用到了`NioEventLoopGroup`，从名字就可以看出来它是`NioEventLoop`的组合，是Netty多线程的基石。

# 类结构
![](https://user-gold-cdn.xitu.io/2019/4/7/169f6c162fd4d054?w=773&h=820&f=jpeg&s=39700)
`NioEventLoopGroup`继承自`MultithreadEventLoopGroup`，多提供了两个方法`setIoRatio`和`rebuildSelectors`，一个用于设置`NioEventLoop`用于IO处理的时间占比，另一个是重新构建Selectors，来处理epoll空轮询导致CPU100%的bug。这个两个的用处在介绍`NioEventLoop`的时候在详细介绍。其它的方法都在接口中有定义，先看下`EventExecutorGroup`。
# EventExecutorGroup
`EventExecutorGroup`继承自`ScheduledExecutorService`和`Iterable`。这意味着`EventExecutorGroup`拥有定时处理任务的能力，同时本身可以迭代。它提供的方法有：
```java
    /**
     * 是否所有事件执行器都处在关闭途中或关闭完成
     */
    boolean isShuttingDown();

    /**
     * 优雅关闭
     */
    Future<?> shutdownGracefully();

    Future<?> shutdownGracefully(long quietPeriod, long timeout, TimeUnit unit);

    /**
     * 返回线程池终止时的异步结果
     */
    Future<?> terminationFuture();

    void shutdown();

    List<Runnable> shutdownNow();

    /**
     * 返回一个事件执行器
     */
    EventExecutor next();
```
其中`shutdown`和`shutdownNow`被标记为过时，不建议使用。`EventExecutorGroup`还重写`ScheduledExecutorService`接口的方法，用于返回自定义的`Future`。
# EventLoopGroup
`EventLoopGroup`继承自`EventExecutorGroup`，它和`EventExecutorGroup`想比多了注册`Channel`和`ChannelPromise`，同时重新next方法返回`EventLoop`。
# MultithreadEventExecutorGroup
创建`NioEventLoopGroup`时，最终都会调用`MultithreadEventExecutorGroup`的构造方法。
```java
    protected MultithreadEventExecutorGroup(int nThreads, Executor executor,
                                            EventExecutorChooserFactory chooserFactory, Object... args) {
        // 线程数必须大于0
        if (nThreads <= 0) {
            throw new IllegalArgumentException(String.format("nThreads: %d (expected: > 0)", nThreads));
        }
        // 没指定Executor就创建新的Executor
        if (executor == null) {
            executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
        }
        // 创建EventExecutor数组
        children = new EventExecutor[nThreads];
        for (int i = 0; i < nThreads; i++) {
            // 创建结果标识
            boolean success = false;
            try {
                // 创建EventExecutor对象
                children[i] = newChild(executor, args);
                // 设置创建成功
                success = true;
            } catch (Exception e) {
                throw new IllegalStateException("failed to create a child event loop", e);
            } finally {
                // 创建失败，关闭所有已创建的EventExecutor
                if (!success) {
                    // 关闭所有已创建的EventExecutor
                    for (int j = 0; j < i; j++) {
                        children[j].shutdownGracefully();
                    }
                    // 确保所有已创建的EventExecutor已关闭
                    for (int j = 0; j < i; j++) {
                        EventExecutor e = children[j];
                        try {
                            while (!e.isTerminated()) {
                                e.awaitTermination(Integer.MAX_VALUE, TimeUnit.SECONDS);
                            }
                        } catch (InterruptedException interrupted) {
                            // Let the caller handle the interruption.
                            Thread.currentThread().interrupt();
                            break;
                        }
                    }
                }
            }
        }
        // 创建EventExecutor选择器
        chooser = chooserFactory.newChooser(children);
        // 创建监听器，用于EventExecutor终止时的监听
        final FutureListener<Object> terminationListener = new FutureListener<Object>() {
            @Override
            public void operationComplete(Future<Object> future) throws Exception {
                // 当EventExecutor全部关闭时
                if (terminatedChildren.incrementAndGet() == children.length) {
                    // 设置结果，并通知监听器们。
                    terminationFuture.setSuccess(null);
                }
            }
        };
        // 给每个EventExecutor添加上监听器
        for (EventExecutor e : children) {
            e.terminationFuture().addListener(terminationListener);
        }
        // 创建只读的EventExecutor集合
        Set<EventExecutor> childrenSet = new LinkedHashSet<EventExecutor>(children.length);
        Collections.addAll(childrenSet, children);
        readonlyChildren = Collections.unmodifiableSet(childrenSet);
    }
```
整个构造方法做的就是`EventExecutor`的创建，包括创建的异常处理，成功通知等。
`AbstractEventExecutorGroup`、`MultithreadEventLoopGroup`和`NioEventLoopGroup`内部没有特殊之处，就不拓展了。

文中帖的代码注释全在：[KAMIJYOUDOUMA](https://github.com/KAMIJYOUDOUMA/nettyForAnalysis.git)， 有兴趣的童鞋可以关注一下。


 

