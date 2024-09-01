# dubbo源码-集群容错之ForkingCluster


### 代码逻辑

```Invoke a specific number of invokers concurrently, usually used for demanding real-time operations, but need to waste more service resources.

并行调用invoker，取最快返回的，比较消耗资源。
```

#### ForkingCluster
```java
public class ForkingCluster implements Cluster {

    public final static String NAME = "forking";

    @Override
    public <T> Invoker<T> join(Directory<T> directory) throws RpcException {
        return new ForkingClusterInvoker<T>(directory);
    }
}
```



#### ForkingClusterInvoker
```java
    @Override
    @SuppressWarnings({"unchecked", "rawtypes"})
    public Result doInvoke(final Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        // 检查 invokers 即可用Invoker集合是否为空，如果为空，那么抛出异常
        checkInvokers(invokers, invocation);
        // 保存选择的 Invoker 集合
        final List<Invoker<T>> selected;
        // 得到最大并行数，默认为 Constants.DEFAULT_FORKS = 2
        final int forks = getUrl().getParameter(Constants.FORKS_KEY, Constants.DEFAULT_FORKS);
        // 获得调用超时时间，默认为 DEFAULT_TIMEOUT = 1000 毫秒
        final int timeout = getUrl().getParameter(Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
        // 若最大并行书小于等于 0，或者大于 invokers 的数量，直接使用 invokers
        if (forks <= 0 || forks >= invokers.size()) {
            selected = invokers;
        } else {
            // 循环，根据负载均衡机制从 invokers，中选择一个个Invoker ，从而组成 Invoker 集合。
            // 注意，因为增加了排重逻辑，所以不能保证获得的 Invoker 集合的大小，小于最大并行数
            selected = new ArrayList<Invoker<T>>();
            for (int i = 0; i < forks; i++) {
                // 在invoker列表(排除selected)后,如果没有选够,则存在重复循环问题.见select实现.
                Invoker<T> invoker = select(loadbalance, invocation, invokers, selected);
                if (!selected.contains(invoker)) {  //防止重复添加invoker
                    selected.add(invoker);
                }
            }
        }
        // 设置已经调用的 Invoker 集合，到 Context 中
        RpcContext.getContext().setInvokers((List) selected);
        // 异常计数器
        final AtomicInteger count = new AtomicInteger();
        // 创建阻塞队列
        final BlockingQueue<Object> ref = new LinkedBlockingQueue<Object>();
        // 循环 selected 集合，提交线程池，发起 RPC 调用
        for (final Invoker<T> invoker : selected) {
            executor.execute(new Runnable() {
                public void run() {
                    try {
                        // RPC 调用，获得 Result 结果
                        Result result = invoker.invoke(invocation);
                        // 添加 Result 到 `ref` 阻塞队列
                        ref.offer(result);
                    } catch (Throwable e) {
                        // 异常计数器 + 1
                        int value = count.incrementAndGet();
                        // 若 RPC 调结果都是异常，则添加异常到 `ref` 阻塞队列
                        if (value >= selected.size()) {
                            ref.offer(e);
                        }
                    }
                }
            });
        }
        try {
            // 从 `ref` 队列中，阻塞等待结果
            Object ret = ref.poll(timeout, TimeUnit.MILLISECONDS);
            // 若是异常结果，抛出 RpcException 异常
            if (ret instanceof Throwable) {
                Throwable e = (Throwable) ret;
                throw new RpcException(e instanceof RpcException ? ((RpcException) e).getCode() : 0, "Failed to forking invoke provider " + selected + ", but no luck to perform the invocation. Last error is: " + e.getMessage(), e.getCause() != null ? e.getCause() : e);
            }
            // 若是正常结果，直接返回
            return (Result) ret;
        } catch (InterruptedException e) {
            throw new RpcException("Failed to forking invoke provider " + selected + ", but no luck to perform the invocation. Last error is: " + e.getMessage(), e);
        }
    }

```

#### 说明：
> 获取最大的并行数和超时时间。
> 如果并行数为0，则用所有的invoker
> 否则的话，循环forks次，调用loadbalance，获取forks个invoker，
> 同时避免重复调用
> 然后开启一个线程调用，并获取结果。