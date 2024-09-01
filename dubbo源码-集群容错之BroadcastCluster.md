# dubbo源码-集群容错之BroadcastCluster

### 代码逻辑

#### BroadcastCluster

```java
public class BroadcastCluster implements Cluster {
    @Override
    public <T> Invoker<T> join(Directory<T> directory) throws RpcException {
        return new BroadcastClusterInvoker<T>(directory);
    }
}
```



#### BroadcastClusterInvoker

```java
 @Override
    @SuppressWarnings({"unchecked", "rawtypes"})
    public Result doInvoke(final Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        // 检查 invokers 即可用Invoker集合是否为空，
        checkInvokers(invokers, invocation);
        // 设置已经调用的 Invoker 集合，到 Context 中
        RpcContext.getContext().setInvokers((List) invokers);
        // 保存最后一次调用的异常
        RpcException exception = null;
        // 保存最后一次调用的结果
        Result result = null;
        // 循环候选的 Invoker 集合，调用所有 Invoker 对象。
        for (Invoker<T> invoker : invokers) {
            try {
                // 发起 RPC 调用
                result = invoker.invoke(invocation);
            } catch (RpcException e) {
                exception = e;
                logger.warn(e.getMessage(), e);
            } catch (Throwable e) {
                exception = new RpcException(e.getMessage(), e); // 封装成 RpcException 异常
                logger.warn(e.getMessage(), e);
            }
        }
        if (exception != null) {
            throw exception;
        }
        return result;
    }
```

#### 说明：
> 对于所有的invoker，都发起调用。
> 保留最后一次调用的结果和异常。
> 广播调用所有provider

