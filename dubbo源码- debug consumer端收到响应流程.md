# dubbo源码- debug consumer端收到响应流程

```版本来自dubbo 2.6.x```



## InternalDecoder

首先进行解码

![](media/17482604981215/17482616120610.jpg)


##  DubboCountCodec

![](media/17482604981215/17482616299211.jpg)



## ExchangeCodec

解码响应体
![](media/17482604981215/17482616465518.jpg)


## 回到DubboCodec

![](media/17482604981215/17482616654364.jpg)


##  DecodeableRpcResult

解码 RPC响应
Flag = 1 表示响应是Ok的。
![](media/17482604981215/17482616829978.jpg)


## 结果已经放到了RpcResult里面了

![](media/17482604981215/17482617003074.jpg)


##  NettyHandler
走Netty Pipeline 跳转到messageReceived
表示已经收集到了结果，接下来相当于onReceived执行。

![](media/17482604981215/17482617155890.jpg)


## 注意接下来会按照这个红框的顺序一层层传递。

![](media/17482604981215/17482617354149.jpg)

## AbstractPeer

没有做什么操作，通过里面的MultiMessageHandler 继续向后面传递

![](media/17482604981215/17482617526770.jpg)


## MultiMessageHandler
 如果是MultiMessage，会一个个向后received。当前不是，所以直接跳到else
 
 ![](media/17482604981215/17482617692758.jpg)


## HeartBeatHandler
设置读时间戳，如果是心跳的话，进行处理，否则继续传递

![](media/17482604981215/17482618120200.jpg)


## AllChannelHandler
封装为一个ChannelEventRunnable 提交给线程池。

![](media/17482604981215/17482618333809.jpg)


## DecodeHandler
提交给DecodeHandler 进一步解码
由于上一步已经解码了，所以这里没什么操作。

![](media/17482604981215/17482618514452.jpg)


## HeaderExhchangeHandler。
写入读时间戳。

![](media/17482604981215/17482618683239.jpg)


## handleResponse

![](media/17482604981215/17482618846977.jpg)




## DefaultFuture 

doReceived

设置返回值，
通知线程
触发回调。

![](media/17482604981215/17482618995195.jpg)


