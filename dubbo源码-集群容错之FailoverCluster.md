# dubbo源码-集群容错之FailoverCluster


### 代码逻辑

```
When invoke fails, log the initial error and retry other invokers (retry n times, which means at most n different invokers will be invoked)
 Note that retry causes latency.
失败自动切换，当出现失败，重试其它服务器。通常用于读操作，但重试会带来更长延迟。可通过 retries="2" 来设置重试次数
```

#### FailoverCluster
```java
public class FailoverCluster implements Cluster {

    public final static String NAME = "failover";

    @Override
    public <T> Invoker<T> join(Directory<T> directory) throws RpcException {
        return new FailoverClusterInvoker<T>(directory);
    }

}
```


#### FailoverClusterInvoker
```java
    @Override
    public Result doInvoke(Invocation invocation, final List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        List<Invoker<T>> copyinvokers = invokers;
        // 检查copyinvokers即可用Invoker集合是否为空，如果为空，那么抛出异常
        checkInvokers(copyinvokers, invocation);
        // 得到最大可调用次数：最大可重试次数+1，默认最大可重试次数Constants.DEFAULT_RETRIES=2
        int len = getUrl().getMethodParameter(invocation.getMethodName(), Constants.RETRIES_KEY, Constants.DEFAULT_RETRIES) + 1;
        if (len <= 0) {
            len = 1;
        }
        
        RpcException le = null; // last exception.
        // 保存已经调用过的Invoker
        List<Invoker<T>> invoked = new ArrayList<Invoker<T>>(copyinvokers.size()); // invoked invokers.
        Set<String> providers = new HashSet<String>(len);
        // failover机制核心实现：如果出现调用失败，那么重试其他服务器
        for (int i = 0; i < len; i++) {
            // Reselect before retry to avoid a change of candidate `invokers`.
            // NOTE: if `invokers` changed, then `invoked` also lose accuracy.
            // 重试时，进行重新选择，避免重试时invoker列表已发生变化.
            // 注意：如果列表发生了变化，那么invoked判断会失效，因为invoker示例已经改变
            if (i > 0) {
                checkWhetherDestroyed();
                // 根据Invocation调用信息从Directory中获取所有可用Invoker
                copyinvokers = list(invocation);
                // check again
                // 重新检查一下
                checkInvokers(copyinvokers, invocation);
            }
            // 根据负载均衡机制从copyinvokers中选择一个Invoker
            Invoker<T> invoker = select(loadbalance, invocation, copyinvokers, invoked);
            // 保存每次调用的Invoker
            invoked.add(invoker);
            // 设置已经调用的 Invoker 集合，到 Context 中
            RpcContext.getContext().setInvokers((List) invoked);
            try {
                // RPC 调用得到 Result
                Result result = invoker.invoke(invocation);
                if (le != null && logger.isWarnEnabled()) {
                    logger.warn("Although retry the method " + invocation.getMethodName()
                            + " in the service " + getInterface().getName()
                            + " was successful by the provider " + invoker.getUrl().getAddress()
                            + ", but there have been failed providers " + providers
                            + " (" + providers.size() + "/" + copyinvokers.size()
                            + ") from the registry " + directory.getUrl().getAddress()
                            + " on the consumer " + NetUtils.getLocalHost()
                            + " using the dubbo version " + Version.getVersion() + ". Last error is: "
                            + le.getMessage(), le);
                }
                return result;
            } catch (RpcException e) {
                // 如果是业务性质的异常，不再重试，直接抛出
                if (e.isBiz()) { // biz exception.
                    throw e;
                }
                // 其他性质的异常统一封装成RpcException
                le = e;
            } catch (Throwable e) {
                le = new RpcException(e.getMessage(), e);
            } finally {
                providers.add(invoker.getUrl().getAddress());
            }
        }
        // 最大可调用次数用完还得到Result的话，抛出RpcException异常：重试了N次还是失败，并输出最后一次异常信息
        throw new RpcException(le.getCode(), "Failed to invoke the method " + invocation.getMethodName()
                + " in the service " + getInterface().getName()
                + ". Tried " + len + " times of the providers "
                + providers + " (" + providers.size() + "/" + copyinvokers.size()
                + ") from the registry " + directory.getUrl().getAddress()
                + " on the consumer " + NetUtils.getLocalHost()
                + " using the dubbo version " + Version.getVersion()
                + ". Last error is: " + le.getMessage(), le.getCause() != null ? le.getCause() : le);
    }

```

#### 说明：
> 
>
>
>