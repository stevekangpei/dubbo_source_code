# dubbo源码-集群容错-负载均衡之-一致性哈希 ConsistentHashLoadBalance


### 代码结构

```java
    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        String key = invokers.get(0).getUrl().getServiceKey() + "." + invocation.getMethodName();
        // 基于 invokers 集合，根据对象内存地址来计算定义哈希值
        int identityHashCode = System.identityHashCode(invokers);
        // 获得 ConsistentHashSelector 对象。若为空，或者定义哈希值变更（说明 invokers 集合发生变化），进行创建新的 ConsistentHashSelector 对象
        // 可能新增也可能减少了，此时 selector.identityHashCode != identityHashCode 条件成立
        ConsistentHashSelector<T> selector = (ConsistentHashSelector<T>) selectors.get(key);
        if (selector == null || selector.identityHashCode != identityHashCode) {
            selectors.put(key, new ConsistentHashSelector<T>(invokers, invocation.getMethodName(), identityHashCode));
            selector = (ConsistentHashSelector<T>) selectors.get(key);
        }
        return selector.select(invocation);
    }
```

#### 说明：
> 首先根据内存地址计算invoker的哈希值。
> 获得 ConsistentHashSelector 对象。若为空，说明invokerList可能变了，可能增加或者减少了
> 需要重新创建。
> 使用 ConsistentHashSelector 进行负载均衡。

 
```java
    private static final class ConsistentHashSelector<T> {

        /**
         * 虚拟节点与 Invoker 的映射关系
         */
        private final TreeMap<Long, Invoker<T>> virtualInvokers;
        /**
         * 每个Invoker 对应的虚拟节点数
         */
        private final int replicaNumber;
        /**
         * 定义哈希值
         */
        private final int identityHashCode;
        /**
         * 取值参数位置数组
         * 存放需要做hash的参数在方法中的是属于第几个参数
         */
        private final int[] argumentIndex;

        ConsistentHashSelector(List<Invoker<T>> invokers, String methodName, int identityHashCode) {
            this.virtualInvokers = new TreeMap<Long, Invoker<T>>();
            // 设置 identityHashCode
            this.identityHashCode = identityHashCode;
            URL url = invokers.get(0).getUrl();
            // 初始化 replicaNumber 虚拟节点的数量
            this.replicaNumber = url.getMethodParameter(methodName, "hash.nodes", 160);
            // 初始化 argumentIndex 相当于是 参与 hash 计算的参数下标值，默认对第一个参数进行 hash 运算
            // 0表示默认对第一个节点进行计算。

            String[] index = Constants.COMMA_SPLIT_PATTERN.split(url.getMethodParameter(methodName, "hash.arguments", "0"));
            argumentIndex = new int[index.length];
            for (int i = 0; i < index.length; i++) {
                argumentIndex[i] = Integer.parseInt(index[i]);
            }
            // 初始化 virtualInvokers
            for (Invoker<T> invoker : invokers) {
                String address = invoker.getUrl().getAddress();
                // 每四个虚拟结点为一组，
                for (int i = 0; i < replicaNumber / 4; i++) {
                    // 这组虚拟结点得到惟一名称
                    byte[] digest = md5(address + i);
                    // Md5是一个16字节长度的数组，将16字节的数组每四个字节一组，分别对应一个虚拟结点，对 address + i 进行 md5 运算，得到一个长度为16的字节数组
                    /**
                     *  对于每四个字节，组成一个long值，做为这个虚拟节点的在环中的惟一key
                     *  h = 0 时，取 digest 中下标为 0 ~ 3 的4个字节进行位运算
                     *  h = 1 时，取 digest 中下标为 4 ~ 7 的4个字节进行位运算
                     *  h = 2, h = 3 时过程同上
                     */
                    for (int h = 0; h < 4; h++) {
                        // 对于每四个字节，组成一个long值数值，做为这个虚拟节点的在环中的惟一key
                        long m = hash(digest, h);
                        virtualInvokers.put(m, invoker);
                    }
                }
            }
        }

        /**
         * 首先是对参数进行 md5 以及 hash 运算，得到一个 hash 值。然后再拿这个值到 TreeMap 中查找目标 Invoker 即可
         * @param invocation
         * @return
         */
        public Invoker<T> select(Invocation invocation) {
            // 基于方法参数，获得 KEY
            String key = toKey(invocation.getArguments());
            // 计算 MD5 值
            byte[] digest = md5(key);
            // 通过参数的md5值哈希找到对应的环中的虚拟节点。
            //
            return selectForKey(hash(digest, 0));
        }

        private String toKey(Object[] args) {
            StringBuilder buf = new StringBuilder();
            for (int i : argumentIndex) {
                if (i >= 0 && i < args.length) {
                    buf.append(args[i]);
                }
            }
            return buf.toString();
        }

        /**
         *  得到大于当前 key 的那个子 Map ，然后从中取出第一个 key ，就是大于且离它最近的那个 key
         *  不存在，则取 virtualInvokers 第一个
         * @param hash
         * @return
         */
        private Invoker<T> selectForKey(long hash) {
            Map.Entry<Long, Invoker<T>> entry = virtualInvokers.tailMap(hash, true).firstEntry();
        	if (entry == null) {
        		entry = virtualInvokers.firstEntry();
        	}
        	// 存在，则返回
        	return entry.getValue();
        }

        private long hash(byte[] digest, int number) {
            return (((long) (digest[3 + number * 4] & 0xFF) << 24)
                    | ((long) (digest[2 + number * 4] & 0xFF) << 16)
                    | ((long) (digest[1 + number * 4] & 0xFF) << 8)
                    | (digest[number * 4] & 0xFF))
                    & 0xFFFFFFFFL;
        }

        // 计算 MD5
        private byte[] md5(String value) {
            MessageDigest md5;
            try {
                md5 = MessageDigest.getInstance("MD5");
            } catch (NoSuchAlgorithmException e) {
                throw new IllegalStateException(e.getMessage(), e);
            }
            md5.reset();
            byte[] bytes;
            try {
                bytes = value.getBytes("UTF-8");
            } catch (UnsupportedEncodingException e) {
                throw new IllegalStateException(e.getMessage(), e);
            }
            md5.update(bytes);
            return md5.digest();
        }

    }
```

#### 说明：
> 初始化虚拟节点的数量，通过URL的参数来解析，默认为160个
> 初始化参数数组，这个数组的作用是用于将接口的参数作为待哈希的值。
> 循环每个 Invoker 对象。循环 replicaNumber / 4 次 因为 md5是一个16字节的数组。
> 真正select的时候
 *  （1）拼接参数
 *  （2）获取md5字节
 *  （3）哈希取得在环上的值
 *  （4）找到大于且离它最近的那个 key
