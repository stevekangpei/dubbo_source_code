# dubbo源码-dubbo SPI机制


## SPI介绍
```
SPI 全称为 Service Provider Interface，是一种服务发现机制。
SPI 的本质是将接口实现类的全限定名配置在文件中，并由服务加载器读取配置文件，加载实现类。
这样可以在运行时，动态为接口替换实现类。正因此特性，
我们可以很容易的通过 SPI 机制为我们的程序提供拓展功能。

在面向的对象的设计里，不同模块之间推崇面向接口编程，不建议在模块中对实现类进行硬编码。
一旦代码里涉及具体的实现类，就违反了可拔插的原则，如果需要替换一种实现，
就需要修改代码。
SPI使得程序能在ClassPath路径下的META-INF/services文件夹查找接口的实现类，
自动加载文件里所定义的实现类。

```

```SPI 实际上是“基于接口的编程＋策略模式＋配置文件”组合实现的动态加载机制```

## 为什么dubbo优化java的SPI

```
JDK 标准的 SPI 会一次性实例化扩展点所有实现，如果有扩展实现初始化很耗时，但如果没用上也加载，会很浪费资源。
Dubbo 有很多的拓展点，例如 Protocol、Filter 等等。并且每个拓展点有多种的实现，例如 Protocol 有 DubboProtocol、InjvmProtocol、RestProtocol 等等。那么使用 JDK SPI 机制，会初始化无用的拓展点及其实现，造成不必要的耗时与资源浪费
```

我们经常在dubbo的SPI中看到如下的代码，
```java
ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(name)
```


#### 成员变量和构造器：
```java
    private static final String SERVICES_DIRECTORY = "META-INF/services/";

    private static final String DUBBO_DIRECTORY = "META-INF/dubbo/";

    private static final String DUBBO_INTERNAL_DIRECTORY = DUBBO_DIRECTORY + "internal/";

    private static final Pattern NAME_SEPARATOR = Pattern.compile("\\s*[,]+\\s*");

    /**
     * 拓展加载器集合
     *
     * key：拓展接口
     */
    private static final ConcurrentMap<Class<?>, ExtensionLoader<?>> EXTENSION_LOADERS = new ConcurrentHashMap<Class<?>, ExtensionLoader<?>>();
    /**
     * 拓展实现类集合
     *
     * key：拓展实现类
     * value：拓展对象。
     *
     * 例如，key 为 Class<AccessLogFilter>
     *      value 为 AccessLogFilter 对象
     */
    private static final ConcurrentMap<Class<?>, Object> EXTENSION_INSTANCES = new ConcurrentHashMap<Class<?>, Object>();

    /**
     * 拓展接口。
     * 例如，Protocol
     */
    private final Class<?> type;
    /**
     * 对象工厂
     *
     * 用于调用 {@link #injectExtension(Object)} 方法，向拓展对象注入依赖的属性。
     *
     * 例如，StubProxyFactoryWrapper 中有 `Protocol protocol` 属性。
     */
    private final ExtensionFactory objectFactory;
    /**
     * 缓存的拓展名与拓展类的映射。
     *
     * 和 {@link #cachedClasses} 的 KV 对调。
     *
     * 通过 {@link #loadExtensionClasses} 加载
     */
    private final ConcurrentMap<Class<?>, String> cachedNames = new ConcurrentHashMap<Class<?>, String>();

// 缓存的拓展名与拓展类的映射。
private final Holder<Map<String, Class<?>>> cachedClasses = new Holder<Map<String, Class<?>>>();

    /**
     * 拓展名与 @Activate 的映射
     *
     * 例如，AccessLogFilter。
     *
     * 用于 {@link #getActivateExtension(URL, String)}
     */
    private final Map<String, Activate> cachedActivates = new ConcurrentHashMap<String, Activate>();


    /**
     * 缓存的拓展对象集合
     *
     * key：拓展名
     * value：拓展对象
     *
     * 例如，Protocol 拓展
     *          key：dubbo value：DubboProtocol
     *          key：injvm value：InjvmProtocol
     *
     * 通过 {@link #loadExtensionClasses} 加载
     */
    private final ConcurrentMap<String, Holder<Object>> cachedInstances = new ConcurrentHashMap<String, Holder<Object>>();

    /**
     * 缓存的自适应( Adaptive )拓展对象
     */
    private final Holder<Object> cachedAdaptiveInstance = new Holder<Object>();
    /**
     * 缓存的自适应拓展对象的类
     *
     * {@link #getAdaptiveExtensionClass()}
     */
    private volatile Class<?> cachedAdaptiveClass = null;
    /**
     * 缓存的默认拓展名
     *
     * 通过 {@link SPI} 注解获得
     */
    private String cachedDefaultName;
    /**
     * 创建 {@link #cachedAdaptiveInstance} 时发生的异常。
     *
     * 发生异常后，不再创建，参见 {@link #createAdaptiveExtension()}
     */
    private volatile Throwable createAdaptiveInstanceError;

    /**
     * 拓展 Wrapper 实现类集合
     *
     * 带唯一参数为拓展接口的构造方法的实现类
     *
     * 通过 {@link #loadExtensionClasses} 加载
     */
    private Set<Class<?>> cachedWrapperClasses;


    private ExtensionLoader(Class<?> type) {
        this.type = type;
        objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
    }

```

