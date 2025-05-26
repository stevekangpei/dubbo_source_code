# dubbo源码之-dubbo provider端远程暴露流程（二）

### ProtocolFilterWrapper
```java
@Override
    public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
        if (Constants.REGISTRY_PROTOCOL.equals(invoker.getUrl().getProtocol())) {/// registry protocol  就到下一个wapper 包装对象中就可以了
            return protocol.export(invoker);
        }
        return protocol.export(buildInvokerChain(invoker, Constants.SERVICE_FILTER_KEY, Constants.PROVIDER));
    }
```

#### 说明：
* 1 这里不是registry protocol 然后会走下面，我们看下buildInvokerChain 方法
* 2 通过套娃的方式，一层层套进去了Invoker。本质就是装饰者模式。

```java
 private static <T> Invoker<T> buildInvokerChain(final Invoker<T> invoker, String key, String group) {
        Invoker<T> last = invoker;  
        List<Filter> filters = ExtensionLoader.getExtensionLoader(Filter.class).getActivateExtension(invoker.getUrl(), key, group);
        if (!filters.isEmpty()) {
            for (int i = filters.size() - 1; i >= 0; i--) {
                final Filter filter = filters.get(i);
                final Invoker<T> next = last;
                last = new Invoker<T>() {
                    ...
                    @Override
                    public Result invoke(Invocation invocation) throws RpcException {
                        return filter.invoke(next, invocation);
                    }
                   ...
                };
            }
        }
        return last;
    }
```

#### 说明：
* 根据dubbo spi 扩展技术自动激活的特性获取到对应的filter们，然后一层一层的包装这个invoker，生成一个过滤调用链，最后到真实的invoker上面。

### QosProtocolWrapper

```java
 public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
        //判断是registry
        if (Constants.REGISTRY_PROTOCOL.equals(invoker.getUrl().getProtocol())) {
            // 启动Qos服务器？
            startQosServer(invoker.getUrl());
            return protocol.export(invoker);
        }
        return protocol.export(invoker);
    }
```

### DubboProtocol

```java
  @Override
    public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
        URL url = invoker.getUrl();

        // export service.  key = com.alibaba.dubbo.demo.DemoService:20880
        String key = serviceKey(url);
        DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
        exporterMap.put(key, exporter);

        //export an stub service for dispatching event
        Boolean isStubSupportEvent = url.getParameter(Constants.STUB_EVENT_KEY, Constants.DEFAULT_STUB_EVENT);
        Boolean isCallbackservice = url.getParameter(Constants.IS_CALLBACK_SERVICE, false);
        if (isStubSupportEvent && !isCallbackservice) {
            String stubServiceMethods = url.getParameter(Constants.STUB_EVENT_METHODS_KEY);
            if (stubServiceMethods == null || stubServiceMethods.length() == 0) {
                if (logger.isWarnEnabled()) {
                    logger.warn(new IllegalStateException("consumer [" + url.getParameter(Constants.INTERFACE_KEY) +
                            "], has set stubproxy support event ,but no stub methods founded."));
                }
            } else {
                stubServiceMethodsMap.put(url.getServiceKey(), stubServiceMethods);
            }
        }

        openServer(url);
        optimizeSerialization(url);
        return exporter;
    }
```

#### 说明：
* 1 这里首先是根据url生成一个服务的key，创建一个DubboExporter 把invoker，key与缓存exporter的map 绑在一起。将 创建的这个exporter缓存到exporterMap里面。
* 2 接着就是调用openServer(url)，来打开服务器

#### openServer

```java
    private void openServer(URL url) {
        // find server.    获得地址 ip：port   192.168.1.104:20880
        String key = url.getAddress();
        //client can export a service which's only for server to invoke  客户端可以暴露仅供服务器调用的服务
        boolean isServer = url.getParameter(Constants.IS_SERVER_KEY, true);
        if (isServer) {// 判断是否是服务器

            //查找缓存的服务器
            ExchangeServer server = serverMap.get(key);
            if (server == null) {// 没有找到server 就要创建server
                serverMap.put(key, createServer(url));
            } else {
                // server supports reset, use together with override  server 支持重置，与覆盖一起使用
                server.reset(url);   //
            }
        }
    }
```

#### createServer:

