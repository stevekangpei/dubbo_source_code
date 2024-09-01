# dubbo源码-集群cluster-router-之ConditionRouter

### ConditionRouterFactory

```java
public class ConditionRouterFactory implements RouterFactory {

    public static final String NAME = "condition";

    @Override
    public Router getRouter(URL url) {
        return new ConditionRouter(url);
    }
}
```

#### 说明： 代码比较简单，只是初始化了ConditionRouter对象。

### 条件路由规则的说明：
```条件路由规则将符合特定条件的请求转发到特定的地址实例子集上。规则首先对发起流量的请求参数进行匹配，符合匹配条件的请求将被转发到包含特定实例地址列表的子集。```

#### 路由格式：
```[服务消费者匹配条件] => [服务提供者匹配条件]
host = 10.20.150.30 => host = 10.20.150.31

该条规则表示 IP 为 10.20.150.30 的服务消费者只可调用 IP 为 10.20.150.31 机器上的服务
```

### 条件路由的一些例子：
> 白名单：
> host != 10.20.30.140.50,10.20.30.140.51 =>
> 不是（10.20.30.140.50,10.20.30.140.51）的不可以访问

---

> 黑名单：
> host != 10.20.30.140.50,10.20.30.140.51 =>
> 是（10.20.30.140.50,10.20.30.140.51）的不可以访问

---

> 读写分离：
> method = fifind,list,get,is  => host = 172.22.3.94,172.22.3.95,172.22.3.96
> 是（fifind , list , get , is）的访问ip为（172.22.3.94,172.22.3.95,172.22.3.96）的服务端


### ConditionRouter

#### 成员变量
```java
    /**
     * 分组正则匹配，详细见 {@link #parseRule(String)} 方法
     *
     * 前 [] 为匹配，分隔符
     * 后 [] 为匹配，内容
     */
    private static Pattern ROUTE_PATTERN = Pattern.compile("([&!=,]*)\\s*([^&!=,\\s]+)");

    /**
     * 路由规则 URL
     */
    private final URL url;
    /**
     * 路由规则的优先级，用于排序，优先级越大越靠前执行，可不填，缺省为 0 。
     */
    private final int priority;
    /**
     * 当路由结果为空时，是否强制执行，如果不强制执行，路由结果为空的路由规则将自动失效，可不填，缺省为 false 。
     */
    private final boolean force;
    /**
     * 消费者匹配条件集合，通过解析【条件表达式 rule 的 `=>` 之前半部分】
     */
    private final Map<String, MatchPair> whenCondition;
    /**
     * 提供者地址列表的过滤条件，通过解析【条件表达式 rule 的 `=>` 之后半部分】
     */
    private final Map<String, MatchPair> thenCondition;

```
#### 说明： 
> whenCondition 消费者匹配条件集合, => 前面的条件走这个condition
> thenCondition 提供者地址列表的过滤条件 匹配后半部分。

#### 构造器

```java
    /**
     * （1）解析基本参数
     * （2）解析whenRule， 其实就是 => 前面的值
     * （3）解析thenRule， 其实就是 => 后面的值。
     * 
     * @param url 
     */
    public ConditionRouter(URL url) {
        this.url = url;
        this.priority = url.getParameter(Constants.PRIORITY_KEY, 0);
        this.force = url.getParameter(Constants.FORCE_KEY, false);
        try {
            // 拆分条件变大时为 when 和 then 两部分
            String rule = url.getParameterAndDecoded(Constants.RULE_KEY);
            if (rule == null || rule.trim().length() == 0) {
                throw new IllegalArgumentException("Illegal route rule!");
            }
            rule = rule.replace("consumer.", "").replace("provider.", "");
            int i = rule.indexOf("=>");
            String whenRule = i < 0 ? null : rule.substring(0, i).trim();
            String thenRule = i < 0 ? rule.trim() : rule.substring(i + 2).trim();
            // 解析 `whenCondition`
            Map<String, MatchPair> when = StringUtils.isBlank(whenRule) || "true".equals(whenRule) ? new HashMap<String, MatchPair>() : parseRule(whenRule);
            // 解析 `thenCondition`
            Map<String, MatchPair> then = StringUtils.isBlank(thenRule) || "false".equals(thenRule) ? null : parseRule(thenRule);
            // NOTE: It should be determined on the business level whether the `When condition` can be empty or not.
            this.whenCondition = when;
            this.thenCondition = then;
        } catch (ParseException e) {
            throw new IllegalStateException(e.getMessage(), e);
        }
    }

```