#### 说明：
* （1）META-INF/dubbo/internal/ 和 META-INF/dubbo/ 目录下，放置 接口全限定名 配置文件，
      每行内容为：拓展名=拓展实现类全限定名
* （2）META-INF/dubbo/ 目录下，用于用户自定义的拓展实现。
* （3）META-INF/service/ 目录下，Java SPI 的配置目录。
* （4）EXTENSION_LOADERS 是一个全局的 加载器集合，可以类似于一个全局的加载器管理容器，Protocol 和 Filter 分别对应一个 ExtensionLoader 对象。
* （5）EXTENSION_INSTANCES 是一个全局的拓展实现类集合，可以类似于一个全局的拓展实现类管理容器。
     比如说: key 为 Class<AccessLogFilter> value 为 AccessLogFilter 对象
* （6）其他的变量，比如说 type，cachedNames 等等 是属于 这个ExtensionLoader 对象内部的变量。
* （7）type 对应的是 当前的拓展接口，比如说 Protocol
* （8）objectFactory 想对象注入依赖
* （9）cachedNames 缓存的拓展名与拓展类的映射。
* （10）cachedActivates 拓展名与 @Activate 的映射
* （11）cachedInstances 缓存的拓展对象集合 key：dubbo value：DubboProtocol


#### getExtensionLoader

```java
    /**
     * 根据拓展点的接口，获得拓展加载器
     * （1）type必须是接口
     * （2）必须有SPI注解
     * （3）如果没有包含的话初始化一个新的loader
     * 
     * @param type 接口
     * @param <T> 泛型
     * @return 加载器
     */
    @SuppressWarnings("unchecked")
    public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
        if (type == null)
            throw new IllegalArgumentException("Extension type == null");
        // 必须是接口
        if (!type.isInterface()) {
            throw new IllegalArgumentException("Extension type(" + type + ") is not interface!");
        }
        // 必须包含 @SPI 注解
        if (!withExtensionAnnotation(type)) {
            throw new IllegalArgumentException("Extension type(" + type +
                    ") is not extension, because WITHOUT @" + SPI.class.getSimpleName() + " Annotation!");
        }

        // 获得接口对应的拓展点加载器
        ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        if (loader == null) {
            EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
            loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        }
        return loader;
    }
```

#### 说明：
 * 根据拓展点的接口，获得拓展加载器
 * （1）type必须是接口
 * （2）必须有SPI注解
 * （3）如果没有包含的话初始化一个新的loader


#### 构造器：
```java
    private ExtensionLoader(Class<?> type) {
        this.type = type;
        objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
```

#### 说明：
* objectFactory 属性，对象工厂，功能上和 Spring IOC 一致。用于调用 #injectExtension(instance) 方法时，向创建的拓展注入其依赖的属性

#### getExtension方法：