```java
    private ExchangeServer createServer(URL url) {
        // send readonly event when server closes, it's enabled by default  服务器关闭时发送只读事件，默认情况下启用
        url = url.addParameterIfAbsent(Constants.CHANNEL_READONLYEVENT_SENT_KEY, Boolean.TRUE.toString());
        // enable heartbeat by default    60 * 1000   设置心跳
        url = url.addParameterIfAbsent(Constants.HEARTBEAT_KEY, String.valueOf(Constants.DEFAULT_HEARTBEAT));
        //获取配置的服务器类型， 缺省就是使用netty 默认服务器是netty
        String str = url.getParameter(Constants.SERVER_KEY, Constants.DEFAULT_REMOTING_SERVER);  // 获取的使用的server  缺省使用netty
        // 不存在Transports 话  ， 就抛出 不支持的server type
        if (str != null && str.length() > 0 && !ExtensionLoader.getExtensionLoader(Transporter.class).hasExtension(str))
            throw new RpcException("Unsupported server type: " + str + ", url: " + url);
        // 设置  Codec 的类型为dubbo
        url = url.addParameter(Constants.CODEC_KEY, DubboCodec.NAME);
        ExchangeServer server;
        try {
            server = Exchangers.bind(url, requestHandler);
        } catch (RemotingException e) {
            throw new RpcException("Fail to start server(url: " + url + ") " + e.getMessage(), e);
        }
        str = url.getParameter(Constants.CLIENT_KEY);// client
        if (str != null && str.length() > 0) {
            Set<String> supportedTypes = ExtensionLoader.getExtensionLoader(Transporter.class).getSupportedExtensions();
            if (!supportedTypes.contains(str)) {
                throw new RpcException("Unsupported client type: " + str);
            }
        }
        return server;
    }
```

#### 说明：
* 1 这个方法前面部分就是设置一些参数，channel.readonly.sent =true，就是服务器关闭的时候指发送只读属性，heartbeat=60*1000 设置默认的心跳时间，获取server，缺省的情况下使用netty，设置Codec 的类型为dubbo。
* 2 然后Exchangers.bind(url, requestHandler)，其实这个Exchangers 是个门面类，封装了bind与connect两个方法的调用。


#### bind方法：

```java
    public static ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
         //验证
        if (url == null) {
            throw new IllegalArgumentException("url == null");
        }
        if (handler == null) {
            throw new IllegalArgumentException("handler == null");
        }
        url = url.addParameterIfAbsent(Constants.CODEC_KEY, "exchange"); // 如果没有codec  的话 就设置为exchange
        // exchanger.bind()
        return getExchanger(url).bind(url, handler);
    }
```
#### getExchanger
```java
 public static Exchanger getExchanger(URL url) {
        // 获取exchanger  缺省header
        String type = url.getParameter(Constants.EXCHANGER_KEY, Constants.DEFAULT_EXCHANGER);
        return getExchanger(type);
    }
public static Exchanger getExchanger(String type) {
        return ExtensionLoader.getExtensionLoader(Exchanger.class).getExtension(type);
}
```

#### 说明： 这里可以看到从url中获取exchanger ，缺省是header，然后使用dubbo spi 获取到HeaderExchanger。

### HeaderExchanger

```java
public class HeaderExchanger implements Exchanger {
    public static final String NAME = "header";
    @Override
    public ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException {

        // 创建一个通信client
        //DecodeHandler => HeaderExchangeHandler => ExchangeHandler( handler ) 。
        return new HeaderExchangeClient(Transporters.connect(url, new DecodeHandler(new HeaderExchangeHandler(handler))), true);
    }
    @Override
    public ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {

        // 创建一个通信server  DecodeHandler  << HeaderExchangeHandler  << handler
        return new HeaderExchangeServer(Transporters.bind(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
    }
}
```

#### 说明：
* 1  将这handler又包装了两层，DecodeHandler  << HeaderExchangeHandler  << handler
* 2 使用Transporters 这个门面类进行bind，
* 3 创建HeaderExchangeServer 将server进行增强，其实HeaderExchangeServer 这个是专门发送心跳的。

### Transporters

```java
 public static Server bind(URL url, ChannelHandler... handlers) throws RemotingException {
        //验证
        if (url == null) {
            throw new IllegalArgumentException("url == null");
        }
        if (handlers == null || handlers.length == 0) {
            throw new IllegalArgumentException("handlers == null");
        }
        ChannelHandler handler;
        if (handlers.length == 1) {
            handler = handlers[0];
        } else {   // 多个channal  对 Channel分发   ChannelHandlerDispatcher 循环
            handler = new ChannelHandlerDispatcher(handlers);
        }
        // 真正服务器 进行bind
        return getTransporter().bind(url, handler);
    }
public static Transporter getTransporter() {  // 获取transporter
        return ExtensionLoader.getExtensionLoader(Transporter.class).getAdaptiveExtension();
}
```

#### 说明：
* 1 Transporters也是门面类，对外统一了bind 与connect。
* 2 是参数校验，接着判断handler的个数，多个话就要使用ChannelHandlerDispatcher类来包装了，其实里面就是对多个handler循环调用。
* 3 接着调用getTransporter获取Transporter扩展点的自适应类。


### NettyTransporter
```java
/**
 * 实现 Transporter 接口，基于 Netty4 的网络传输实现类
 */
public class NettyTransporter implements Transporter {
    /**
     * 扩展名
     */
    public static final String NAME = "netty";

    @Override
    public Server bind(URL url, ChannelHandler listener) throws RemotingException {
        return new NettyServer(url, listener);
    }
    @Override
    public Client connect(URL url, ChannelHandler listener) throws RemotingException {
        return new NettyClient(url, listener);
    }

}
```

#### 说明：
* bind方法创建了NettyServer对象。并启动了服务器。