#### 说明：
 * （1）解析基本参数
 * （2）解析whenRule， 其实就是 => 前面的值
 * （3）解析thenRule， 其实就是 => 后面的值。


#### parseRule方法:

```java
private static Map<String, MatchPair> parseRule(String rule) throws ParseException {
        Map<String, MatchPair> condition = new HashMap<String, MatchPair>();
        if (StringUtils.isBlank(rule)) {
            return condition;
        }
        // Key-Value pair, stores both match and mismatch conditions
        MatchPair pair = null;
        // Multiple values
        Set<String> values = null;
        final Matcher matcher = ROUTE_PATTERN.matcher(rule);
        while (matcher.find()) { // Try to match one by one
            String separator = matcher.group(1);
            String content = matcher.group(2);
            // Start part of the condition expression.
            if (separator == null || separator.length() == 0) {
                pair = new MatchPair();
                condition.put(content, pair);
            }
            // The KV part of the condition expression
            else if ("&".equals(separator)) {
                if (condition.get(content) == null) {
                    pair = new MatchPair();
                    condition.put(content, pair);
                } else {
                    pair = condition.get(content);
                }
            }
            // The Value in the KV part.
            else if ("=".equals(separator)) {
                if (pair == null) {
                    throw new ParseException("Illegal route rule \"" + rule + "\", The error char '" + separator + "' at index " + matcher.start() + " before \"" + content + "\".", matcher.start());
                }
                values = pair.matches;
                values.add(content);
            }
            // The Value in the KV part.
            else if ("!=".equals(separator)) {
                if (pair == null) {
                    throw new ParseException("Illegal route rule \"" + rule + "\", The error char '" + separator + "' at index " + matcher.start() + " before \"" + content + "\".", matcher.start());
                }
                values = pair.mismatches;
                values.add(content);
            }
            // The Value in the KV part, if Value have more than one items.
            else if (",".equals(separator)) { // Should be seperateed by ','
                if (values == null || values.isEmpty()) {
                    throw new ParseException("Illegal route rule \"" + rule + "\", The error char '" + separator + "' at index " + matcher.start() + " before \"" + content + "\".", matcher.start());
                }
                values.add(content);
            } else {
                throw new ParseException("Illegal route rule \"" + rule + "\", The error char '" + separator + "' at index " + matcher.start() + " before \"" + content + "\".", matcher.start());
            }
        }
        return condition;
    }
```

#### 说明：
> （1）如果是&条件的话，其实相当于 condition 的条件模块，初始化MatchPair
> (2) 如果是=条件的话，相当于获取condition的value 值模块，那么 values进行更新
> （3）如果是！=的话，也是值模块，不过更新的是mismatches.
> (4) 如果是，的话，依旧更新值模块


### MatchPair类

```java
    private static final class MatchPair {

        /**
         * 匹配的值集合
         */
        final Set<String> matches = new HashSet<String>();
        /**
         * 不匹配的值集合
         */
        final Set<String> mismatches = new HashSet<String>();

        /**
         * 判断 value 是否匹配 matches + mismatches
         * (1) 如果只有matches的话，那么只匹配matches 匹配就返回，否则返回false
         * （2）如果只有迷失matches的话，那么就只匹配mismatches，如果匹配到返回false，否则返回true
         * （3）两者都有的话，先匹配mismatches，后匹配 matches。
         *  
         * @param value 值
         * @param param URL
         * @return 是否匹配
         */
        private boolean isMatch(String value, URL param) {
            // 只匹配 matches
            if (!matches.isEmpty() && mismatches.isEmpty()) {
                for (String match : matches) {
                    if (UrlUtils.isMatchGlobPattern(match, value, param)) {
                        return true;
                    }
                }
                return false; // 如果没匹配上，认为为 false ，即不匹配
            }

            // 只匹配 mismatches
            if (!mismatches.isEmpty() && matches.isEmpty()) {
                for (String mismatch : mismatches) {
                    if (UrlUtils.isMatchGlobPattern(mismatch, value, param)) {
                        return false;
                    }
                }
                return true; // 注意，这里和上面不同。原因，你懂的。
            }

            // 匹配 mismatches + matches
            if (!matches.isEmpty()) {
                //when both mismatches and matches contain the same value, then using mismatches first
                for (String mismatch : mismatches) {
                    if (UrlUtils.isMatchGlobPattern(mismatch, value, param)) {
                        return false;
                    }
                }
                for (String match : matches) {
                    if (UrlUtils.isMatchGlobPattern(match, value, param)) {
                        return true;
                    }
                }
                return false; // 如果没匹配上，认为为 false ，即不匹配
            }
            return false;
        }
    }

```
#### 说明：
* (1) 如果只有matches的话，那么只匹配matches 匹配就返回，否则返回false
* （2）如果只有迷失matches的话，那么就只匹配mismatches，如果匹配到返回false，否则返回true
* （3）两者都有的话，先匹配mismatches，后匹配 matches。

