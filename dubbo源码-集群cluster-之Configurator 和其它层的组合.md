# dubbo源码-集群cluster-之Configurator 和其它层的组合



```
RegistryDirectory 和RegistryProtocol集成了Configurator，
用于实现相关配置规则的动态配置
```

### RegistryDirectory

```java
    @Override
    public synchronized void notify(List<URL> urls) {
        // 根据 URL 的分类或协议，分组成三个集合 。
        List<URL> invokerUrls = new ArrayList<URL>(); // 服务提供者 URL 集合
        List<URL> routerUrls = new ArrayList<URL>();
        List<URL> configuratorUrls = new ArrayList<URL>();
        for (URL url : urls) {
            String protocol = url.getProtocol();
            String category = url.getParameter(Constants.CATEGORY_KEY, Constants.DEFAULT_CATEGORY);
            if (Constants.ROUTERS_CATEGORY.equals(category) || Constants.ROUTE_PROTOCOL.equals(protocol)) {
                routerUrls.add(url);
            } else if (Constants.CONFIGURATORS_CATEGORY.equals(category) || Constants.OVERRIDE_PROTOCOL.equals(protocol)) {
                configuratorUrls.add(url);
            } else if (Constants.PROVIDERS_CATEGORY.equals(category)) {
                invokerUrls.add(url);
            } else {
                logger.warn("Unsupported category " + category + " in notified url: " + url + " from registry " + getUrl().getAddress() + " to consumer " + NetUtils.getLocalHost());
            }
        }
        // 处理配置规则 URL 集合
        // configurators
        if (!configuratorUrls.isEmpty()) {
            this.configurators = toConfigurators(configuratorUrls);
        }
.........// 省略其他方法
```

#### 说明：
* 当配置发生变更的时候，会通过注册中心调用到Zookeeper对应的Registry，最后会调用到RegistryDirectory的notify方法。
* notify方法将URL进行分类，归类，对于Configurator URL，通过调用 #toConfigurators 方法更新
  Directory中的cofigurator。

#### toConfigurators

```java
    /**
     * Convert override urls to map for use when re-refer.
     * Send all rules every time, the urls will be reassembled and calculated
     *
     * @param urls Contract:
     *             </br>1.override://0.0.0.0/...( or override://ip:port...?anyhost=true)&para1=value1... means global rules (all of the providers take effect)
     *             </br>2.override://ip:port...?anyhost=false Special rules (only for a certain provider)
     *             </br>3.override:// rule is not supported... ,needs to be calculated by registry itself.
     *             </br>4.override://0.0.0.0/ without parameters means clearing the override
     * @return
     */
    /**
     * 将overrideURL 转换为 map，供重新 refer 时使用.
     * 每次下发全部规则，全部重新组装计算
     *
     * @param urls 契约：
     *             </br>1.override://0.0.0.0/...(或override://ip:port...?anyhost=true)&para1=value1...表示全局规则(对所有的提供者全部生效)
     *             </br>2.override://ip:port...?anyhost=false 特例规则（只针对某个提供者生效）
     *             </br>3.不支持override://规则... 需要注册中心自行计算.
     *             </br>4.不带参数的override://0.0.0.0/ 表示清除override
     *
     * @return Configurator 集合
     */

      public static List<Configurator> toConfigurators(List<URL> urls) {
        // 忽略，若配置规则 URL 集合为空
        if (urls == null || urls.isEmpty()) {
            return Collections.emptyList();
        }

        // 创建 Configurator 集合
        List<Configurator> configurators = new ArrayList<Configurator>(urls.size());
        for (URL url : urls) {
            // 若协议为 `empty://` ，意味着清空所有配置规则，因此返回空 Configurator 集合
            if (Constants.EMPTY_PROTOCOL.equals(url.getProtocol())) {
                configurators.clear();
                break;
            }
            // 对应第 4 条契约，不带参数的 override://0.0.0.0/ 表示清除 override
            Map<String, String> override = new HashMap<String, String>(url.getParameters());
            // The anyhost parameter of override may be added automatically, it can't change the judgement of changing url
            // override 上的 anyhost 可能是自动添加的，不能影响改变url判断
            override.remove(Constants.ANYHOST_KEY);
            if (override.size() == 0) {
                configurators.clear();
                continue;
            }
            // 获得 Configurator 对象，并添加到 `configurators` 中
            configurators.add(configuratorFactory.getConfigurator(url));
        }
        // 排序
        Collections.sort(configurators);
        return configurators;
}
```

#### 说明：
* 若协议为 `empty://` ，意味着清空所有配置规则，因此返回空 Configurator 集合,即协议的protocol为empty://
* 不带参数的 override://0.0.0.0/ 表示清除 override
* 否则的话，使用ConfiguratorFactory加载Configurator。
* 排序，并返回Configurator。

