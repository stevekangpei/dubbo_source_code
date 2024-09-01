# dubbo源码-集群容错-负载均衡之-LeastActiveLoadBalance 最小连接数负载均衡


### 代码结构
```java
    /**
     * （1）计算最小连接的数量和，invoker的权重。
     *      （1）遍历invokerList
     *      （2）如果发现更小的活跃数，重新开始计算
     *      （3）如果发现了同样的活跃数，累计相同最小活跃数下标，并计算总权重
     *      (4) leastIndexes 这个数组记录了最小活跃数组的idx。
     *
     * （2）如果最小连接只有一个节点，直接返回
     * （3）如果相同连接数量的有多个，且权重不相同。
     *      则和加权随机的机制类似。根据权重看落到哪个区间取对应的invoker
     * （4）如果权重相同或权重为0则均等随机
     * @param invokers
     * @param url
     * @param invocation
     * @param <T>
     * @return
     */
    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        int length = invokers.size(); // 总个数
        int leastActive = -1; // 最小的活跃数
        int leastCount = 0; // 相同最小活跃数的个数
        int[] leastIndexes = new int[length]; // 相同最小活跃数的下标
        int totalWeight = 0; // 总权重
        int firstWeight = 0; // 第一个权重，用于于计算是否相同
        boolean sameWeight = true; // 是否所有权重相同
        // 计算获得相同最小活跃数的数组和个数
        for (int i = 0; i < length; i++) {
            Invoker<T> invoker = invokers.get(i);
            int active = RpcStatus.getStatus(invoker.getUrl(), invocation.getMethodName()).getActive(); // 活跃数
            int weight = invoker.getUrl().getMethodParameter(invocation.getMethodName(), Constants.WEIGHT_KEY, Constants.DEFAULT_WEIGHT); // 权重
            if (leastActive == -1 || active < leastActive) { // 发现更小的活跃数，重新开始
                leastActive = active; // 记录最小活跃数
                leastCount = 1; // 重新统计相同最小活跃数的个数
                leastIndexes[0] = i; // 重新记录最小活跃数下标
                totalWeight = weight; // 重新累计总权重
                firstWeight = weight; // 记录第一个权重
                sameWeight = true; // 还原权重相同标识
            } else if (active == leastActive) { // 累计相同最小的活跃数
                leastIndexes[leastCount++] = i; // 累计相同最小活跃数下标
                totalWeight += weight; // 累计总权重
                // 判断所有权重是否一样
                if (sameWeight && weight != firstWeight) {
                    sameWeight = false;
                }
            }
        }
        // assert(leastCount > 0)
        if (leastCount == 1) {
            // 如果只有一个最小则直接返回
            return invokers.get(leastIndexes[0]);
        }
        if (!sameWeight && totalWeight > 0) {
            // 如果权重不相同且权重大于0则按总权重数随机
            int offsetWeight = random.nextInt(totalWeight);
            // 并确定随机值落在哪个片断上
            for (int i = 0; i < leastCount; i++) {
                int leastIndex = leastIndexes[i];
                offsetWeight -= getWeight(invokers.get(leastIndex), invocation);
                if (offsetWeight <= 0) {
                    return invokers.get(leastIndex);
                }
            }
        }
        // 如果权重相同或权重为0则均等随机
        return invokers.get(leastIndexes[random.nextInt(leastCount)]);
    }

```
#### 说明：
> 计算获得相同最小活跃数的数组( leastIndexes )和个数( leastCount )。注意，leastIndexes 是重用的，所以需要 leastCount 作为下标。
> 每个 Invoker 的活跃数计算，通过 RpcStatus 获取。每次调用的时候，通过Filter过滤器
ActiveLimitFilter 记录了调用的次数。

```java
com.alibaba.dubbo.rpc.filter.ActiveLimitFilter
            long begin = System.currentTimeMillis();
            // 调用开始的计数
            RpcStatus.beginCount(url, methodName);
            try {
                // 服务调用
                Result result = invoker.invoke(invocation);
                // 调用结束的计数（成功）
                RpcStatus.endCount(url, methodName, System.currentTimeMillis() - begin, true);
                return result;
            } catch (RuntimeException t) {
                // 调用结束的计数（失败）
                RpcStatus.endCount(url, methodName, System.currentTimeMillis() - begin, false);
                throw t;
            }
```

> 权重不相等，随机权重后，判断在哪个 Invoker 的权重区间中。 权重相等，直接选中。