#### isMatchGlobPattern 方法：

```java
    public static boolean isMatchGlobPattern(String pattern, String value, URL param) {
        // 以美元符 `$` 开头，表示引用参数,需要从 URL param 里面获取对应的值
        if (param != null && pattern.startsWith("$")) {
            pattern = param.getRawParameter(pattern.substring(1));
        }
        // 匹配
        return isMatchGlobPattern(pattern, value);
    }

    /**
     * （1） 如果是通配符的话，直接返回true
     * （2） 全部为空也返回true
     * （3）有一个不为空返回false
     * （4）找到通配符 *
     * （5）如果通配符在末尾，则看前面的数据是否匹配
     * （6）如果通配符在前面，则看后面的数据是否一致。
     * （7）如果在中间的话，需要前后都匹配
     * @param pattern
     * @param value
     * @return
     */

    public static boolean isMatchGlobPattern(String pattern, String value) {
        // 全匹配
        if ("*".equals(pattern)) {
            return true;
        }
        // 全部为空，匹配
        if ((pattern == null || pattern.length() == 0) && (value == null || value.length() == 0)) {
            return true;
        }
        // 有一个为空，不匹配
        if ((pattern == null || pattern.length() == 0) || (value == null || value.length() == 0)) {
            return false;
        }

        // 支持 * 的通配
        int i = pattern.lastIndexOf('*');
        // doesn't find "*"
        if (i == -1) {
            return value.equals(pattern);
        }
        // "*" is at the end
        else if (i == pattern.length() - 1) {
            return value.startsWith(pattern.substring(0, i));
        }
        // "*" is at the beginning
        else if (i == 0) {
            return value.endsWith(pattern.substring(i + 1));
        }
        // "*" is in the middle
        else {
            String prefix = pattern.substring(0, i);
            String suffix = pattern.substring(i + 1);
            return value.startsWith(prefix) && value.endsWith(suffix);
        }
    }

```
#### 说明：
 * （1） 如果是通配符的话，直接返回true
 * （2） 全部为空也返回true
 * （3）有一个不为空返回false
 * （4）找到通配符 *
 * （5）如果通配符在末尾，则看前面的数据是否匹配
 * （6）如果通配符在前面，则看后面的数据是否一致。
 * （7）如果在中间的话，需要前后都匹配


#### 真正的route 路由方法：