```java
    /**
     * （1）如果是true的话，查找默认的拓展对象
     * （2）如果对象在的话，从缓存中查出来返回
     * （3）否则 createExtension 创建缓存对象
     * 
     * @param name 拓展名
     * @return 拓展对象
     */
    @SuppressWarnings("unchecked")
    public T getExtension(String name) {
        if (name == null || name.length() == 0)
            throw new IllegalArgumentException("Extension name == null");
        // 查找 默认的 拓展对象
        if ("true".equals(name)) {
            return getDefaultExtension();
        }
        // 从 缓存中 获得对应的拓展对象
        Holder<Object> holder = cachedInstances.get(name);
        if (holder == null) {
            cachedInstances.putIfAbsent(name, new Holder<Object>());
            holder = cachedInstances.get(name);
        }
        Object instance = holder.get();
        if (instance == null) {
            synchronized (holder) {
                instance = holder.get();
                // 从 缓存中 未获取到，进行创建缓存对象。
                if (instance == null) {
                    instance = createExtension(name);
                    // 设置创建对象到缓存中
                    holder.set(instance);
                }
            }
        }
        return (T) instance;
    }
```
#### 说明：
 * （1）如果是true的话，查找默认的拓展对象
 * （2）如果对象在的话，从缓存中查出来返回
 * （3）否则 createExtension 创建缓存对象


#### createExtension

```java
    /**
     * （1）getExtensionClasses 先获取扩展类，并找扩展类，找不到的话抛出异常
     * （2）从缓存中，获得拓展对象。
     * （3）注入依赖的属性
     * （4）创建 Wrapper 拓展对象，将 instance 包装在其中。
     * （5）wrapper对象也实现了对应的接口，同时初始的时候，传入了instance。
     * （6）相当于装饰者模式，AOP，对功能做了增强。比如说ProtocolFilterWrapper。
     *
     * Wrapper 类同样实现了扩展点接口，但是 Wrapper 不是扩展点的真正实现。
     * 它的用途主要是用于从 ExtensionLoader 返回扩展点时，包装在真正的扩展点实现外。
     * 即从 ExtensionLoader 中返回的实际上是 Wrapper 类的实例
     * ，Wrapper 持有了实际的扩展点实现类。
     * 
     * @param name 拓展名
     * @return 拓展对象
     */
    @SuppressWarnings("unchecked")
    private T createExtension(String name) {
        // 获得拓展名对应的拓展实现类
        Class<?> clazz = getExtensionClasses().get(name);
        if (clazz == null) {
            throw findException(name); // 抛出异常
        }
        try {
            // 从缓存中，获得拓展对象。
            T instance = (T) EXTENSION_INSTANCES.get(clazz);
            if (instance == null) {
                // 当缓存不存在时，创建拓展对象，并添加到缓存中。
                EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
                instance = (T) EXTENSION_INSTANCES.get(clazz);
            }
            // 注入依赖的属性
            injectExtension(instance);
            // 创建 Wrapper 拓展对象
            Set<Class<?>> wrapperClasses = cachedWrapperClasses;
            if (wrapperClasses != null && !wrapperClasses.isEmpty()) {
                for (Class<?> wrapperClass : wrapperClasses) {
                    instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
                }
            }
            return instance;
        } catch (Throwable t) {
            throw new IllegalStateException("Extension instance(name: " + name + ", class: " +
                    type + ")  could not be instantiated: " + t.getMessage(), t);
        }
    }
```