#### notify方法

```java
      .....// 省略其他内容
        // 合并配置规则，到 `directoryUrl` 中，形成 `overrideDirectoryUrl` 变量。
        List<Configurator> localConfigurators = this.configurators; // local reference
        // merge override parameters
        this.overrideDirectoryUrl = directoryUrl;
        if (localConfigurators != null && !localConfigurators.isEmpty()) {
            for (Configurator configurator : localConfigurators) {
                this.overrideDirectoryUrl = configurator.configure(overrideDirectoryUrl);
            }
        }
    refreshInvoker(invokerUrls);
```

#### 说明：
* 获取刚才更新的configurators
* 对每个Configurator，都调用 configurator.configure()方法, 合并Override参数。



### RegistryProtocol

```
RegistryProtocol 通过向注册中心注册 OverrideListener 监听器，从而集成配置规则到服务提供者中。
```


#### export方法

```
export方法 创建一个OverrideListener，
向注册中心注册，
```

```java
    @Override
    public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
        // 暴露服务
        // export invoker
        final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker);

        // 获得注册中心 URL
        URL registryUrl = getRegistryUrl(originInvoker);

        // 获得注册中心对象
        // registry provider
        final Registry registry = getRegistry(originInvoker);

        // 获得服务提供者 URL
        final URL registedProviderUrl = getRegistedProviderUrl(originInvoker);

        //to judge to delay publish whether or not
        boolean register = registedProviderUrl.getParameter("register", true);

        // 向注册中心订阅服务消费者
        ProviderConsumerRegTable.registerProvider(originInvoker, registryUrl, registedProviderUrl);

        // 向注册中心注册服务提供者（自己）
        if (register) {
            register(registryUrl, registedProviderUrl);
            ProviderConsumerRegTable.getProviderWrapper(originInvoker).setReg(true); // // 标记向本地注册表的注册服务提供者，已经注册
        }

        // 使用 OverrideListener 对象，订阅配置规则
        // Subscribe the override data
        // 创建订阅配置规则的 URL
        final URL overrideSubscribeUrl = getSubscribedOverrideUrl(registedProviderUrl);
        // 创建 OverrideListener 对象，并添加到 `overrideListeners` 中
        final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl, originInvoker);
        overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);
        // 向注册中心，发起订阅
        registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);
        //Ensure that a new exporter instance is returned every time export
        return new DestroyableExporter<T>(exporter, originInvoker, overrideSubscribeUrl, registedProviderUrl);
    }


// 可以先省略其他内容， 主要看这几行代码

        // 使用 OverrideListener 对象，订阅配置规则
        // Subscribe the override data
        // 创建订阅配置规则的 URL
        final URL overrideSubscribeUrl = getSubscribedOverrideUrl(registedProviderUrl);
        // 创建 OverrideListener 对象，并添加到 `overrideListeners` 中
        final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl, originInvoker);
        overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);
        // 向注册中心，发起订阅
        registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);
        //Ensure that a new exporter instance is returned every time export
        return new DestroyableExporter<T>(exporter, originInvoker, overrideSubscribeUrl, registedProviderUrl);

```

```java
    private URL getSubscribedOverrideUrl(URL registedProviderUrl) {
        return registedProviderUrl.setProtocol(Constants.PROVIDER_PROTOCOL)
                .addParameters(Constants.CATEGORY_KEY, Constants.CONFIGURATORS_CATEGORY, // configurators
                        Constants.CHECK_KEY, String.valueOf(false)); // 订阅失败，不校验。因为，不需要检查。
    }
```

#### 说明：
* (1) 创建基于配置规则的URL
* （2）创建一个OverrideListener， 把配置规则注册到注册中心。
* （3）通过overrideListeners管理好当前url对应的 override监听器。

### OverrideListener

```
OverrideListener 是 RegistryProtocol 内部类，实现 NotifyListener 接口，官方注释如下：
```

 * Reexport: the exporter destroy problem in protocol
 * 1.Ensure that the exporter returned by registryprotocol can be normal destroyed
 * 2.No need to re-register to the registry after notify
 * 3.The invoker passed by the export method , would better to be the invoker of exporter

 * 重新 export ：protocol 中的 exporter destroy 问题
 *
 * 1. 要求 registry protocol 返回的 exporter 可以正常 destroy
 * 2. notify 后不需要重新向注册中心注册
 * 3. export 方法传入的 invoker 最好能一直作为 exporter 的 invoker.