```java
    /**
     * (1) 如果没有matchWhen条件的话，直接返回invokers
     * （2）如果match了when条件了，thenCondition为空，说明满足when条件的后续没有符合条件的，直接返回空
     * （3）对每一个invoker，判断是否matchThen，添加到result集合里面
     * （4）如果 `force=true` ，代表强制执行，返回空 Invoker 集合
     * @param invokers   Invoker 集合 
     * @param url        refer url
     * @param invocation
     * @param <T>
     * @return
     * @throws RpcException
     */
    public <T> List<Invoker<T>> route(List<Invoker<T>> invokers, URL url, Invocation invocation) throws RpcException {
        // 为空，直接返回空 Invoker 集合
        if (invokers == null || invokers.isEmpty()) {
            return invokers;
        }
        try {
            // 不匹配 `whenCondition` ，直接返回 `invokers` 集合，因为不需要走 `whenThen` 的匹配
            if (!matchWhen(url, invocation)) {
                return invokers;
            }
            List<Invoker<T>> result = new ArrayList<Invoker<T>>();
            // `whenThen` 为空，则返回空 Invoker 集合
            if (thenCondition == null) {
                logger.warn("The current consumer in the service blacklist. consumer: " + NetUtils.getLocalHost() + ", service: " + url.getServiceKey());
                return result;
            }
            // 使用 `whenThen` ，匹配 `invokers` 集合。若符合，添加到 `result` 中
            for (Invoker<T> invoker : invokers) {
                if (matchThen(invoker.getUrl(), url)) {
                    result.add(invoker);
                }
            }
            // 若 `result` 非空，返回它
            if (!result.isEmpty()) {
                return result;
            // 如果 `force=true` ，代表强制执行，返回空 Invoker 集合
            } else if (force) {
                logger.warn("The route result is empty and force execute. consumer: " + NetUtils.getLocalHost() + ", service: " + url.getServiceKey() + ", router: " + url.getParameterAndDecoded(Constants.RULE_KEY));
                return result;
            }
        } catch (Throwable t) {
            logger.error("Failed to execute condition router rule: " + getUrl() + ", invokers: " + invokers + ", cause: " + t.getMessage(), t);
        }
        // 如果 `force=false` ，代表不强制执行，返回 `invokers` 集合，即忽略路由规则
        return invokers;
    }
    
boolean matchWhen(URL url, Invocation invocation) {
        return whenCondition == null || whenCondition.isEmpty() || matchCondition(whenCondition, url, null, invocation);
}

private boolean matchThen(URL url, URL param) {
    return !(thenCondition == null || thenCondition.isEmpty()) && matchCondition(thenCondition, url, param, null);
}


```
#### 说明：
* (1) 如果没有matchWhen条件的话，直接返回invokers
* （2）如果match了when条件了，thenCondition为空，说明满足when条件的后续没有符合条件的，直接返回空
* （3）对每一个invoker，判断是否matchThen，添加到result集合里面
* （4）如果 `force=true` ，代表强制执行，返回空 Invoker 集合
* （5）判断条件真正调用的是#matchCondition方法

#### matchCondition方法：

```java
    /**
     * (1) 将url的参数和值转换为sample
     * （2）如果key是方法的话，获取方法名
     * （3）否则直接获取key对应的value，如果是null的话，增加一个 default.前缀获取value
     * （4）就是调用MatchPair的isMatch方法判断是否符合要求
     * 
     * @param condition 
     * @param url
     * @param param
     * @param invocation
     * @return
     */
private boolean matchCondition(Map<String, MatchPair> condition, URL url, URL param, Invocation invocation) {
    Map<String, String> sample = url.toMap();
    boolean result = false; // 是否匹配
    for (Map.Entry<String, MatchPair> matchPair : condition.entrySet()) {
        // 获得条件属性
        String key = matchPair.getKey();
        String sampleValue;
        // get real invoked method name from invocation
        if (invocation != null && (Constants.METHOD_KEY.equals(key) || Constants.METHODS_KEY.equals(key))) {
            sampleValue = invocation.getMethodName();
        } else {
            sampleValue = sample.get(key);
            if (sampleValue == null) {
                sampleValue = sample.get(Constants.DEFAULT_KEY_PREFIX + key);
            }
        }
        // 匹配条件值
        if (sampleValue != null) {
            if (!matchPair.getValue().isMatch(sampleValue, param)) { // 返回不匹配
                return false;
            } else {
                result = true;
            }
        } else {
            // not pass the condition
            if (!matchPair.getValue().matches.isEmpty()) { // 无条件值，但是有匹配条件 `matches` ，则返回不匹配。
                return false;
            } else {
                result = true;
            }
        }
    }
    return result;
}
```
#### 说明：
 * (1) 将url的参数和值转换为sample
 * （2）如果key是方法的话，获取方法名
 * （3）否则直接获取key对应的value，如果是null的话，增加一个 default.前缀获取value
 * （4）就是调用MatchPair的isMatch方法判断是否符合要求
