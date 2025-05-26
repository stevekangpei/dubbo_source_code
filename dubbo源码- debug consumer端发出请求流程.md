# dubbo源码- debug consumer端发出请求流程

```版本来自dubbo 2.6.x```

## 发起调用
![](media/17482604624384/17482606704566.jpg)

## InvokerInvocationHandler

![](media/17482604624384/17482606819157.jpg)


![](media/17482604624384/17482609999244.jpg)



## MockClusterInvoker

![](media/17482604624384/17482610170622.jpg)


## AbstractClusterInvoker

![](media/17482604624384/17482611178441.jpg)



## FailoverClusterInvoker
默认的retry是2次，还加了1 所以是三次。

![](media/17482604624384/17482611563751.jpg)



## InvokerWrapper 
只做了一层转发

![](media/17482604624384/17482610597361.jpg)



## ListenerInvokerWrapper

![](media/17482604624384/17482608240682.jpg)


## ProtocolFilterWrapper
按照如下顺序向后执行

ConsumerContextFilter
FutureFilter
MonitorFilter

![](media/17482604624384/17482609523347.jpg)



## AbstractInvoker

![](media/17482604624384/17482612466169.jpg)


## DubboInvoker

![](media/17482604624384/17482612653660.jpg)

## ReferenceCountExchangeClient

![](media/17482604624384/17482612840260.jpg)


## HeaderExchangeClient

![](media/17482604624384/17482613037449.jpg)



## AbstractClient

获取了Dubbo自定义的Channel，里面包装了真正的NettyChannel

![](media/17482604624384/17482613240610.jpg)



## NettyChannel

调用了真正的Netty的channel进行发送。
注意接下来这个地方要把断点停到InternalEncoder
因为接下来会走NettyPipeline进行编码。

![](media/17482604624384/17482613514949.jpg)

## InternalEncoder

![](media/17482604624384/17482613699650.jpg)


## ExchangeCodec

encodeRequest

![](media/17482604624384/17482613887311.jpg)


## DubboCodec 
编码和序列化 Invocation

这个地方真正将请求体的数据写进入

![](media/17482604624384/17482614220829.jpg)


## 再回到ExchangeCodec

在请求头里面写入长度，我们的长度是185
数据后续会走NettyPipeline 发生到Provider。

![](media/17482604624384/17482614469790.jpg)


