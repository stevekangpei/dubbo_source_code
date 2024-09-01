# dubbo源码-集群容错之FailsafeCluster

### 代码逻辑
 ```Usually used to write audit logs and other operations
    失败安全，出现异常时，直接忽略。通常用于写入日志等操作
 ```

#### FailsafeCluster
```java
public class FailfastCluster implements Cluster {

    public final static String NAME = "failfast";

    @Override
    public <T> Invoker<T> join(Directory<T> directory) throws RpcException {
        return new FailfastClusterInvoker<T>(directory);
    }

}

```
#### FailsafeClusterInvoker
```java

    @Override
    public Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        try {
            checkInvokers(invokers, invocation);
            // 根据负载均衡机制从 invokers 中选择一个Invoker
            Invoker<T> invoker = select(loadbalance, invocation, invokers, null);
            // RPC 调用得到 Result
            return invoker.invoke(invocation);
        } catch (Throwable e) {
            // 打印异常日志
            logger.error("Failsafe ignore exception: " + e.getMessage(), e);
            // 忽略异常
            return new RpcResult(); // ignore
        }
    }
```
#### 说明：
> 失败了，只打印异常日志即可，没有其他的操作。
