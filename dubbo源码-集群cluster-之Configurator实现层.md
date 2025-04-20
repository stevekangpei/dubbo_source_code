# dubbo源码-集群cluster-之Configurator实现层


### Configurator 使用场景
    
```
详情可见dubbo用户指南-配置规则
向注册中心写入动态配置覆盖规则，该功能通常由监控中心或治理中心的页面完成
```

```java
RegistryFactory registryFactory = ExtensionLoader.getExtensionLoader(RegistryFactory.class).getAdaptiveExtension();
Registry registry = registryFactory.getRegistry(URL.valueOf("zookeeper://10.20.153.10:2181"));
registry.register(URL.valueOf("override://0.0.0.0/com.foo.BarService?category=configurators&dynamic=false&application=foo&timeout=1000"));
```

#### 其中：

* override:// 表示数据采用覆盖方式，支持 override 和 absent，可扩展，必填。
* 0.0.0.0 表示对所有 IP 地址生效，如果只想覆盖某个 IP 的数据，请填入具体 IP，必填。
* com.foo.BarService 表示只对指定服务生效，必填。
* category=configurators 表示该数据为动态配置类型，必填。
* dynamic=false 表示该数据为持久数据，当注册方退出时，数据依然保存在注册中心，必填。
* enabled=true 覆盖规则是否生效，可不填，缺省生效。
* application=foo 表示只对指定应用生效，可不填，表示对所有应用生效。
* timeout=1000 表示将满足以上条件的 timeout 参数的值覆盖为 1000。如果想覆盖其它参数，直接加在 override 的 URL 参数上。


#### 使用例子：

* 1 禁用提供者：(通常用于临时踢除某台提供者机器，相似的，禁止消费者访问请使用路由规则)

```java
override://10.20.153.10/com.foo.BarService?category=configurators&dynamic=false&disbaled=true
```

* 2,调整权重：(通常用于容量评估，缺省权重为 100)

```java
override://10.20.153.10/com.foo.BarService?category=configurators&dynamic=false&weight=200
```

* 3 调整负载均衡策略：(缺省负载均衡策略为 random)

```java
override://10.20.153.10/com.foo.BarService?category=configurators&dynamic=false&loadbalance=leastactive
```


### Configurator接口

```java
public interface Configurator extends Comparable<Configurator> {
    /**
     * get the configurator url.
     *
     * 配置规则
     *
     * @return configurator url.
     */
    URL getUrl();

    /**
     * Configure the provider url.
     *
     * 配置到 URL 中
     *
     * @param url - old rovider url.
     * @return new provider url.
     */
    URL configure(URL url);
}
```

#### 说明：
* 一个 Configurator 对象，对应一条配置规则。
* Configurator 有优先级的要求，所以实现 Comparable 接口。
* #getUrl() 接口方法，获得配置 URL ，里面带有配置规则。
* #configure(Url url) 接口方法，设置配置规则到指定 URL 中。



### AbstractConfigurator

#### 构造方法
```java
    /**
     * 配置规则 URL
     */
    private final URL configuratorUrl;

    public AbstractConfigurator(URL url) {
        if (url == null) {
            throw new IllegalArgumentException("configurator url == null");
        }
        this.configuratorUrl = url;
    }
    
    @Override
    public URL getUrl() {
        return configuratorUrl;
    }

```
#### 说明：
* 实现 Configurator 接口，实现公用的配置规则的匹配、排序的逻辑。
* 主要是初始化了configuratorUrl。


#### compareTo方法， 用于配置规则的排序逻辑。

```java
    /**
     * 根据 host、priority 依次排序
     *
     * 1. 特定 host 优先级高于 anyhost 0.0.0.0
     * 2. priority值越大，优先级越高；
     */
    @Override
    public int compareTo(Configurator o) {
        if (o == null) {
            return -1;
        }
        // host 升序
        int ipCompare = getUrl().getHost().compareTo(o.getUrl().getHost());
        // 若 host 相同，按照 priority 降序
        if (ipCompare == 0) {//host is the same, sort by priority
            int i = getUrl().getParameter(Constants.PRIORITY_KEY, 0);
            int j = o.getUrl().getParameter(Constants.PRIORITY_KEY, 0);
            if (i < j) {
                return -1;
            } else if (i > j) {
                return 1;
            } else {
                return 0;
            }
        } else {
            return ipCompare;
        }
    }
```

