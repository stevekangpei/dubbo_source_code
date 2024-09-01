# dubbo源码-集群cluster-之StaticDirectory

```实现 AbstractDirectory 抽象类，静态 Directory 实现类。逻辑比较简单，将传入的 invokers 集合，封装成静态的 Directory 对象。
```


### 构造器
```java
    public StaticDirectory(URL url, List<Invoker<T>> invokers, List<Router> routers) {
        // 默认使用 `url` 参数。当它为空时，使用 `invokers[0].url` 。
        super(url == null && invokers != null && !invokers.isEmpty() ? invokers.get(0).getUrl() : url, routers);
        if (invokers == null || invokers.isEmpty()) {
            throw new IllegalArgumentException("invokers == null");
        }
        this.invokers = invokers;
    }
    
        @Override
    public Class<T> getInterface() {
        return invokers.get(0).getInterface();
    }

    @Override
    public boolean isAvailable() {
        // 若已经销毁，则不可用
        if (isDestroyed()) {
            return false;
        }
        // 任一一个 Invoker 可用，则为可用
        for (Invoker<T> invoker : invokers) {
            if (invoker.isAvailable()) {
                return true;
            }
        }
        return false;
    }

    @Override
    public void destroy() {
        // 若已经销毁， 跳过
        if (isDestroyed()) {
            return;
        }
        // 销毁
        super.destroy();
        // 销毁每个 Invoker
        for (Invoker<T> invoker : invokers) {
            invoker.destroy();
        }
        // 清空 Invoker 集合
        invokers.clear();
    }

    @Override
    protected List<Invoker<T>> doList(Invocation invocation) throws RpcException {
        return invokers;
    }

```
#### 说明：
> 通过传入invokerList 初始化staticDirectory。
> 任意一个invoker可以用，就认为available，#isAvailable
> doList方法，就是将传入的invoker列表返回即可。



