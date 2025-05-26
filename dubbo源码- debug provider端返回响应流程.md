# dubbo源码- debug provider端返回响应流程

```版本来自dubbo 2.6.x```


## DemoServiceImpl

![](media/17482605304551/17482624324383.jpg)


## JavaAssistFactory
向后传递

![](media/17482605304551/17482624511329.jpg)



## DelegateProviderMetaDataInvoker 
没做什么操作，继续向后传递。

![](media/17482605304551/17482624687018.jpg)



## 后面回到encode方法

先走Netty Pipeline 流程进行编码和序列化，发送到网卡。
后面 相当于调用sent方法做一些数据记录。


## NettyChannel 
先调用Netty 发送数据。

![](media/17482605304551/17482624859876.jpg)


## InternalEncoder
进行编码和序列化

![](media/17482605304551/17482625086349.jpg)


## DubboCountCodec
调用 DubboCodec

![](media/17482605304551/17482625235699.jpg)



## ExchangeCodec

encodeResponse

![](media/17482605304551/17482625450559.jpg)

## 先按照Dubbo协议encode
最后encode 响应。

![](media/17482605304551/17482625638482.jpg)



## 调用netty流程发送出去。

![](media/17482605304551/17482625812411.jpg)

## 最后走到DubboHandler的业务流程。进行数据记录。

![](media/17482605304551/17482625959667.jpg)


## 注意这个里面的handler，会按照这个顺序，一层层向后面传递

![](media/17482605304551/17482626116106.jpg)




## HeartBeatHandler
设置写时间戳。继续向后传递。

![](media/17482605304551/17482626269974.jpg)




## HeaderExchangeHandler
在设置写时间戳，继续向后面传递

![](media/17482605304551/17482626414560.jpg)


