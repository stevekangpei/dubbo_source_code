# dubbo源码-集群容错-负载均衡之-RandomLoadBalance实现


### RandomLoadBalance

#### 成员变量
```java
    public static final String NAME = "random";

    private final Random random = new Random();
```

```java
    /**
     * (1) 先判断invoker的权重是否相等。如果相等了，直接随机取一个。
     * （2）如果权重不想等了
     *      随机从【0，totalWeight】中选择一个
     *      然后判断当前的随机值在哪一个区间
     *      去对应的invoker
     * @param invokers
     * @param url
     * @param invocation
     * @param <T>
     * @return
     */
    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        int length = invokers.size(); // Number of invokers
        int totalWeight = 0; // The sum of weights
        boolean sameWeight = true; // Every invoker has the same weight?
        // 计算总权限
        for (int i = 0; i < length; i++) {
            int weight = getWeight(invokers.get(i), invocation); // 获得权重
            totalWeight += weight; // Sum
            if (sameWeight && i > 0 && weight != getWeight(invokers.get(i - 1), invocation)) {
                sameWeight = false;
            }
        }
        // 权重不相等，随机后，判断在哪个 Invoker 的权重区间中
        if (totalWeight > 0 && !sameWeight) {
            // 随机
            // If (not every invoker has the same weight & at least one invoker's weight>0), select randomly based on totalWeight.
            int offset = random.nextInt(totalWeight);
            // Return a invoker based on the random value.
            // 区间判断
            for (Invoker<T> invoker : invokers) {
                offset -= getWeight(invoker, invocation);
                if (offset < 0) {
                    return invoker;
                }
            }
        }
        // 权重相等，平均随机
        // If all invokers have the same weight value or totalWeight=0, return evenly.
        return invokers.get(random.nextInt(length));
    }
```
#### 说明：是一个加权随机的机制。
