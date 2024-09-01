# dubbo源码-集群容错-之AvailableClusterInvoker

### 代码逻辑

#### AvailableCluster
com.alibaba.dubbo.rpc.cluster.support.AvailableCluster ，实现 Cluster 接口.
只要第一个invoker可以用，就直接调用返回。

```java
public class AvailableCluster implements Cluster {

    public static final String NAME = "available";

    public <T> Invoker<T> join(Directory<T> directory) throws RpcException {
        return new AvailableClusterInvoker<T>(directory);
    }

}
```

#### AvailableClusterInvoker
```java
public class AvailableClusterInvoker<T> extends AbstractClusterInvoker<T> {

    public AvailableClusterInvoker(Directory<T> directory) {
        super(directory);
    }

    @Override
    public Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        // 循环候选的 Invoker 集合，调用首个可用的 Invoker 对象。
        for (Invoker<T> invoker : invokers) {
            if (invoker.isAvailable()) { // 可用
                // 发起 RPC 调用
                return invoker.invoke(invocation);
            }
        }
        throw new RpcException("No provider available in " + invokers);
    }
}
```
#### 说明： 代码相当简单，只要找到第一个可用的invoker 就发起调用。
