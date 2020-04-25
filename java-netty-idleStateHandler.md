---
layout: post
comments: true
title: netty如何实现心跳检查
date: 2017-02-28 21:08:11
tags:
- netty
categories:
- java
---

### 前言

Java网络编程中，处理Idle事件是很必须的，一般情况下，客户端与服务端在指定时间内没有任何读写请求，就会认为连接是idle的。此时需要有某种机制来实现idle连接的检查，并通过心跳包来保持连接的存活。Netty作为时下非常流行的Java网络编程库，当然提供了空闲连接检查的能力。

<!-- more -->

### IdleStateHandler

Netty的空闲连接检查是通过类`IdleStateHandler`实现的。从名字上看它应该是一个ChannelHandler, ChannelHandler 只有在连接建立或关闭，处理读写请求时别使用，那`IdleStateHandler`纠结是通过哪种机制来实现周期性的检查任务呢？

了解Netty的同学都知道Netty本身提供了处理定时任务的功能，其实IdleStateHandler就是一种类型的定时任务，我们从代码逻辑就可以看出来。

```java
public class ChannelDuplexHandler extends ChannelInboundHandlerAdapter implements ChannelOutboundHandler {
    
}

public class IdleStateHandler extends ChannelDuplexHandler {
      @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        // This method will be invoked only if this handler was added
        // before channelActive() event is fired.  If a user adds this handler
        // after the channelActive() event, initialize() will be called by beforeAdd().
        initialize(ctx);
        super.channelActive(ctx);
    }
    
    private void initialize(ChannelHandlerContext ctx) {
        // Avoid the case where destroy() is called before scheduling timeouts.
        // See: https://github.com/netty/netty/issues/143
        switch (state) {
        case 1:
        case 2:
            return;
        }

        state = 1;
        initOutputChanged(ctx);

        lastReadTime = lastWriteTime = ticksInNanos();
        if (readerIdleTimeNanos > 0) {
            readerIdleTimeout = schedule(ctx, new ReaderIdleTimeoutTask(ctx),
                    readerIdleTimeNanos, TimeUnit.NANOSECONDS);
        }
        if (writerIdleTimeNanos > 0) {
            writerIdleTimeout = schedule(ctx, new WriterIdleTimeoutTask(ctx),
                    writerIdleTimeNanos, TimeUnit.NANOSECONDS);
        }
        if (allIdleTimeNanos > 0) {
            allIdleTimeout = schedule(ctx, new AllIdleTimeoutTask(ctx),
                    allIdleTimeNanos, TimeUnit.NANOSECONDS);
        }
    }
        
    /**
    * Is called when an {@link IdleStateEvent} should be fired. This implementation calls
    * {@link ChannelHandlerContext#fireUserEventTriggered(Object)}.
    */
    protected void channelIdle(ChannelHandlerContext ctx, IdleStateEvent evt) throws Exception {
        ctx.fireUserEventTriggered(evt);
    }
}
```

从上面的代码逻辑我们至少应该能得出下面的结论：

1. Netty的空闲连接检查确实是通过定时任务来实现。
2. Netty是在连接建立后，通过处理`channelActive`事件时开始检查连接是否空闲的。
3. Netty会根据`readerIdleTimeSeconds`,`writerIdleTimeSeconds`,`allIdleTimeSeconds`的值来注册不同的维度的空闲连接检查任务。
4. 当空闲连接的检查任务检查到连接空闲时，会触发对应的事件。事件类型如下：

