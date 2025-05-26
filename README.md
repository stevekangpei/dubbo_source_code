Dubbo源码详细解读

```
本项目基于dubbo 2.6.1版本，将dubbo的代码比较详细的做了分析。用于学习使用。
包括暴露流程，引用流程，请求与响应流程。还包括dubbo各个 module的详细解析。
可以先从总流程看起来。只要总流程理解了，dubbo子module自然也就理解了
```


## 打断点 Debug 请求与响应截图版

### provider端（断点截图）

[provider端收到请求](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%2520debug%2520provider%25E7%25AB%25AF%25E6%2594%25B6%25E5%2588%25B0%25E8%25AF%25B7%25E6%25B1%2582%25E6%25B5%2581%25E7%25A8%258B.md)

[provider端返回响应](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%2520debug%2520provider%25E7%25AB%25AF%25E8%25BF%2594%25E5%259B%259E%25E5%2593%258D%25E5%25BA%2594%25E6%25B5%2581%25E7%25A8%258B.md)

### consumer端 （断点截图）
[consumer端发出请求](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%2520debug%2520consumer%25E7%25AB%25AF%25E5%258F%2591%25E5%2587%25BA%25E8%25AF%25B7%25E6%25B1%2582%25E6%25B5%2581%25E7%25A8%258B.md)

[consumer端收到响应](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%2520debug%2520consumer%25E7%25AB%25AF%25E6%2594%25B6%25E5%2588%25B0%25E5%2593%258D%25E5%25BA%2594%25E6%25B5%2581%25E7%25A8%258B.md)


## Dubbo consumer端引用流程（无断点）

[consumer端本地引用](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581%25E4%25B9%258B%2520consumer%25E7%25AB%25AF%2520%25E6%259C%25AC%25E5%259C%25B0%25E5%25BC%2595%25E7%2594%25A8%25EF%25BC%2588injvm%25EF%25BC%2589%25E5%2588%2586%25E6%259E%2590.md)


[consumer端远程引用（一）](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581%25E4%25B9%258B-dubbo%2520consumer%25E7%25AB%25AF%25E8%25BF%259C%25E7%25A8%258B%25E5%25BC%2595%25E7%2594%25A8%25E5%2588%2586%25E6%259E%2590%2520%25EF%25BC%2588%25E4%25B8%2580%25EF%25BC%2589.md)

[consumer端远程引用（二）](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581%25E4%25B9%258B-dubbo%2520consumer%25E7%25AB%25AF%25E8%25BF%259C%25E7%25A8%258B%25E5%25BC%2595%25E7%2594%25A8%25E5%2588%2586%25E6%259E%2590%2520%25EF%25BC%2588%25E4%25BA%258C%25EF%25BC%2589.md)


## Dubbo provider端暴露流程 （无断点）

[provider端本地暴露流程](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581%25E4%25B9%258B-dubbo%2520provider%25E7%25AB%25AF%25E6%259C%25AC%25E5%259C%25B0%25E6%259A%25B4%25E9%259C%25B2(injvm)%25E6%25B5%2581%25E7%25A8%258B.md)

[provider端远程暴露流程（一）](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581%25E4%25B9%258B-dubbo%2520provider%25E7%25AB%25AF%25E8%25BF%259C%25E7%25A8%258B%25E6%259A%25B4%25E9%259C%25B2%25E6%25B5%2581%25E7%25A8%258B%25EF%25BC%2588%25E4%25B8%2580%25EF%25BC%2589.md)

[provider端远程暴露流程（二）](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581%25E4%25B9%258B-dubbo%2520provider%25E7%25AB%25AF%25E8%25BF%259C%25E7%25A8%258B%25E6%259A%25B4%25E9%259C%25B2%25E6%25B5%2581%25E7%25A8%258B%25EF%25BC%2588%25E4%25BA%258C%25EF%25BC%2589.md)

[provider端远程暴露流程（三）](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581%25E4%25B9%258B-dubbo%2520provider%25E7%25AB%25AF%25E8%25BF%259C%25E7%25A8%258B%25E6%259A%25B4%25E9%259C%25B2%25E6%25B5%2581%25E7%25A8%258B%25EF%25BC%2588%25E4%25B8%2589%25EF%25BC%2589.md)


