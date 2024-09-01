# dubbo源码-集群cluster-router-ScriptRouter


### ScriptRouterFactory

```java
public class ScriptRouterFactory implements RouterFactory {

    public static final String NAME = "script";

    @Override
    public Router getRouter(URL url) {
        return new ScriptRouter(url);
    }
}
```

#### 说明： 代码比较简单，只是初始化了ScriptRouter对象。

### ScriptRouter

```脚本路由规则 通过 type=javascript 参数设置脚本类型，缺省为 javascript。```

#### 一个脚本例子：
```javascript
// 如果当前ip满足条件的话，返回列表，
function route(invokers) {
    var result = new java.util.ArrayList(invokers.size());
     for (i = 0; i < invokers.size(); i ++) {
         if ("192.168.1.1".equals(invokers.get(i).getUrl().getHost())) {
             result.add(invokers.get(i));
         }
     }
     return result;
 } (invokers);
```


#### 成员变量与构造方法：
```java
private static final Map<String, ScriptEngine> engines = new ConcurrentHashMap<String, ScriptEngine>();

/**
 * 路由规则 URL
 */
private final ScriptEngine engine;
/**
 * 路由规则的优先级，用于排序，优先级越大越靠前执行，可不填，缺省为 0 。
 */
private final int priority;
/**
 * 路由规则内容
 */
private final String rule;
/**
 * 路由规则 URL
 */
private final URL url;



public ScriptRouter(URL url) {
    this.url = url;
    String type = url.getParameter(Constants.TYPE_KEY);
    this.priority = url.getParameter(Constants.PRIORITY_KEY, 0);
    String rule = url.getParameterAndDecoded(Constants.RULE_KEY);
    // 初始化 `engine`
    if (type == null || type.length() == 0) {
        type = Constants.DEFAULT_SCRIPT_TYPE_KEY;
    }
    if (rule == null || rule.length() == 0) {
        throw new IllegalStateException(new IllegalStateException("route rule can not be empty. rule:" + rule));
    }
    ScriptEngine engine = engines.get(type);
    if (engine == null) { // 在缓存中不存在，则进行创建 ScriptEngine 对象
        engine = new ScriptEngineManager().getEngineByName(type);
        if (engine == null) {
            throw new IllegalStateException(new IllegalStateException("Unsupported route rule type: " + type + ", rule: " + rule));
        }
        engines.put(type, engine);
    }
    this.engine = engine;
    this.rule = rule;
}
```

#### 真正的route方法：
```java
    /**
     * (1) 其实就是把 invokersList， invocation参数，context传到bindings里面去。
     * （2）通过engine编译
     * （3）function.eval(bindings) 返回脚本执行之后的结果
     * @param invokers   Invoker 集合 
     * @param url        refer url
     * @param invocation
     * @param <T>
     * @return
     * @throws RpcException
     */
    public <T> List<Invoker<T>> route(List<Invoker<T>> invokers, URL url, Invocation invocation) throws RpcException {
        try {
            // 执行脚本
            List<Invoker<T>> invokersCopy = new ArrayList<Invoker<T>>(invokers);
            Compilable compilable = (Compilable) engine;
            Bindings bindings = engine.createBindings();
            bindings.put("invokers", invokersCopy);
            bindings.put("invocation", invocation);
            bindings.put("context", RpcContext.getContext());
            CompiledScript function = compilable.compile(rule); // 编译
            Object obj = function.eval(bindings); // 执行
            // 根据结果类型，转换成 (List<Invoker<T>> 类型返回
            if (obj instanceof Invoker[]) {
                invokersCopy = Arrays.asList((Invoker<T>[]) obj);
            } else if (obj instanceof Object[]) {
                invokersCopy = new ArrayList<Invoker<T>>();
                for (Object inv : (Object[]) obj) {
                    invokersCopy.add((Invoker<T>) inv);
                }
            } else {
                invokersCopy = (List<Invoker<T>>) obj;
            }
            return invokersCopy;
        } catch (ScriptException e) {
            // 发生异常，忽略路由规则，返回全 `invokers` 集合
            // fail then ignore rule .invokers.
            logger.error("route error , rule has been ignored. rule: " + rule + ", method:" + invocation.getMethodName() + ", url: " + RpcContext.getContext().getUrl(), e);
            return invokers;
        }
    }

```
#### 说明：
 * (1) 其实就是把 invokersList， invocation参数，context传到bindings里面去。
 * （2）通过engine编译
 * （3）function.eval(bindings) 返回脚本执行之后的结果
 * （4）脚本引擎为java自身jdk支持的脚本解析工具。javax.script.ScriptEngine


