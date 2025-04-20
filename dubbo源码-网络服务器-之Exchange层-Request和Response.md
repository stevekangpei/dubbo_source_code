# dubbo源码-网络服务器-之Exchange层-Request和Response


### Request

```java
/**
 * 事件 - 心跳
 */
public static final String HEARTBEAT_EVENT = null;
/**
 * 事件 - 只读
 */
public static final String READONLY_EVENT = "R";

/**
 * 请求编号自增序列
 */
private static final AtomicLong INVOKE_ID = new AtomicLong(0);

/**
 * 请求编号
 */
private final long mId;
/**
 * Dubbo 版本
 */
private String mVersion;
/**
 * 是否需要响应
 *
 * true-需要
 * false-不需要
 */
private boolean mTwoWay = true;
/**
 * 是否是事件。例如，心跳事件。
 */
private boolean mEvent = false;
/**
 * 是否异常的请求。
 *
 * 在消息解析的时候，会出现。
 */
private boolean mBroken = false;
/**
 * 数据
 */
private Object mData;
```

#### 说明：
* 1，HEARTBEAT_EVENT ：心跳。因为心跳比较常用，所以在事件上时候了 null 。
* 2，READONLY_EVENT ：只读。上文已经解释。
* 3，mId 属性：编号。使用 INVOKE_ID 属性生成，JVM 进程内唯一。
* 4, version 属性，版本号。目前使用 Dubbo 大版本，"2.0.0" 。
* 5, mTwoWay 属性，标记请求是否响应( Response )，默认需要。
* 6, mBroken 属性，是否异常的请求。在消息解析的时候，会出现。
* 7, mData 属性，请求具体数据。


```java
private static long newId() {
    // getAndIncrement() When it grows to MAX_VALUE, it will grow to MIN_VALUE, and the negative can be used as ID
    return INVOKE_ID.getAndIncrement();
}
```

### Response
```java
/**
 * 响应编号
 *
 * 一个 {@link Request#mId} 和 {@link Response#mId} 一一对应。
 */
private long mId = 0;
/**
 * 版本
 */
private String mVersion;
/**
 * 状态
 */
private byte mStatus = OK;
/**
 * 是否事件
 */
private boolean mEvent = false;
/**
 * 错误消息
 */
private String mErrorMsg;
/**
 * 结果
 */
private Object mResult;
```
#### 说明：
* mEvent 属性，是否事件。和 Request 内置了一样的事件，但是 READONLY_EVENT 并未使用。因为目前，只读事件，无需响应。
* mErrorMsg 属性，错误消息。
* mResult 属性，结果。

### DefaultFuture

```java
    /**
     * 请求编号
     */
    // invoke id.
    private final long id;
    /**
     * 通道
     */
    private final Channel channel;
    /**
     * 请求
     */
    private final Request request;
    /**
     * 超时
     */
    private final int timeout;
    /**
     * 锁
     */
    private final Lock lock = new ReentrantLock();
    /**
     * 完成 Condition
     */
    private final Condition done = lock.newCondition();
    /**
     * 创建开始时间
     */
    private final long start = System.currentTimeMillis();
    /**
     * 发送请求时间
     */
    private volatile long sent;
    /**
     * 响应
     */
    private volatile Response response;
    /**
     * 回调
     */
    private volatile ResponseCallback callback;

    public DefaultFuture(Channel channel, Request request, int timeout) {
        this.channel = channel;
        this.request = request;
        this.id = request.getId();
        this.timeout = timeout > 0 ? timeout : channel.getUrl().getPositiveParameter(Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
        // put into waiting map.
        FUTURES.put(id, this);
        CHANNELS.put(id, channel);
    }
```
#### 说明：
* CHANNELS 静态属性，通道集合。通过 #hasFuture(channel) 方法，判断通道是否有未结束的请求。代码如下：

* sent 属性，发送请求时间。因为在目前 Netty Mina 等通信框架中，发送请求一般是异步的，因此在 ChannelHandler#sent(channel, message) 方法中，调用 DefaultFuture#sent(channel, request) 静态方法，代码如下：


#### get方法

```java
    @Override
    public Object get(int timeout) throws RemotingException {
        if (timeout <= 0) {
            timeout = Constants.DEFAULT_TIMEOUT;
        }
        // 若未完成，等待
        if (!isDone()) {
            long start = System.currentTimeMillis();
            lock.lock();
            try {
                // 等待完成或超时
                while (!isDone()) {
                    done.await(timeout, TimeUnit.MILLISECONDS);
                    if (isDone() || System.currentTimeMillis() - start > timeout) {
                        break;
                    }
                }
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            } finally {
                lock.unlock();
            }
            // 未完成，抛出超时异常 TimeoutException
            if (!isDone()) {
                throw new TimeoutException(sent > 0, channel, getTimeoutMessage(false));
            }
        }
        // 返回响应
        return returnFromResponse();
    }
```

#### 说明：
* 1，调用 #isDone() 方法，判断是否完成。若未完成，基于 Lock + Condition 的方式，实现等待。而等待的唤醒，通过 ChannelHandler#received(channel, message) 方法，接收到请求时执行 DefaultFuture#received(channel, response) 方法。
* 