## consumer发起调用 和收到响应流程（无断点）

[consumer端发起调用流程](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581%25E4%25B9%258B-dubbo%2520consumer%2520%25E5%258F%2591%25E8%25B5%25B7%25E8%25B0%2583%25E7%2594%25A8.md)

[consumer端收到响应](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581%25E4%25B9%258B-dubbo%2520consumer%2520%25E6%2594%25B6%25E5%2588%25B0%25E5%2593%258D%25E5%25BA%2594.md)

## provider端收到请求和返回响应流程（无断点）

[provider端收到请求流程](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581%25E4%25B9%258B-dubbo%2520provider%2520%25E6%2594%25B6%25E5%2588%25B0%25E8%25AF%25B7%25E6%25B1%2582.md)

[Provider端返回响应流程](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581%25E4%25B9%258B-dubbo%2520provider%2520%25E8%25BF%2594%25E5%259B%259E%25E5%2593%258D%25E5%25BA%2594.md)



## dubbo 序列化

[Dubbo序列化接口层实现](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%25E5%25BA%258F%25E5%2588%2597%25E5%258C%2596-%25E6%258E%25A5%25E5%258F%25A3%25E5%25B1%2582%25E5%25AE%259E%25E7%258E%25B0.md)

[Dubbo序列化Kryo实现](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%25E5%25BA%258F%25E5%2588%2597%25E5%258C%2596-Kryo%25E5%25AE%259E%25E7%258E%25B0.md)

[Dubbo序列化 FST实现](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%25E5%25BA%258F%25E5%2588%2597%25E5%258C%2596-FST%25E5%25AE%259E%25E7%258E%25B0.md)

## Dubbo注册中心