#### 说明：
 * 根据 host、priority 依次排序
 * 1. 特定 host 优先级高于 anyhost 0.0.0.0
 * 2. priority值越大，优先级越高；



#### configure方法

```java
    /**
     * (1) 如果配置URL规则带有 port，说明肯定是provider， 可以在提供端生效 也可以在消费端生效
     * （2）如果是消费者端的话，只能在消费端生效
     * （3） 如果是0.0.0.0可以是控制提供端，也可以是控制提供端，控制所有提供端，地址必定是0.0.0.0，否则就要配端口从而执行上面的if分支了
     * @param url - old rovider url. 
     * @return
     */
    @Override
    public URL configure(URL url) {
        if (configuratorUrl.getHost() == null || url == null || url.getHost() == null) {
            return url;
        }
        // If override url has port, means it is a provider address. We want to control a specific provider with this override url, it may take effect on the specific provider instance or on consumers holding this provider instance.
        // 配置规则，URL 带有端口( port )，意图是控制提供者机器。可以在提供端生效 也可以在消费端生效
        if (configuratorUrl.getPort() != 0) {
            if (url.getPort() == configuratorUrl.getPort()) {
                return configureIfMatch(url.getHost(), url);
            }
        // override url don't have a port, means the ip override url specify is a consumer address or 0.0.0.0
        // 配置规则，URL 没有端口，override 输入消费端地址 或者 0.0.0.0
        } else {
            // 1.If it is a consumer ip address, the intention is to control a specific consumer instance, it must takes effect at the consumer side, any provider received this override url should ignore;
            // 2.If the ip is 0.0.0.0, this override url can be used on consumer, and also can be used on provider
            // 1. 如果是消费端地址，则意图是控制消费者机器，必定在消费端生效，提供端忽略；
            // 2. 如果是0.0.0.0可能是控制提供端，也可能是控制提供端
            if (url.getParameter(Constants.SIDE_KEY, Constants.PROVIDER).equals(Constants.CONSUMER)) {
                // NetUtils.getLocalHost是消费端注册到zk的消费者地址
                return configureIfMatch(NetUtils.getLocalHost(), url);// NetUtils.getLocalHost is the ip address consumer registered to registry.
            } else if (url.getParameter(Constants.SIDE_KEY, Constants.CONSUMER).equals(Constants.PROVIDER)) {
                // 控制所有提供端，地址必定是0.0.0.0，否则就要配端口从而执行上面的if分支了
                return configureIfMatch(Constants.ANYHOST_VALUE, url);// take effect on all providers, so address must be 0.0.0.0, otherwise it won't flow to this if branch
            }
        }
        return url;
    }
```

#### 说明：

 * (1) 如果配置URL规则带有 port，说明肯定是provider， 可以在提供端生效 也可以在消费端生效
 * （2）如果是消费者端的话，只能在消费端生效
 * （3） 如果是0.0.0.0可以是控制提供端，也可以是控制提供端，控制所有提供端，地址必定是0.0.0.0，否则就要配端口从而执行上面的if分支了

 

#### configureIfMatch方法

