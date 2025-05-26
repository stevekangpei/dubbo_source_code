Dubbo源码详细解读

```
本项目基于dubbo 2.6.1版本，将dubbo的代码比较详细的做了分析。用于学习使用。
包括暴露流程，引用流程，请求与响应流程。还包括dubbo各个 module的详细解析。
可以先从总流程看起来。只要总流程理解了，dubbo子module自然也就理解了
```


## Debug Dubbo 截图版

### provider端

[provider端收到请求](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%2520debug%2520provider%25E7%25AB%25AF%25E6%2594%25B6%25E5%2588%25B0%25E8%25AF%25B7%25E6%25B1%2582%25E6%25B5%2581%25E7%25A8%258B.md)

[provider端返回响应](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%2520debug%2520provider%25E7%25AB%25AF%25E8%25BF%2594%25E5%259B%259E%25E5%2593%258D%25E5%25BA%2594%25E6%25B5%2581%25E7%25A8%258B.md)

### consumer端
[consumer端发出请求](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%2520debug%2520consumer%25E7%25AB%25AF%25E5%258F%2591%25E5%2587%25BA%25E8%25AF%25B7%25E6%25B1%2582%25E6%25B5%2581%25E7%25A8%258B.md)

[consumer端收到响应](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581-%2520debug%2520consumer%25E7%25AB%25AF%25E6%2594%25B6%25E5%2588%25B0%25E5%2593%258D%25E5%25BA%2594%25E6%25B5%2581%25E7%25A8%258B.md)


## Dubbo consumer端引用流程

[consumer端本地引用](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581%25E4%25B9%258B%2520consumer%25E7%25AB%25AF%2520%25E6%259C%25AC%25E5%259C%25B0%25E5%25BC%2595%25E7%2594%25A8%25EF%25BC%2588injvm%25EF%25BC%2589%25E5%2588%2586%25E6%259E%2590.md)


[consumer端远程引用（一）](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581%25E4%25B9%258B-dubbo%2520consumer%25E7%25AB%25AF%25E8%25BF%259C%25E7%25A8%258B%25E5%25BC%2595%25E7%2594%25A8%25E5%2588%2586%25E6%259E%2590%2520%25EF%25BC%2588%25E4%25B8%2580%25EF%25BC%2589.md)

[consumer端远程引用（二）](https://github.com/stevekangpei/dubbo_source_code/blob/master/dubbo%25E6%25BA%2590%25E7%25A0%2581%25E4%25B9%258B-dubbo%2520consumer%25E7%25AB%25AF%25E8%25BF%259C%25E7%25A8%258B%25E5%25BC%2595%25E7%2594%25A8%25E5%2588%2586%25E6%259E%2590%2520%25EF%25BC%2588%25E4%25BA%258C%25EF%25BC%2589.md)


## Dubbo provider端暴露流程