```java
/**
 * An {@link Enum} that represents the idle state of a {@link Channel}.
 */
public enum IdleState {
    /**
     * No data was received for a while.
     */
    READER_IDLE,
    /**
     * No data was sent for a while.
     */
    WRITER_IDLE,
    /**
     * No data was either received or sent for a while.
     */
    ALL_IDLE
}
/**
 * A user event triggered by {@link IdleStateHandler} when a {@link Channel} is idle.
 */
public class IdleStateEvent {
    public static final IdleStateEvent FIRST_READER_IDLE_STATE_EVENT = new IdleStateEvent(IdleState.READER_IDLE, true);
    public static final IdleStateEvent READER_IDLE_STATE_EVENT = new IdleStateEvent(IdleState.READER_IDLE, false);
    public static final IdleStateEvent FIRST_WRITER_IDLE_STATE_EVENT = new IdleStateEvent(IdleState.WRITER_IDLE, true);
    public static final IdleStateEvent WRITER_IDLE_STATE_EVENT = new IdleStateEvent(IdleState.WRITER_IDLE, false);
    public static final IdleStateEvent FIRST_ALL_IDLE_STATE_EVENT = new IdleStateEvent(IdleState.ALL_IDLE, true);
    public static final IdleStateEvent ALL_IDLE_STATE_EVENT = new IdleStateEvent(IdleState.ALL_IDLE, false);

    private final IdleState state;
    private final boolean first;

    /**
     * Constructor for sub-classes.
     *
     * @param state the {@link IdleStateEvent} which triggered the event.
     * @param first {@code true} if its the first idle event for the {@link IdleStateEvent}.
     */
    protected IdleStateEvent(IdleState state, boolean first) {
        this.state = ObjectUtil.checkNotNull(state, "state");
        this.first = first;
    }

    /**
     * Returns the idle state.
     */
    public IdleState state() {
        return state;
    }

    /**
     * Returns {@code true} if this was the first event for the {@link IdleState}
     */
    public boolean isFirst() {
        return first;
    }
}
```
5. 我们的自定义的Handler应该处理`IdleStateEvent`事件来实现自己的心跳检测逻辑。

### 一个简单的例子：