```java
    private URL configureIfMatch(String host, URL url) {
        // 匹配 Host
        if (Constants.ANYHOST_VALUE.equals(configuratorUrl.getHost()) || host.equals(configuratorUrl.getHost())) {
            // 匹配 "application"
            String configApplication = configuratorUrl.getParameter(Constants.APPLICATION_KEY, configuratorUrl.getUsername());
            String currentApplication = url.getParameter(Constants.APPLICATION_KEY, url.getUsername());
            if (configApplication == null || Constants.ANY_VALUE.equals(configApplication)
                    || configApplication.equals(currentApplication)) {
                // 配置 URL 中的条件 KEYS 集合。其中下面四个 KEY ，不算是条件，而是内置属性。考虑到下面要移除，所以添加到该集合中。
                Set<String> conditionKeys = new HashSet<String>();
                conditionKeys.add(Constants.CATEGORY_KEY);
                conditionKeys.add(Constants.CHECK_KEY);
                conditionKeys.add(Constants.DYNAMIC_KEY);
                conditionKeys.add(Constants.ENABLED_KEY);
                // 判断传入的 url 是否匹配配置规则 URL 的条件。除了 "application" 和 "side" 之外，带有 `"~"` 开头的 KEY ，也是条件。
                for (Map.Entry<String, String> entry : configuratorUrl.getParameters().entrySet()) {
                    String key = entry.getKey();
                    String value = entry.getValue();
                    if (key.startsWith("~") || Constants.APPLICATION_KEY.equals(key) || Constants.SIDE_KEY.equals(key)) {
                        conditionKeys.add(key);
                        // 若不相等，则不匹配配置规则，直接返回
                        if (value != null && !Constants.ANY_VALUE.equals(value)
                                && !value.equals(url.getParameter(key.startsWith("~") ? key.substring(1) : key))) {
                            return url;
                        }
                    }
                }
                // 移除条件 KEYS 集合，并配置到 URL 中
                return doConfigure(url, configuratorUrl.removeParameters(conditionKeys));
            }
        }
        return url;
    }

    protected abstract URL doConfigure(URL currentUrl, URL configUrl);
```

#### 说明：
* 1，配置 URL 中的条件 KEYS 集合。其中下面四个 KEY ，不算是条件，而是内置属性。考虑到下面要移除，所以添加到该集合中。
* 2，判断传入的 url 是否匹配配置规则 URL 的条件。除了 "application" 和 "side" 之外，带有 "~" 开头的 KEY ，也是条件。
* 3，若不相等，则不匹配配置规则，直接返回 url 。
* 4，从 configuratorUrl 移除条件 KEYS 集合，并调用 #doConfigure(URL currentUrl, URL configUrl) 抽象方法，实现子类设置配置规则到 url 中。


### OverrideConfigurator

```java
public class OverrideConfigurator extends AbstractConfigurator {

    public OverrideConfigurator(URL url) {
        super(url);
    }

    @Override
    public URL doConfigure(URL currentUrl, URL configUrl) {
        return currentUrl.addParameters(configUrl.getParameters()); // 覆盖添加
    }
}
```

#### doConfigure方法：
* 当前URL 覆盖式的添加参数



#### URL#addParameters方法

```java
    public URL addParameters(Map<String, String> parameters) {
        if (parameters == null || parameters.size() == 0) {
            return this;
        }

        boolean hasAndEqual = true;
        for (Map.Entry<String, String> entry : parameters.entrySet()) {
            String value = getParameters().get(entry.getKey());
            if (value == null) {
                if (entry.getValue() != null) {
                    hasAndEqual = false;
                    break;
                }
            } else {
                if (!value.equals(entry.getValue())) {
                    hasAndEqual = false;
                    break;
                }
            }
        }
        // return immediately if there's no change
        if (hasAndEqual) return this;

        Map<String, String> map = new HashMap<String, String>(getParameters());
        map.putAll(parameters);
        return new URL(protocol, username, password, host, port, path, map);
    }
```

#### 说明：
* URL类的addParameters 方法
* 如果parameters没有改变的话，那么直接返回
* 否则将所有的参数put进去，覆盖式的返回一个新的URL


### AbsentConfigurator
```java
public class AbsentConfigurator extends AbstractConfigurator {

    public AbsentConfigurator(URL url) {
        super(url);
    }

    @Override
    public URL doConfigure(URL currentUrl, URL configUrl) {
        return currentUrl.addParametersIfAbsent(configUrl.getParameters());     
    }
}
```
#### 说明：
* 不存在的时候添加参数。

