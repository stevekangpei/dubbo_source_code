# dubbo源码-网络服务器-之Channel通道


![](media/17451604909123/17451606874744.jpg)


### AbstractChannel


```java
@Override
public void send(Object message, boolean sent) throws RemotingException {
    if (isClosed()) {
        throw new RemotingException(this, "Failed to send message "
                + (message == null ? "" : message.getClass().getName()) + ":" + message
                + ", cause: Channel closed. channel: " + getLocalAddress() + " -> " + getRemoteAddress());
    }
}
```

#### 说明：
* com.alibaba.dubbo.remoting.transport.AbstractChannel ，实现 Channel 接口，实现 AbstractPeer 抽象类，通道抽象类。
* 具体的发送方法，子类实现。在 AbstractChannel 中，目前只做状态检查。


### NettyChannel

```java
/**
 * 通道集合
 */
private static final ConcurrentMap<io.netty.channel.Channel, NettyChannel> channelMap = new ConcurrentHashMap<Channel, NettyChannel>();

/**
 * 通道
 */
private final io.netty.channel.Channel channel;
/**
 * 属性集合
 */
private final Map<String, Object> attributes = new ConcurrentHashMap<String, Object>();

private NettyChannel(io.netty.channel.Channel channel, URL url, ChannelHandler handler) {
    super(url, handler);
    if (channel == null) {
        throw new IllegalArgumentException("netty channel == null;");
    }
    this.channel = channel;
}
```

#### 说明：
* io.netty.channel.ChannelFuture.NettyChannel ，实现 AbstractChannel 抽象类，封装 Netty Channel 的通道实现类。
* channel 属性，通道。NettyChannel 是传入 channel 属性的装饰器，每个实现的方法，都会调用 channel 。
* attributes 属性，属性集合。注意，setAttribute(...) 等方法，使用的是该属性，而不是 io.netty.channel.Channel 的。
* channelMap 静态属性，通道集合。在实际 Netty Handler 里（例如下面我们会看到的 NettyServerHandler 和 NettyClientHandler），每个方法参数里，传递的是
* io.netty.channel.Channel 对象。通过 NettyChannel.channelMap 中，获得对应的 NettyChannel 对象。


####  getOrAddChannel 用于创建 NettyChannel 对象
```java
static NettyChannel getOrAddChannel(io.netty.channel.Channel ch, URL url, ChannelHandler handler) {
    if (ch == null) {
        return null;
    }
    NettyChannel ret = channelMap.get(ch);
    if (ret == null) {
        NettyChannel nettyChannel = new NettyChannel(ch, url, handler);
        if (ch.isActive()) { // 连接中
            ret = channelMap.putIfAbsent(ch, nettyChannel); // 添加到 channelMap
        }
        if (ret == null) {
            ret = nettyChannel;
        }
    }
    return ret;
}
```

#### send

```java
    @Override
        /**
     * (1) 检查连接状态
     * (2) 发送消息 调用真正的 io.netty.channel.Channel#writeAndFlush(message) 方法，发送消息。
     * (3) success ，是否执行成功。若不需要等待发送成功( sent = false ) ，默认成功。
     * @param message 消息 
     * @param sent    already sent to socket?
     * @throws RemotingException
     */
    public void send(Object message, boolean sent) throws RemotingException {
        // 检查连接状态
        super.send(message, sent);

        boolean success = true; // 如果没有等待发送成功，默认成功。
        int timeout = 0;
        try {
            // 发送消息
            ChannelFuture future = channel.writeAndFlush(message);
            // 等待发送成功
            if (sent) {
                timeout = getUrl().getPositiveParameter(Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
                success = future.await(timeout);
            }
            // 若发生异常，抛出
            Throwable cause = future.cause();
            if (cause != null) {
                throw cause;
            }
        } catch (Throwable e) {
            throw new RemotingException(this, "Failed to send message " + message + " to " + getRemoteAddress() + ", cause: " + e.getMessage(), e);
        }

        // 发送失败，抛出异常
        if (!success) {
            throw new RemotingException(this, "Failed to send message " + message + " to " + getRemoteAddress()
                    + "in timeout(" + timeout + "ms) limit");
        }
    }

```

#### 说明：
 * (1) 检查连接状态
 * (2) 发送消息调用真正的 io.netty.channel.Channel#writeAndFlush(message) 方法，发送消息。
 * (3) success ，是否执行成功。若不需要等待发送成功( sent = false ) ，默认成功。


#### isConnected
```java
@Override
public boolean isConnected() {
    return !isClosed() && channel.isActive();
}
```


#### close方法

```java
@Override
@SuppressWarnings("Duplicates")
public void close() {
    // 标记关闭
    try {
        super.close();
    } catch (Exception e) {
        logger.warn(e.getMessage(), e);
    }
    // 移除连接
    try {
        removeChannelIfDisconnected(channel);
    } catch (Exception e) {
        logger.warn(e.getMessage(), e);
    }
    // 清空属性 attributes
    try {
        attributes.clear();
    } catch (Exception e) {
        logger.warn(e.getMessage(), e);
    }
    // 关闭真正的通道 channel
    try {
        if (logger.isInfoEnabled()) {
            logger.info("Close netty channel " + channel);
        }
        channel.close();
    } catch (Exception e) {
        logger.warn(e.getMessage(), e);
    }
}
```

#### 说明：
* 1，标记关闭
* 2，移除连接
* 3，清空属性 attributes
* 4，关闭真正的通道 channel   channel.close();

