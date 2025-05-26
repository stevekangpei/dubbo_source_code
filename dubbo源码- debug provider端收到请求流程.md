# dubbo源码- debug provider端收到请求流程

```版本来自dubbo 2.6.x```


## InternalDecoder
首先走Netty Pipeline进行解码 

![](media/17482605114385/17482620805731.jpg)


## DubboCountCodec

进一步调用DubboCodec进行解码

![](media/17482605114385/17482621023702.jpg)


## ExchangeCodec 解码请求

![](media/17482605114385/17482621182986.jpg)



## DecodeableRpcInvocation
传递给 DecodeableRpcInvocation 解码和反序列化Request。

![](media/17482605114385/17482621447374.jpg)



## 接下来走Dubbo流程。

会到NettyHandler， Netty解码和反序列化数据完了之后，会开始通过Dubbo各个handler进行传递。
一直到业务实现

![](media/17482605114385/17482621722587.jpg)


## MultiMessageHandler

![](media/17482605114385/17482621954204.jpg)



## HeartBeatHandler
设置读时间戳。
然后传递给AllChannelHandler 

![](media/17482605114385/17482622107682.jpg)

## AllChannelHandler 提交给线程池执行。

![](media/17482605114385/17482622259442.jpg)



## 提交给Decodehandler
这个地方已经解码过了，所以不再继续反序列

![](media/17482605114385/17482622442820.jpg)


## HeaderExchangehandler

再一次设置写时间戳。


![](media/17482605114385/17482622599219.jpg)


## 将数据转为request。提交给后续的handler处理。

![](media/17482605114385/17482622764706.jpg)


## DubboProtocol 里面的 ExchangeHandler

获取invoker 执行。
![](media/17482605114385/17482623045693.jpg)




## 注意这里的Invoker。
先走FIlterChain， EchoFilter，ClassloaderFilter，GenericFilter，ContextFilter，
TraceFilter，TimeOutFilter，MonitorFilter，ExceptionFilter，
后面走到了DelegateProviderMetaDataInvoker。

![](media/17482605114385/17482623278358.jpg)


## DelegateProviderMetaDataInvoker 
没做什么操作，继续向后传递。

![](media/17482605114385/17482623447032.jpg)



## 走到了JavaAssistFactory


调用Wrapper类 进行invoke
这个Wrapper类是一个动态编译生成的类。

![](media/17482605114385/17482623618983.jpg)



## 真实的实现 

DemoServiceImpl

![](media/17482605114385/17482623779481.jpg)


