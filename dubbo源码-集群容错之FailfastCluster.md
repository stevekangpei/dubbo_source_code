# dubbo源码-集群容错之FailfastCluster

### 代码逻辑

#### FailfastCluster
 失败就直接返回，通常用于非幂等性的操作。
 
```java
public class FailfastCluster implements Cluster {

    public final static String NAME = "failfast";

    @Override
    public <T> Invoker<T> join(Directory<T> directory) throws RpcException {
        return new FailfastClusterInvoker<T>(directory);
    }
}
```

#### FailfastClusterInvoker
```java
    @Override
    public Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        checkInvokers(invokers, invocation);
        // 根据负载均衡机制从 invokers 中选择一个Invoker
        Invoker<T> invoker = select(loadbalance, invocation, invokers, null);
        try {
            // RPC 调用得到 Result
            return invoker.invoke(invocation);
        } catch (Throwable e) {
            // 若是业务性质的异常，直接抛出
            if (e instanceof RpcException && ((RpcException) e).isBiz()) { // biz exception.
                throw (RpcException) e;
            }
            // 封装 RpcException 异常，并抛出
            throw new RpcException(e instanceof RpcException ? ((RpcException) e).getCode() : 0,
                    "Failfast invoke providers " + invoker.getUrl() + " " + loadbalance.getClass().getSimpleName() + " select from all providers " + invokers + " for service " + getInterface().getName() + " method " + invocation.getMethodName() + " on consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion() + ", but no luck to perform the invocation. Last error is: " + e.getMessage(), e.getCause() != null ? e.getCause() : e);
        }
    }
```
#### 说明：
> 选择一个invoker，调用。失败了抛出异常，不做容错处理
> Usually used for non-idempotent write operations
> http://en.wikipedia.org/wiki/Fail-fast