#### 说明：
 * （1）getExtensionClasses 先获取扩展类，并找扩展类，找不到的话抛出异常
 * （2）从缓存中，获得拓展对象。
 * （3）注入依赖的属性
 * （4）创建 Wrapper 拓展对象，将 instance 包装在其中。
 * （5）wrapper对象也实现了对应的接口，同时初始的时候，传入了instance。
 * （6）相当于装饰者模式，AOP，对功能做了增强。比如说ProtocolFilterWrapper。
 *
 * Wrapper 类同样实现了扩展点接口，但是 Wrapper 不是扩展点的真正实现。
 * 它的用途主要是用于从 ExtensionLoader 返回扩展点时，包装在真正的扩展点实现外。
 * 即从 ExtensionLoader 中返回的实际上是 Wrapper 类的实例
 * ，Wrapper 持有了实际的扩展点实现类。
 
 
 #### getExtensionClasses

 ```java
     private Map<String, Class<?>> getExtensionClasses() {
        // 从缓存中，获得拓展实现类数组
        Map<String, Class<?>> classes = cachedClasses.get();
        if (classes == null) {
            synchronized (cachedClasses) {
                classes = cachedClasses.get();
                if (classes == null) {
                    // 从配置文件中，加载拓展实现类数组
                    classes = loadExtensionClasses();
                    // 设置到缓存中
                    cachedClasses.set(classes);
                }
            }
        }
        return classes;
    }
 ```
 
 #### 说明：
 * （1）cachedClasses 属性，缓存的拓展实现类集合。它不包含如下两种类型的拓展实现：
      自适应拓展实现类。例如 AdaptiveExtensionFactory 。
      拓展 Adaptive 实现类，会添加到 cachedAdaptiveClass 属性中。
      带唯一参数为拓展接口的构造方法的实现类，或者说拓展 Wrapper 实现类。例如，               ProtocolFilterWrapper 。
* （2）拓展 Wrapper 实现类，会添加到 cachedWrapperClasses 属性中。
* （3）总结来说，cachedClasses + cachedAdaptiveClass + cachedWrapperClasses 才是完整缓存的拓展实现类的配置。

 


#### loadExtensionClasses

```java
    /**
     * 加载拓展实现类数组
     * 
     * （1）从传入的type class接口获取SPI注解
     * （2）更新cachedDefaultName 默认的扩展实现
     * （3）从配置文件中，加载拓展实现类数组，分别是META-INF/dubbo/internal, META-INF/dubbo/, META-INF/services/
     * 
     * @return 拓展实现类数组
     */
    private Map<String, Class<?>> loadExtensionClasses() {
        // 通过 @SPI 注解，获得默认的拓展实现类名
        final SPI defaultAnnotation = type.getAnnotation(SPI.class);
        if (defaultAnnotation != null) {
            String value = defaultAnnotation.value();
            if ((value = value.trim()).length() > 0) {
                String[] names = NAME_SEPARATOR.split(value);
                if (names.length > 1) {
                    throw new IllegalStateException("more than 1 default extension name on extension " + type.getName()
                            + ": " + Arrays.toString(names));
                }
                if (names.length == 1) cachedDefaultName = names[0];
            }
        }

        // 从配置文件中，加载拓展实现类数组
        Map<String, Class<?>> extensionClasses = new HashMap<String, Class<?>>();
        loadFile(extensionClasses, DUBBO_INTERNAL_DIRECTORY);
        loadFile(extensionClasses, DUBBO_DIRECTORY);
        loadFile(extensionClasses, SERVICES_DIRECTORY);
        return extensionClasses;
    }
```

#### 说明：
 * （1）从传入的type class接口获取SPI注解
 * （2）更新cachedDefaultName 默认的扩展实现
 * （3）从配置文件中，加载拓展实现类数组，分别是META-INF/dubbo/internal, META-INF/dubbo/, META-INF/services/

#### loadFile