#### 构造方法
```java
/**
 * 订阅 URL 对象
 */
private final URL subscribeUrl;
/**
 * 原始 Invoker 对象
 */
private final Invoker originInvoker;

public OverrideListener(URL subscribeUrl, Invoker originalInvoker) {
    this.subscribeUrl = subscribeUrl;
    this.originInvoker = originalInvoker;
}
```

#### notify

```java
        @Override
        public synchronized void notify(List<URL> urls) {
            // 获得匹配的规则配置 URL 集合
            logger.debug("original override urls: " + urls);
            List<URL> matchedUrls = getMatchedUrls(urls, subscribeUrl);
            logger.debug("subscribe url: " + subscribeUrl + ", override urls: " + matchedUrls);
            // No matching results
            if (matchedUrls.isEmpty()) {
                return;
            }
            // 将配置规则 URL 集合，**转换**成对应的 Configurator 集合
            List<Configurator> configurators = RegistryDirectory.toConfigurators(matchedUrls);

            // 获得真实的 Invoker 对象
            final Invoker<?> invoker;
            if (originInvoker instanceof InvokerDelegete) {
                invoker = ((InvokerDelegete<?>) originInvoker).getInvoker();
            } else {
                invoker = originInvoker;
            }
            // The origin invoker
            // 获得真实的 Invoker 的 URL 对象
            URL originUrl = RegistryProtocol.this.getProviderUrl(invoker);

            // 忽略，若对应的 Exporter 对象不存在
            String key = getCacheKey(originInvoker);
            ExporterChangeableWrapper<?> exporter = bounds.get(key);
            if (exporter == null) {
                logger.warn(new IllegalStateException("error state, exporter should not be null"));
                return;
            }

            // The current, may have been merged many times
            // 获得 Invoker 当前的 URL 对象，可能已经被之前的配置规则合并过
            URL currentUrl = exporter.getInvoker().getUrl();
            // Merged with this configuration
            // 基于 originUrl 对象，合并配置规则，生成新的 newUrl 对象
            URL newUrl = getConfigedInvokerUrl(configurators, originUrl);
            // 判断新老 Url 不匹配，重新暴露 Invoker
            if (!currentUrl.equals(newUrl)) {
                RegistryProtocol.this.doChangeLocalExport(originInvoker, newUrl);
                logger.info("exported provider url changed, origin url: " + originUrl + ", old export url: " + currentUrl + ", new export url: " + newUrl);
            }
        }
```

```java
        private List<URL> getMatchedUrls(List<URL> configuratorUrls, URL currentSubscribe) {
            List<URL> result = new ArrayList<URL>();
            for (URL url : configuratorUrls) {
                URL overrideUrl = url;
                // 【忽略】，兼容老版本
                // Compatible with the old version
                if (url.getParameter(Constants.CATEGORY_KEY) == null
                        && Constants.OVERRIDE_PROTOCOL.equals(url.getProtocol())) {
                    overrideUrl = url.addParameter(Constants.CATEGORY_KEY, Constants.CONFIGURATORS_CATEGORY);
                }
                // 判断是否匹配
                // Check whether url is to be applied to the current service
                if (UrlUtils.isMatch(currentSubscribe, overrideUrl)) {
                    result.add(url);
                }
            }
            return result;
        }
```

```java
        private URL getConfigedInvokerUrl(List<Configurator> configurators, URL url) {
            for (Configurator configurator : configurators) {
                // 合并配置规则
                url = configurator.configure(url);
            }
            return url;
        }
    }
```

#### 说明：
* getMatchedUrls 判断 当前的配置URL是否和 待更新的URL匹配，若匹配加入result列表。
* 通过 RegistryDirectory.toConfigurators(matchedUrls) 获取匹配好的Configurator。
* 通过 overrideListener 获取 invoker里面真实的URL对象。
* 如果对应的exporter对象不存在的话，则忽略，否则获取exporter对象对应的invoker和currentUrl
* 通过getConfigedInvokerUrl  基于originURL 合并最新的配置规则。
* 如果新老 Url 不匹配，重新暴露 Invoker


#### 解释
* （1）当调用export方法的时候， 会通过originInvoker初始化对应的configuratorUrl。
* （2）configuratorUrl 会初始化对应的 监听器，并注册好。

前两步相当于把 Override 对应的配置更新逻辑注册好，在服务即将暴露的时候。

* （3）当配置变更的时候，找到匹配的ConfiguratorUrl列表，然后根据Configurator列表，更新当前的URL
* 如果判断新老URL不相同，说明invoker参数已经变更了，需要重新发布。
* 调用doChangeLocalExport 更新invoker，也完成了invoker的配置变更。