[dubbo 注册中心 接口层实现](https://github.com/stevekangpei/dubbo_source_code/blob/master/%2520dubbo%25E6%25BA%2590%25E7%25A0%2581-%25E6%25B3%25A8%25E5%2586%258C%25E4%25B8%25AD%25E5%25BF%2583-%25E4%25B9%258B%25E6%258E%25A5%25E5%258F%25A3%25E5%25B1%2582.md)

[Dubbo注册中心 抽象层实现](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%25E6%25B3%25A8%25E5%2586%258C%25E4%25B8%25AD%25E5%25BF%2583-%25E6%258A%25BD%25E8%25B1%25A1%25E5%25B1%2582%25E5%25AE%259E%25E7%258E%25B0%25E3%2580%2582.md)

[Dubbo 注册中心 Zookeeper注册中心](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%25E6%25B3%25A8%25E5%2586%258C%25E4%25B8%25AD%25E5%25BF%2583-%25E4%25B9%258BZookeeper%25E6%25B3%25A8%25E5%2586%258C%25E4%25B8%25AD%25E5%25BF%2583.md)

[Dubbo注册中心 Redis注册中心](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%25E6%25B3%25A8%25E5%2586%258C%25E4%25B8%25AD%25E5%25BF%2583-%25E4%25B9%258BRedis%25E6%25B3%25A8%25E5%2586%258C%25E4%25B8%25AD%25E5%25BF%2583.md)

[Dubbo注册中心 Zk工具类](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%25E6%25B3%25A8%25E5%2586%258C%25E4%25B8%25AD%25E5%25BF%2583%25E4%25B9%258BZookeeper%25E5%25B7%25A5%25E5%2585%25B7%25E7%25B1%25BB.md)


## Dubbo SPI与扩展点

[Dubbo SPI机制](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-dubbo%2520SPI%25E6%259C%25BA%25E5%2588%25B6.md)

[Dubbo扩展点整理](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-dubbo%25E6%2589%25A9%25E5%25B1%2595%25E7%2582%25B9%25E6%2595%25B4%25E7%2590%2586.md)


## Dubbo Filter层

[Dubbo Filter层整理](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-filter%25E5%25B1%2582.md)


## Dubbo cluster层

[Dubbo-Cluter 之Directory抽象](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%25E9%259B%2586%25E7%25BE%25A4cluster-%25E4%25B9%258Bdirectory%25E6%258A%25BD%25E8%25B1%25A1%25E3%2580%2582.md)

[Dubbo-Cluster 之 StaticDirectory](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%25E9%259B%2586%25E7%25BE%25A4cluster-%25E4%25B9%258BStaticDirectory.md)

[Dubbo-Cluster之 RegistryDirectory](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%25E9%259B%2586%25E7%25BE%25A4cluster-%25E4%25B9%258BRegistryDirectory.md)



[Dubbo-Cluster 之Configurator抽象层](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%25E9%259B%2586%25E7%25BE%25A4cluster-%25E4%25B9%258Bconfigurator%25E6%258A%25BD%25E8%25B1%25A1%25E5%25B1%2582.md)

[Dubbo-cluster 之 configurator 和其它层的抽象](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%25E9%259B%2586%25E7%25BE%25A4cluster-%25E4%25B9%258BConfigurator%2520%25E5%2592%258C%25E5%2585%25B6%25E5%25AE%2583%25E5%25B1%2582%25E7%259A%2584%25E7%25BB%2584%25E5%2590%2588.md)

[Dubbo-Cluster 之 Configurator实现](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%25E9%259B%2586%25E7%25BE%25A4cluster-%25E4%25B9%258BConfigurator%25E5%25AE%259E%25E7%258E%25B0%25E5%25B1%2582.md)


[Dubbo-cluster之Router实现](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%25E9%259B%2586%25E7%25BE%25A4cluster-%25E4%25B9%258BRouter.md)

[Dubbo-cluster 之ScriptRouter实现](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%25E9%259B%2586%25E7%25BE%25A4cluster-router-ScriptRouter.md)


[Dubbo-cluster-之 ConditionRouter实现](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%25E9%259B%2586%25E7%25BE%25A4cluster-router-%25E4%25B9%258BConditionRouter.md)


[Dubbo-cluster 之merger实现](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%25E9%259B%2586%25E7%25BE%25A4cluster-%25E4%25B9%258Bmerger%25E5%25AE%259E%25E7%258E%25B0.md)


## Dubbo-cluster 集群容错

[Dubbo集群容错之容错全流程](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%25E9%259B%2586%25E7%25BE%25A4%25E5%25AE%25B9%25E9%2594%2599-%25E4%25B9%258B%25E5%25AE%25B9%25E9%2594%2599%25E5%2585%25A8%25E6%25B5%2581%25E7%25A8%258B.md)

[Dubbo集群负载均衡之最小连接数负载均衡](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%25E9%259B%2586%25E7%25BE%25A4%25E5%25AE%25B9%25E9%2594%2599-%25E8%25B4%259F%25E8%25BD%25BD%25E5%259D%2587%25E8%25A1%25A1%25E4%25B9%258B-LeastActiveLoadBalance%2520%25E6%259C%2580%25E5%25B0%258F%25E8%25BF%259E%25E6%258E%25A5%25E6%2595%25B0%25E8%25B4%259F%25E8%25BD%25BD%25E5%259D%2587%25E8%25A1%25A1.md)

[Dubbo集群负载均衡之-随机负载均衡](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%25E9%259B%2586%25E7%25BE%25A4%25E5%25AE%25B9%25E9%2594%2599-%25E8%25B4%259F%25E8%25BD%25BD%25E5%259D%2587%25E8%25A1%25A1%25E4%25B9%258B-RandomLoadBalance%25E5%25AE%259E%25E7%258E%25B0.md)

[Dubbo集群负载均衡一致性哈希负载均衡](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%25E9%259B%2586%25E7%25BE%25A4%25E5%25AE%25B9%25E9%2594%2599-%25E8%25B4%259F%25E8%25BD%25BD%25E5%259D%2587%25E8%25A1%25A1%25E4%25B9%258B-%25E4%25B8%2580%25E8%2587%25B4%25E6%2580%25A7%25E5%2593%2588%25E5%25B8%258C%2520ConsistentHashLoadBalance.md)

[Dubbo集群负载均衡之--RoungRobin负载均衡](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%25E9%259B%2586%25E7%25BE%25A4%25E5%25AE%25B9%25E9%2594%2599-%25E8%25B4%259F%25E8%25BD%25BD%25E5%259D%2587%25E8%25A1%25A1%25E4%25B9%258BRoundRobin%25E5%25AE%259E%25E7%258E%25B0.md)

[Dubbo集群容错之BroadCastCluster](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%25E9%259B%2586%25E7%25BE%25A4%25E5%25AE%25B9%25E9%2594%2599%25E4%25B9%258BBroadcastCluster.md)

[Dubbo集群容错之-FailbackCluster](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%25E9%259B%2586%25E7%25BE%25A4%25E5%25AE%25B9%25E9%2594%2599%25E4%25B9%258BFailbackCluster.md)

[Dubbo集群容错之-FailFastCluster](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%25E9%259B%2586%25E7%25BE%25A4%25E5%25AE%25B9%25E9%2594%2599%25E4%25B9%258BFailfastCluster.md)

[Dubbo集群容错之FailOverCluster](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%25E9%259B%2586%25E7%25BE%25A4%25E5%25AE%25B9%25E9%2594%2599%25E4%25B9%258BFailoverCluster.md)

[Dubbo集群容错之FailSafeCluster](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%25E9%259B%2586%25E7%25BE%25A4%25E5%25AE%25B9%25E9%2594%2599%25E4%25B9%258BFailsafeCluster.md)

[Dubbo集群容错之MergableCluster
](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%25E9%259B%2586%25E7%25BE%25A4%25E5%25AE%25B9%25E9%2594%2599%25E4%25B9%258BMergeableCluster.md)

[Dubbo集群容错之-ForkingCluster](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%25E9%259B%2586%25E7%25BE%25A4%25E5%25AE%25B9%25E9%2594%2599%25E4%25B9%258BForkingCluster.md)




## Dubbo-网络服务器层

[Dubbo 服务器 之 Transporter抽象](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%25E7%25BD%2591%25E7%25BB%259C%25E6%259C%258D%25E5%258A%25A1%25E5%2599%25A8-%25E4%25B9%258Btransporter%2520%25E6%258A%25BD%25E8%25B1%25A1.md)

[Dubbo服务器之 服务端Server实现](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%25E7%25BD%2591%25E7%25BB%259C%25E6%259C%258D%25E5%258A%25A1%25E5%2599%25A8-%25E4%25B9%258Btransporter%2520%25E6%259C%258D%25E5%258A%25A1%25E7%25AB%25AF%25E5%25AE%259E%25E7%258E%25B0.md)

[Dubbo服务器之客户端实现](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%25E7%25BD%2591%25E7%25BB%259C%25E6%259C%258D%25E5%258A%25A1%25E5%2599%25A8-Transporter%25E4%25B9%258B%25E5%25AE%25A2%25E6%2588%25B7%25E7%25AB%25AF%25E5%25AE%259E%25E7%258E%25B0.md)


[Dubbo服务器之Exchange层](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%25E7%25BD%2591%25E7%25BB%259C%25E6%259C%258D%25E5%258A%25A1%25E5%2599%25A8-%25E4%25B9%258BExchange.md)

[Dubbo服务器之Exchange层-Request 和Response](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%25E7%25BD%2591%25E7%25BB%259C%25E6%259C%258D%25E5%258A%25A1%25E5%2599%25A8-%25E4%25B9%258BExchange%25E5%25B1%2582-Request%25E5%2592%258CResponse.md)

[Dubbo服务器之-Exchange层-之Codec 和Dubbo协议](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%25E7%25BD%2591%25E7%25BB%259C%25E6%259C%258D%25E5%258A%25A1%25E5%2599%25A8-%25E4%25B9%258BExchange%25E5%25B1%2582-codec%25E5%2592%258Cdubbo%25E5%258D%258F%25E8%25AE%25AE.md)


[Dubbo服务器之-Channel通道](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%25E7%25BD%2591%25E7%25BB%259C%25E6%259C%258D%25E5%258A%25A1%25E5%2599%25A8-%25E4%25B9%258BChannel%25E9%2580%259A%25E9%2581%2593.md)