```java
/**
 * Created by leo on 17/2/28.
 */
public class HeartBeatCheckClient {

    private final static int readerIdleTimeSeconds = 10;//读操作空闲30秒
    private final static int writerIdleTimeSeconds = 20;//写操作空闲60秒
    private final static int allIdleTimeSeconds = 30;//读写全部空闲100秒

    public void connect(String host, int port) throws Exception {
        // 配置客户端NIO线程组
        EventLoopGroup group = new NioEventLoopGroup();
        try {
            Bootstrap b = new Bootstrap();
            b.group(group).channel(NioSocketChannel.class)
                    .option(ChannelOption.TCP_NODELAY, true)
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        public void initChannel(SocketChannel ch) throws Exception {
                            ByteBuf delimiter = Unpooled.copiedBuffer("\n".getBytes());
                            ch.pipeline().addLast("idleStateHandler", new IdleStateHandler(readerIdleTimeSeconds, writerIdleTimeSeconds,allIdleTimeSeconds));
                            ch.pipeline().addLast( new DelimiterBasedFrameDecoder(1024, delimiter));
                            ch.pipeline().addLast(new StringDecoder());
                            ch.pipeline().addLast(new StringEncoder());
                            ch.pipeline().addLast(new EchoClientHandler());
                        }
                    });

            // 发起异步连接操作
            ChannelFuture f = b.connect(host, port).sync();
            // 当代客户端链路关闭
            f.channel().closeFuture().sync();
        } finally {
            // 优雅退出，释放NIO线程组
            group.shutdownGracefully();
        }
    }

    public class EchoClientHandler extends ChannelInboundHandlerAdapter {

        private int counter;
        static final String ECHO_REQ = "Hi, Lilinfeng. Welcome to Netty.\n";

        public EchoClientHandler() {
        }

        @Override
        public void channelActive(ChannelHandlerContext ctx) {
            for (int i = 0; i < 10; i++) {
                ctx.writeAndFlush(Unpooled.copiedBuffer(ECHO_REQ.getBytes()));
            }
        }

        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
            System.out.println("This is " + ++counter + " times receive server : [" + msg + "]");
        }

        @Override
        public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
            ctx.flush();
        }

        @Override
        public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
            cause.printStackTrace();
            ctx.close();
        }

        @Override
        public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
            if (IdleStateEvent.class.isAssignableFrom(evt.getClass())) {
                IdleStateEvent event = (IdleStateEvent) evt;
                if (event.state() == IdleState.READER_IDLE){
                    System.out.println("client read idle");
                } else if (event.state() == IdleState.WRITER_IDLE){
                    System.out.println("client write idle");
                } else if (event.state() == IdleState.ALL_IDLE){
                    System.out.println("client all idle");
                    ctx.writeAndFlush("ping\n");
                }
            }
        }
    }

    public static void main(String[] args) {
        HeartBeatCheckClient heartBeatCheckClient = new HeartBeatCheckClient();
        try {
            heartBeatCheckClient.connect("127.0.0.1", 8080);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

/**
 * Created by leo on 17/2/28.
 */
public class HeartBeatCheckServer {

    private final static int readerIdleTimeSeconds = 15;//读操作空闲30秒
    private final static int writerIdleTimeSeconds = 20;//写操作空闲60秒
    private final static int allIdleTimeSeconds = 30;//读写全部空闲100秒

    private static void bind(int port){
        NioEventLoopGroup bossGroup = new NioEventLoopGroup(1);
        NioEventLoopGroup workGroup = new NioEventLoopGroup(4);
        try {
            ServerBootstrap bootstrap = new ServerBootstrap();
            bootstrap
                    .group(bossGroup, workGroup)
                    .channel(NioServerSocketChannel.class)
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        protected void initChannel(SocketChannel ch) throws Exception {
                            ByteBuf delimiter = Unpooled.copiedBuffer("\n".getBytes());
                            ch.pipeline().addLast(new IdleStateHandler(readerIdleTimeSeconds, writerIdleTimeSeconds, allIdleTimeSeconds));
                            ch.pipeline().addLast(new DelimiterBasedFrameDecoder(1024, delimiter));
                            ch.pipeline().addLast(new StringDecoder());
                            ch.pipeline().addLast(new StringEncoder());
                            ch.pipeline().addLast(new ServerHandler());
                        }
                    });

            Channel ch = bootstrap.bind(port).sync().channel();
            System.out.println("server listen at " + port);
            ch.closeFuture().sync();
        } catch (Exception e) {
            throw new RuntimeException(e);
        } finally {
            bossGroup.shutdownGracefully();
            workGroup.shutdownGracefully();
        }
    }

    static class ServerHandler extends ChannelInboundHandlerAdapter {

        private ConcurrentMap<String, AtomicInteger> concurrentMap = new ConcurrentHashMap();

        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
            System.out.println("This is " + " receive from client : [" + msg + "]");
            if (msg.toString().equals("ping")){
                ctx.writeAndFlush("pong\n");
                String channelId = ctx.channel().id().asLongText();
                concurrentMap.remove(channelId);
            }else {
                String channelId = ctx.channel().id().asLongText();
                String resMsg = "hello client " + channelId;
                ctx.writeAndFlush(Unpooled.copiedBuffer(resMsg.getBytes()));
            }
        }

        @Override
        public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
            ctx.flush();
        }

        @Override
        public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
            cause.printStackTrace();
            ctx.close();
        }

        @Override
        public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
            if (IdleStateEvent.class.isAssignableFrom(evt.getClass())) {
                IdleStateEvent event = (IdleStateEvent) evt;
                if (event.state() == IdleState.READER_IDLE){
                    processReadIdle(ctx);
                } else if (event.state() == IdleState.WRITER_IDLE){
                    processWriteIdle(ctx);
                } else if (event.state() == IdleState.ALL_IDLE){
                    processAllIdle(ctx);
                }
            }
        }

        private void processReadIdle(ChannelHandlerContext ctx){
            String channelId = ctx.channel().id().asLongText();
            AtomicInteger idleTimes = concurrentMap.get(channelId);
            if (null == idleTimes){
                idleTimes = new AtomicInteger(1);
                concurrentMap.putIfAbsent(channelId, idleTimes);
            }
            int times = idleTimes.getAndIncrement();
            System.out.println("server read idle : " + times);
            if (times == 3){
                ctx.close();
            }
        }

        private void processWriteIdle(ChannelHandlerContext ctx){
            System.out.println("server write idle");
        }

        private void processAllIdle(ChannelHandlerContext ctx){
            System.out.println("server all idle");
        }

    }

    public static void main(String[] args) {
        HeartBeatCheckServer heartBeatCheckServer = new HeartBeatCheckServer();
        heartBeatCheckServer.bind(8080);
    }
}
```

### 参考

- Netty源代码
- [浅析 Netty 实现心跳机制与断线重连](https://segmentfault.com/a/1190000006931568#articleHeader7)