```java
    /**
     *  （1）先加载文件 classLoader.getResources(fileName);
     *  （2）过滤掉注释掉的内容 line = line.substring(0, ci);
     *  （3）拆分，key=value 的配置格式 比如说 kryo=com.alibaba.dubbo.common.serialize.support.kryo.KryoSerialization
     *  （4）判断当前类是否实现了扩展接口。
     *  （5）如果是自适应对象的话，缓存自适应拓展对象的类到 `cachedAdaptiveClass`
     *  （6）判断是不是wrapper类，如果是wrapper类，那么必然有当前实例作为参数的构造方法，这个方法就不会报错。clazz.getConstructor(type);
     *      如果是 缓存拓展 Wrapper 实现类到 `cachedWrapperClasses`。
     *  （7）缓存到cachenames 和 extensionClasses 里面
     *  
     *  
     * @param extensionClasses 拓展类名数组
     * @param dir 文件名
     */
    private void loadFile(Map<String, Class<?>> extensionClasses, String dir) {
        // 完整的文件名
        String fileName = dir + type.getName();
        try {
            Enumeration<java.net.URL> urls;
            // 获得文件名对应的所有文件数组
            ClassLoader classLoader = findClassLoader();
            if (classLoader != null) {
                urls = classLoader.getResources(fileName);
            } else {
                urls = ClassLoader.getSystemResources(fileName);
            }
            // 遍历文件数组
            if (urls != null) {
                while (urls.hasMoreElements()) {
                    java.net.URL url = urls.nextElement();
                    try {
                        BufferedReader reader = new BufferedReader(new InputStreamReader(url.openStream(), "utf-8"));
                        try {
                            String line;
                            while ((line = reader.readLine()) != null) {
                                // 跳过当前被注释掉的情况，例如 #spring=xxxxxxxxx
                                final int ci = line.indexOf('#');
                                if (ci >= 0) line = line.substring(0, ci);
                                line = line.trim();
                                if (line.length() > 0) {
                                    try {
                                        // 拆分，key=value 的配置格式
                                        String name = null;
                                        int i = line.indexOf('=');
                                        if (i > 0) {
                                            name = line.substring(0, i).trim();
                                            line = line.substring(i + 1).trim();
                                        }
                                        if (line.length() > 0) {
                                            // 判断拓展实现，是否实现拓展接口
                                            Class<?> clazz = Class.forName(line, true, classLoader);
                                            if (!type.isAssignableFrom(clazz)) {
                                                throw new IllegalStateException("Error when load extension class(interface: " +
                                                        type + ", class line: " + clazz.getName() + "), class "
                                                        + clazz.getName() + "is not subtype of interface.");
                                            }
                                            // 缓存自适应拓展对象的类到 `cachedAdaptiveClass`
                                            if (clazz.isAnnotationPresent(Adaptive.class)) {
                                                if (cachedAdaptiveClass == null) {
                                                    cachedAdaptiveClass = clazz;
                                                } else if (!cachedAdaptiveClass.equals(clazz)) {
                                                    throw new IllegalStateException("More than 1 adaptive class found: "
                                                            + cachedAdaptiveClass.getClass().getName()
                                                            + ", " + clazz.getClass().getName());
                                                }
                                            } else {
                                                // 缓存拓展 Wrapper 实现类到 `cachedWrapperClasses`
                                                try {
                                                    clazz.getConstructor(type);
                                                    Set<Class<?>> wrappers = cachedWrapperClasses;
                                                    if (wrappers == null) {
                                                        cachedWrapperClasses = new ConcurrentHashSet<Class<?>>();
                                                        wrappers = cachedWrapperClasses;
                                                    }
                                                    wrappers.add(clazz);
                                                // 缓存拓展实现类到 `extensionClasses`
                                                } catch (NoSuchMethodException e) {
                                                    clazz.getConstructor();
                                                    // 未配置拓展名，自动生成。例如，DemoFilter 为 demo 。主要用于兼容 Java SPI 的配置。
                                                    if (name == null || name.length() == 0) {
                                                        name = findAnnotationName(clazz);
                                                        if (name == null || name.length() == 0) {
                                                            if (clazz.getSimpleName().length() > type.getSimpleName().length()
                                                                    && clazz.getSimpleName().endsWith(type.getSimpleName())) {
                                                                name = clazz.getSimpleName().substring(0, clazz.getSimpleName().length() - type.getSimpleName().length()).toLowerCase();
                                                            } else {
                                                                throw new IllegalStateException("No such extension name for the class " + clazz.getName() + " in the config " + url);
                                                            }
                                                        }
                                                    }
                                                    // 获得拓展名，可以是数组，有多个拓展名。
                                                    String[] names = NAME_SEPARATOR.split(name);
                                                    if (names != null && names.length > 0) {
                                                        // 缓存 @Activate 到 `cachedActivates` 。
                                                        Activate activate = clazz.getAnnotation(Activate.class);
                                                        if (activate != null) {
                                                            cachedActivates.put(names[0], activate);
                                                        }
                                                        for (String n : names) {
                                                            // 缓存到 `cachedNames`
                                                            if (!cachedNames.containsKey(clazz)) {
                                                                cachedNames.put(clazz, n);
                                                            }
                                                            // 缓存拓展实现类到 `extensionClasses`
                                                            Class<?> c = extensionClasses.get(n);
                                                            if (c == null) {
                                                                extensionClasses.put(n, clazz);
                                                            } else if (c != clazz) {
                                                                throw new IllegalStateException("Duplicate extension " + type.getName() + " name " + n + " on " + c.getName() + " and " + clazz.getName());
                                                            }
                                                        }
                                                    }
                                                }
                                            }
                                        }
                                    } catch (Throwable t) {
                                        // 发生异常，记录到异常集合
                                        IllegalStateException e = new IllegalStateException("Failed to load extension class(interface: " + type + ", class line: " + line + ") in " + url + ", cause: " + t.getMessage(), t);
                                        exceptions.put(line, e);
                                    }
                                }
                            } // end of while read lines
                        } finally {
                            reader.close();
                        }
                    } catch (Throwable t) {
                        logger.error("Exception when load extension class(interface: " +
                                type + ", class file: " + url + ") in " + url, t);
                    }
                } // end of while urls
            }
        } catch (Throwable t) {
            logger.error("Exception when load extension class(interface: " +
                    type + ", description file: " + fileName + ").", t);
        }
    }
```

