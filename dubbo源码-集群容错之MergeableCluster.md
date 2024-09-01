# dubbo源码-集群容错之MergeableCluster


### 代码逻辑

#### MergeableCluster
```java
public class MergeableCluster implements Cluster {

    public static final String NAME = "mergeable";

    @Override
    public <T> Invoker<T> join(Directory<T> directory) throws RpcException {
        return new MergeableClusterInvoker<T>(directory);
    }
}
```


#### MergeableClusterInvoker
```java

    /**
     * （1）先通过directory获得invokerList
     * （2）如果没有配置merger扩展的话，那么找到第一个可用的invoker直接调用。
     * （3）通过反射获取方法的返回值类型
     * （4）提交线程池，并行执行，发起 RPC 调用，并添加到 results 中，阻塞等待结果返回、。
     * （5）结果大小为空，返回空的 RpcResult，为一个的话，返回第一个结果。
     * （6）进行合并：
     *      （1）如果是以. 为开头的，说明是使用自定义的合并方法。
     *            （1）如果方法返回结果一样，直接设置方法属性，invoke即可
     *            （2）如果merge方法返回结果不一样，，合并时，不修改 result
     *      （2）通过默认的实现来返回结果
     *          （1）如果是default。通过 MergerFactory.getMerger 来获取merge实现类
     *          （2）否则通过 ExtensionLoader.getExtensionLoader(Merger.class).getExtension(merger);
     *              通过SPI扩展加载merger实现。
     *          （3）有 Merger ，循环调用 Merger#merge(T... items) 方法，进行合并。
     *              没有merger，抛出异常。
     *
     *
     * @param invocation
     * @return
     * @throws RpcException
     */

    @SuppressWarnings("rawtypes")
    public Result invoke(final Invocation invocation) throws RpcException {
        // 获得 Invoker 集合
        List<Invoker<T>> invokers = directory.list(invocation);
        // 获得 Merger 拓展名
        String merger = getUrl().getMethodParameter(invocation.getMethodName(), Constants.MERGER_KEY);
        // 若果未配置拓展，直接调用首个可用的 Invoker 对象
        if (ConfigUtils.isEmpty(merger)) { // If a method doesn't have a merger, only invoke one Group
            for (final Invoker<T> invoker : invokers) {
                if (invoker.isAvailable()) {
                    return invoker.invoke(invocation);
                }
            }
            return invokers.iterator().next().invoke(invocation);
        }

        // 通过反射，获得返回类型
        Class<?> returnType;
        try {
            returnType = getInterface().getMethod(invocation.getMethodName(), invocation.getParameterTypes()).getReturnType();
        } catch (NoSuchMethodException e) {
            returnType = null;
        }

        // 提交线程池，并行执行，发起 RPC 调用，并添加到 results 中
        Map<String, Future<Result>> results = new HashMap<String, Future<Result>>();
        for (final Invoker<T> invoker : invokers) {
            Future<Result> future = executor.submit(new Callable<Result>() {
                public Result call() {
                    // RPC 调用
                    return invoker.invoke(new RpcInvocation(invocation, invoker));
                }
            });
            results.put(invoker.getUrl().getServiceKey(), future);
        }

        // 阻塞等待执行执行结果，并添加到 resultList 中
        List<Result> resultList = new ArrayList<Result>(results.size());
        int timeout = getUrl().getMethodParameter(invocation.getMethodName(), Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
        for (Map.Entry<String, Future<Result>> entry : results.entrySet()) {
            Future<Result> future = entry.getValue();
            try {
                Result r = future.get(timeout, TimeUnit.MILLISECONDS);
                if (r.hasException()) { // 异常 Result ，打印错误日志，忽略
                    log.error(new StringBuilder(32).append("Invoke ").append(getGroupDescFromServiceKey(entry.getKey())).append(" failed: ").append(r.getException().getMessage()).toString(), r.getException());
                } else { // 正常 Result ，添加到 resultList 中
                    resultList.add(r);
                }
            } catch (Exception e) { // 异常，抛出 RpcException 异常
                throw new RpcException(new StringBuilder(32).append("Failed to invoke service ").append(entry.getKey()).append(": ").append(e.getMessage()).toString(), e);
            }
        }

        // 结果大小为空，返回空的 RpcResult
        if (resultList.isEmpty()) {
            return new RpcResult((Object) null);
        // 结果大小为 1 ，返回首个 RpcResult
        } else if (resultList.size() == 1) {
            return resultList.iterator().next();
        }
        // 返回类型为 void ，返回空的 RpcResult
        if (returnType == void.class) {
            return new RpcResult((Object) null);
        }

        Object result;
        // 【第 1 种】基于合并方法
        if (merger.startsWith(".")) {
            // 获得合并方法 Method
            merger = merger.substring(1);
            Method method;
            try {
                method = returnType.getMethod(merger, returnType);
            } catch (NoSuchMethodException e) {
                throw new RpcException(new StringBuilder(32).append("Can not merge result because missing method [ ").append(merger).append(" ] in class [ ").append(returnType.getClass().getName()).append(" ]").toString());
            }
            // 有 Method ，进行合并
            if (method != null) {
                if (!Modifier.isPublic(method.getModifiers())) {
                    method.setAccessible(true);
                }
                result = resultList.remove(0).getValue();
                try {
                    // 方法返回类型匹配，合并时，修改 result
                    if (method.getReturnType() != void.class && method.getReturnType().isAssignableFrom(result.getClass())) {
                        for (Result r : resultList) {
                            result = method.invoke(result, r.getValue());
                        }
                    // 方法返回类型不匹配，合并时，不修改 result
                    } else {
                        for (Result r : resultList) {
                            method.invoke(result, r.getValue());
                        }
                    }
                } catch (Exception e) {
                    throw new RpcException(new StringBuilder(32).append("Can not merge result: ").append(e.getMessage()).toString(), e);
                }
            // 无 Method ，抛出 RpcException 异常
            } else {
                throw new RpcException(new StringBuilder(32).append("Can not merge result because missing method [ ").append(merger).append(" ] in class [ ").append(returnType.getClass().getName()).append(" ]").toString());
            }
        // 【第 2 种】基于 Merger
        } else {
            Merger resultMerger;
            // 【第 2.1 种】根据返回值类型自动匹配 Merger
            if (ConfigUtils.isDefault(merger)) {
                resultMerger = MergerFactory.getMerger(returnType);
            // 【第 2.2 种】指定 Merger
            } else {
                resultMerger = ExtensionLoader.getExtensionLoader(Merger.class).getExtension(merger);
            }
            // 有 Merger ，进行合并
            if (resultMerger != null) {
                List<Object> rets = new ArrayList<Object>(resultList.size());
                for (Result r : resultList) {
                    rets.add(r.getValue());
                }
                result = resultMerger.merge(rets.toArray((Object[]) Array.newInstance(returnType, 0)));
            // 无 Merger ，抛出 RpcException 异常
            } else {
                throw new RpcException("There is no merger to merge result.");
            }
        }
        // 返回 RpcResult 结果
        return new RpcResult(result);
    }

```

#### 说明：
 * （1）先通过directory获得invokerList
 * （2）如果没有配置merger扩展的话，那么找到第一个可用的invoker直接调用。
 * （3）通过反射获取方法的返回值类型
 * （4）提交线程池，并行执行，发起 RPC 调用，并添加到 results 中，阻塞等待结果返回、。
 * （5）结果大小为空，返回空的 RpcResult，为一个的话，返回第一个结果。
 * （6）进行合并：
 *      （1）如果是以. 为开头的，说明是使用自定义的合并方法。
 *            （1）如果方法返回结果一样，直接设置方法属性，invoke即可
 *            （2）如果merge方法返回结果不一样，，合并时，不修改 result
 *      （2）通过默认的实现来返回结果
 *          （1）如果是default。通过 MergerFactory.getMerger 来获取merge实现类
 *          （2）否则通过 ExtensionLoader.getExtensionLoader(Merger.class).getExtension(merger);
 *              通过SPI扩展加载merger实现。
 *          （3）有 Merger ，循环调用 Merger#merge(T... items) 方法，进行合并。
 *              没有merger，抛出异常。

