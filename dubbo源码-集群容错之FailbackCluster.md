# dubbo源码-集群容错之FailbackCluster

### 代码逻辑

#### FailbackCluster

```java
public class FailbackCluster implements Cluster {

    public final static String NAME = "failback";

    @Override
    public <T> Invoker<T> join(Directory<T> directory) throws RpcException {
        return new FailbackClusterInvoker<T>(directory);
    }
}
```

#### FailbackClusterInvoker

```java
@Override
protected Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    try {
        // 检查 invokers 即可用Invoker集合是否为空，
        checkInvokers(invokers, invocation);
        Invoker<T> invoker = select(loadbalance, invocation, invokers, null);
        // RPC 调用得到 Result
        return invoker.invoke(invocation);
    } catch (Throwable e) {
        logger.error("Failback to invoke method " + invocation.getMethodName() + ", wait for retry in background. Ignored exception: " + e.getMessage() + ", ", e);
        // 添加到失败任务
        addFailed(invocation, this);
        return new RpcResult(); // ignore
    }
}
```

#### 说明： 
> 根据负载均衡找到一个invoker，然后发起调用。
> 失败的话，#addFailed

```java
private void addFailed(Invocation invocation, AbstractClusterInvoker<?> router) {
    // 若定时任务未初始化，进行创建
    if (retryFuture == null) {
        synchronized (this) {
            if (retryFuture == null) {
                retryFuture = scheduledExecutorService.scheduleWithFixedDelay(new Runnable() {

                    public void run() {
                        // collect retry statistics
                        try {
                            retryFailed();
                        } catch (Throwable t) { // Defensive fault tolerance
                            logger.error("Unexpected error occur at collect statistic", t);
                        }
                    }
                }, RETRY_FAILED_PERIOD, RETRY_FAILED_PERIOD, TimeUnit.MILLISECONDS);
            }
        }
    }
    // 添加到失败任务
    failed.put(invocation, router);
}
```
#### 说明：
> 如果定时任务线程没有创建，就创建，否则就直接添加到定时任务线程里面去。
> 定时任务线程会周期执行 #retryFailed 方法

```java
void retryFailed() {
    if (failed.size() == 0) {
        return;
    }
    // 循环重试任务，逐个调用
    for (Map.Entry<Invocation, AbstractClusterInvoker<?>> entry : new HashMap<Invocation, AbstractClusterInvoker<?>>(failed).entrySet()) { // 创建集合
        Invocation invocation = entry.getKey();
        Invoker<?> invoker = entry.getValue();
        try {
            // RPC 调用得到 Result
            invoker.invoke(invocation);
            // 移除失败任务
            failed.remove(invocation);
        } catch (Throwable e) {
            logger.error("Failed retry to invoke method " + invocation.getMethodName() + ", waiting again.", e);
        }
    }
}

```
#### 说明：
> 执行每个之前失败的任务，执行成功就删除
> 失败自动恢复，后台记录失败请求，定时重发。