#### 说明：
 *  （1）先加载文件 classLoader.getResources(fileName);
 *  （2）过滤掉注释掉的内容 line = line.substring(0, ci);
 *  （3）拆分，key=value 的配置格式 比如说 kryo=com.alibaba.dubbo.common.serialize.support.kryo.KryoSerialization
 *  （4）判断当前类是否实现了扩展接口。
 *  （5）如果是自适应对象的话，缓存自适应拓展对象的类到 `cachedAdaptiveClass`
 *  （6）判断是不是wrapper类，如果是wrapper类，那么必然有当前实例作为参数的构造方法，这个方法就不会报错。clazz.getConstructor(type);
 *      如果是 缓存拓展 Wrapper 实现类到 `cachedWrapperClasses`。
 *  （7）缓存到cachenames 和 extensionClasses 里面



#### injectExtension

```java
    /**
     * 注入依赖的属性
     * (1) 找到 public 的set方法
     *（2） 通过 objectFactory.getExtension(pt, property);获取属性值
     *（3） 通过 method.invoke(instance, object); 设置属性值
     * @param instance 拓展对象
     * @return 拓展对象
     */
    private T injectExtension(T instance) {
        try {
            if (objectFactory != null) {
                for (Method method : instance.getClass().getMethods()) {
                    if (method.getName().startsWith("set")
                            && method.getParameterTypes().length == 1
                            && Modifier.isPublic(method.getModifiers())) { // setting && public 方法
                        // 获得属性的类型
                        Class<?> pt = method.getParameterTypes()[0];
                        try {
                            // 获得属性
                            String property = method.getName().length() > 3 ? method.getName().substring(3, 4).toLowerCase() + method.getName().substring(4) : "";
                            // 获得属性值
                            Object object = objectFactory.getExtension(pt, property);
                            // 设置属性值
                            if (object != null) {
                                method.invoke(instance, object);
                            }
                        } catch (Exception e) {
                            logger.error("fail to inject via method " + method.getName()
                                    + " of interface " + type.getName() + ": " + e.getMessage(), e);
                        }
                    }
                }
            }
        } catch (Exception e) {
            logger.error(e.getMessage(), e);
        }
        return instance;
    }
```

#### 说明：
 *  注入依赖的属性
 *  (1) 找到 public 的set方法
 * （2） 通过 objectFactory.getExtension(pt, property);获取属性值
 * （3） 通过 method.invoke(instance, object); 设置属性值


### 获取自适应的扩展点对象
```java
在dubbo中有很多这种代码 getAdaptiveExtension 获取自适应的实现。
ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension()
```

#### getAdaptiveExtension

```java
    /**
     * 获得自适应拓展对象
     * (1) 从缓存 cachedAdaptiveInstance 属性中，获得自适应拓展对象。
     * (2) 之前的报错被缓存了起来，若之前创建报错，则抛出异常 IllegalStateException 。
     *（3）当缓存不存在时，调用 #createAdaptiveExtension() 方法，创建自适应拓展对象，并添加到 cachedAdaptiveInstance 中。
     *（4）若创建发生异常，记录异常到 createAdaptiveInstanceError
     * @return 拓展对象
     */
    @SuppressWarnings("unchecked")
    public T getAdaptiveExtension() {
        // 从缓存中，获得自适应拓展对象
        Object instance = cachedAdaptiveInstance.get();
        if (instance == null) {
            // 若之前未创建报错，
            if (createAdaptiveInstanceError == null) {
                synchronized (cachedAdaptiveInstance) {
                    instance = cachedAdaptiveInstance.get();
                    if (instance == null) {
                        try {
                            // 创建自适应拓展对象
                            instance = createAdaptiveExtension();
                            // 设置到缓存
                            cachedAdaptiveInstance.set(instance);
                        } catch (Throwable t) {
                            // 记录异常
                            createAdaptiveInstanceError = t;
                            throw new IllegalStateException("fail to create adaptive instance: " + t.toString(), t);
                        }
                    }
                }
            // 若之前创建报错，则抛出异常 IllegalStateException
            } else {
                throw new IllegalStateException("fail to create adaptive instance: " + createAdaptiveInstanceError.toString(), createAdaptiveInstanceError);
            }
        }
        return (T) instance;
    }
```

#### 说明：
 * 获得自适应拓展对象
 * (1) 从缓存 cachedAdaptiveInstance 属性中，获得自适应拓展对象。
 * (2) 之前的报错被缓存了起来，若之前创建报错，则抛出异常 IllegalStateException 。
 * （3）当缓存不存在时，调用 #createAdaptiveExtension() 方法，创建自适应拓展对象，并添加到 cachedAdaptiveInstance 中。
 * （4）若创建发生异常，记录异常到 createAdaptiveInstanceError



#### createAdaptiveExtension:
```java
/**
 * 创建自适应拓展对象
 *
 * @return 拓展对象
 */
@SuppressWarnings("unchecked")
private T createAdaptiveExtension() {
    try {
        return injectExtension((T) getAdaptiveExtensionClass().newInstance());
    } catch (Exception e) {
        throw new IllegalStateException("Can not create adaptive extension " + type + ", cause: " + e.getMessage(), e);
    }
}
```

#### 说明：
* #getAdaptiveExtensionClass() 方法，获得自适应拓展类。
* Class#newInstance() 方法，创建自适应拓展对象。
* #injectExtension(instance) 方法，注入属性。


#### getAdaptiveExtensionClass
```java
    /**
     * @return 自适应拓展类
     */
    private Class<?> getAdaptiveExtensionClass() {
        getExtensionClasses();
        if (cachedAdaptiveClass != null) {
            return cachedAdaptiveClass;
        }
        return cachedAdaptiveClass = createAdaptiveExtensionClass();
    }
```
#### 说明：
* 若 cachedAdaptiveClass 已存在，直接返回
* 调用 #createAdaptiveExtensionClass() 方法，自动生成自适应拓展的代码实现，并编译后返回该类。


#### createAdaptiveExtensionClass:
```java
    private Class<?> createAdaptiveExtensionClass() {
        // 自动生成自适应拓展的代码实现的字符串
        String code = createAdaptiveExtensionClassCode();
        // 编译代码，并返回该类
        ClassLoader classLoader = findClassLoader();
        com.alibaba.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
        return compiler.compile(code, classLoader);
    }
```
#### 说明：
* createAdaptiveExtensionClassCode 生成一段代码
* 找到类加载器和compiler，并编译该代码。


## 总结：

* （1）ExtensionLoader内部有容器，根据不同的扩展点存放了对应的ExtensionLoader，比如说Protocol对应的ExtensionLoader
* （2）初始化的时候，从配置中加载对应的class文件。并分开常规class和wrapper class，并缓存起来
* （3）通过class类初始化instance实例，然后通过属性注入。如果是wrapper类的话，则返回的是对应的wrapper类。
* （4）自适应的class，会自动生成自适应拓展的代码实现的字符串，然后编译。


