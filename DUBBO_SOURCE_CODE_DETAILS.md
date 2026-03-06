# Dubbo 源码实现细节与代码示例

## 目录

1. [核心接口定义](#核心接口定义)
2. [关键实现类](#关键实现类)
3. [源代码示例](#源代码示例)
4. [常见扩展点](#常见扩展点)
5. [性能优化点](#性能优化点)
6. [故障排查](#故障排查)

---

## 核心接口定义

### 1. Invoker 接口

**定义位置:** `dubbo-rpc/dubbo-rpc-api/src/main/java/org/apache/dubbo/rpc/Invoker.java`

```java
/**
 * Invoker 代表一个可调用的服务实现
 * 三种常见类型：
 * 1. ProviderInvoker - 提供者端的本地实现
 * 2. ConsumerInvoker - 消费者端的远程代理
 * 3. ClusterInvoker - 集群处理后的Invoker
 */
public interface Invoker<T> extends Node {
    
    /**
     * 获取服务接口类型
     */
    Class<T> getInterface();
    
    /**
     * 执行RPC调用
     * @param invocation 调用信息，包含方法名、参数等
     * @return 调用结果
     * @throws RpcException RPC异常
     */
    Result invoke(Invocation invocation) throws RpcException;
}
```

**Node接口 (父接口):**
```java
public interface Node {
    // 获取服务URL标识
    URL getUrl();
    
    // 判断是否可用
    boolean isAvailable();
    
    // 销毁资源
    void destroy();
}
```

### 2. Protocol 接口

**定义位置:** `dubbo-rpc/dubbo-rpc-api/src/main/java/org/apache/dubbo/rpc/Protocol.java`

```java
/**
 * RPC协议扩展点 (SPI)
 * Dubbo框架通过Protocol实现不同的通信协议
 * 
 * 主要实现：
 * - triple (gRPC兼容)
 * - dubbo (Dubbo二进制协议)
 * - rest (HTTP REST)
 * - injvm (本地JVM调用)
 */
@SPI(value = "dubbo", scope = ExtensionScope.FRAMEWORK)
public interface Protocol {
    
    /**
     * 获取协议的默认端口
     */
    int getDefaultPort();
    
    /**
     * 导出服务 (Provider侧)
     * 
     * 调用约定：
     * 1. Protocol必须幂等，多次调用同一URL结果一致
     * 2. Invoker由框架提供，Protocol无需关心
     * 3. Protocol需记录请求源地址
     * 
     * @param invoker 服务实现的包装
     * @return exporter 用于管理导出的服务
     */
    <T> Exporter<T> export(Invoker<T> invoker) throws RpcException;
    
    /**
     * 引用服务 (Consumer侧)
     * 
     * 调用约定：
     * 1. refer()返回的Invoker需由框架负责销毁
     * 2. Invoker需线程安全
     * 3. 需支持同步和异步调用
     * 
     * @param type 服务接口
     * @param url 服务URL
     * @return invoker 服务代理
     */
    <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException;
}
```

### 3. Registry 接口

**定义位置:** `dubbo-registry/dubbo-registry-api/src/main/java/org/apache/dubbo/registry/Registry.java`

```java
/**
 * 注册中心接口 (SPI)
 * 
 * 实现：
 * - zookeeper (推荐，支持watch)
 * - nacos (云原生)
 * - multicast (本地网络多播)
 * - file (本地文件)
 */
public interface Registry extends Node, RegistryService {
    
    /**
     * 注册服务
     * Provider启动时调用，注册自己可供消费
     * 
     * @param url 服务URL
     *            格式: dubbo://ip:port/service-interface?params
     */
    void register(URL url);
    
    /**
     * 取消注册
     * Provider关闭时调用，注销自己
     */
    void unregister(URL url);
    
    /**
     * 订阅服务变化
     * Consumer启动时调用，监听可用服务实例变化
     * 
     * @param url 订阅的服务URL
     * @param listener 变化通知监听器
     */
    void subscribe(URL url, NotifyListener listener);
    
    /**
     * 取消订阅
     * Consumer关闭时调用
     */
    void unsubscribe(URL url, NotifyListener listener);
    
    /**
     * 查询已注册的服务
     * Consumer启动时获取初始的可用提供者列表
     */
    @Override
    List<URL> lookup(URL url);
}
```

### 4. Cluster 接口

**定义位置:** `dubbo-cluster/src/main/java/org/apache/dubbo/rpc/cluster/Cluster.java`

```java
/**
 * 集群策略 (SPI)
 * 
 * 作用：
 * 1. 从多个提供者中选择一个并调用
 * 2. 处理失败重试策略
 * 3. 实现不同的容错机制
 * 
 * 实现：
 * - failover (失败转移，默认)
 * - failfast (快速失败)
 * - failsafe (失败安全)
 * - failback (失败回退，异步重试)
 * - forking (并行调用)
 * - available (就近可用)
 */
@SPI(Cluster.DEFAULT)  // 默认: failover
public interface Cluster {
    
    String DEFAULT = "failover";
    
    /**
     * 合并Directory中的invoker到虚拟invoker
     * 
     * @param directory 维护当前可用的服务提供者
     * @param buildFilterChain 是否构建过滤器链
     * @return 包装后的invoker，支持路由、负载均衡等
     */
    @Adaptive
    <T> Invoker<T> join(Directory<T> directory, 
                        boolean buildFilterChain) throws RpcException;
}
```

### 5. LoadBalance 接口

**定义位置:** `dubbo-cluster/src/main/java/org/apache/dubbo/rpc/cluster/LoadBalance.java`

```java
/**
 * 负载均衡 (SPI)
 * 
 * 在多个提供者中选择一个进行调用
 * 
 * 实现：
 * - random (随机，默认)
 * - round_robin (轮询)
 * - least_active (最少活跃)
 * - consistent_hash (一致性哈希)
 * - shortest_response (最短响应时间)
 */
@SPI(LoadBalance.RANDOM)
public interface LoadBalance {
    
    String RANDOM = "random";
    
    /**
     * 选择一个服务提供者
     * 
     * @param invokers 所有可用的提供者
     * @param url 消费者URL
     * @param invocation 方法调用信息
     * @return 选中的提供者invoker
     */
    @Adaptive("loadbalance")
    <T> Invoker<T> select(List<Invoker<T>> invokers, 
                          URL url, 
                          Invocation invocation);
}
```

### 6. Filter 接口

**定义位置:** `dubbo-rpc/dubbo-rpc-api/src/main/java/org/apache/dubbo/rpc/Filter.java`

```java
/**
 * RPC拦截器 (SPI)
 * 
 * 用于在调用前后执行逻辑：
 * - 参数验证
 * - 权限认证
 * - 上下文传递
 * - 监控统计
 * - 超时控制
 * - 日志记录
 */
@SPI
public interface Filter {
    
    /**
     * 执行过滤
     * 
     * @param invoker 下一个处理器
     * @param invocation 调用信息
     * @return 调用结果
     */
    Result invoke(Invoker<?> invoker, 
                  Invocation invocation) throws RpcException;
}
```

---

## 关键实现类

### 1. AbstractInvoker - 抽象基类

**位置:** `dubbo-rpc/dubbo-rpc-api/src/main/java/org/apache/dubbo/rpc/protocol/AbstractInvoker.java`

```java
/**
 * Invoker的抽象实现
 * 提供通用功能：
 * - 参数验证
 * - 超时控制
 * - 异常处理
 */
public abstract class AbstractInvoker<T> implements Invoker<T> {
    private final Class<T> type;
    private final URL url;
    
    protected AbstractInvoker(Class<T> type, URL url) {
        this.type = type;
        this.url = url;
    }
    
    @Override
    public Class<T> getInterface() {
        return type;
    }
    
    @Override
    public Result invoke(Invocation invocation) throws RpcException {
        // 1. 验证调用信息
        if (destroyed.get()) {
            throw new RpcException("...");
        }
        
        // 2. 设置调用ID
        RpcInvocation invo = (RpcInvocation) invocation;
        invo.setInvoker(this);
        
        // 3. 执行实际调用
        try {
            Result result = doInvoke(invocation);
            return result;
        } catch (InvocationTargetException e) {
            // 处理业务异常
            ...
        }
    }
    
    /**
     * 由具体协议实现
     */
    protected abstract Result doInvoke(Invocation invocation) 
        throws RpcException;
}
```

### 2. RegistryDirectory - 服务发现

**位置:** `dubbo-registry/dubbo-registry-api/src/main/java/org/apache/dubbo/registry/integration/RegistryDirectory.java`

```java
/**
 * 从注册中心动态获取服务提供者
 * 
 * 功能：
 * 1. 订阅注册中心，获取初始提供者列表
 * 2. 监听提供者变化，动态更新列表
 * 3. 缓存服务元数据和配置
 */
public class RegistryDirectory<T> extends AbstractDirectory<T> 
    implements NotifyListener {
    
    private final Map<String, List<Invoker<T>>> methodInvokers 
        = new ConcurrentHashMap<>();
    
    /**
     * 当注册中心通知服务提供者变化时调用
     */
    @Override
    public synchronized void notify(List<URL> urls) {
        if (urls == null || urls.isEmpty()) {
            return;
        }
        
        // 1. 分类处理URL
        Map<String, List<URL>> categoryUrls = urls.stream()
            .collect(Collectors.groupingBy(URL::getProtocol));
        
        // 2. 解析provider URLs
        List<URL> providerUrls = categoryUrls
            .get(PROVIDERS_CATEGORY);
        
        // 3. 为每个provider URL创建Invoker
        ...
        
        // 4. 应用路由规则
        List<Router> routers = ...;
        for (Router router : routers) {
            invokers = router.route(invokers, url, 
                                    invocation);
        }
        
        // 5. 更新invoker列表
        this.invokers = invokers;
    }
    
    /**
     * 获取可用的服务提供者
     */
    @Override
    public List<Invoker<T>> list(Invocation invocation) {
        List<Invoker<T>> invokers = methodInvokers
            .get(invocation.getMethodName());
        
        if (invokers == null) {
            invokers = new ArrayList<>(this.invokers);
        }
        
        return invokers;
    }
}
```

### 3. FailoverClusterInvoker - 失败转移

**位置:** `dubbo-cluster/src/main/java/org/apache/dubbo/rpc/cluster/support/FailoverClusterInvoker.java`

```java
/**
 * 失败转移（默认集群策略）
 * 
 * 逻辑：
 * 1. 第1次调用A失败 → 转移到调用B
 * 2. B也失败 → 转移到调用C
 * 3. 全部失败 → 抛出异常
 */
public class FailoverClusterInvoker<T> 
    extends AbstractClusterInvoker<T> {
    
    @Override
    protected Result doInvoke(Invocation invocation, 
                              List<Invoker<T>> invokers, 
                              LoadBalance loadbalance) 
        throws RpcException {
        
        List<Invoker<T>> copyInvokers = new ArrayList<>(invokers);
        int len = copyInvokers.size();
        
        // 重试次数
        int invoked = 0;
        RpcException lastException = null;
        
        for (; invoked < len; invoked++) {
            try {
                // 1. 通过负载均衡选择一个invoker
                Invoker<T> invoker = select(loadbalance, 
                                            invocation, 
                                            copyInvokers, 
                                            invoked);
                
                // 2. 执行调用
                Result result = invoker.invoke(invocation);
                
                // 3. 调用成功，返回
                return result;
                
            } catch (RpcException e) {
                lastException = e;
                // 继续下一个提供者
            }
        }
        
        // 4. 全部失败，抛出异常
        throw lastException;
    }
}
```

### 4. URL - 统一资源识别符

**位置:** `dubbo-common/src/main/java/org/apache/dubbo/common/URL.java`

```java
/**
 * Dubbo统一的资源标识符
 * 格式: protocol://user:password@host:port/path?key=value&key2=value2
 * 
 * 例子: dubbo://admin:admin@10.20.130.230:20880/
 *       com.alibaba.dubbo.demo.DemoService?
 *       version=1.0&group=default&timeout=3000
 */
public class URL implements Serializable {
    
    private String protocol;      // 协议: dubbo, triple, http等
    private String username;      // 用户名
    private String password;      // 密码
    private String host;          // 主机名
    private int port;             // 端口
    private String path;          // 路径 (通常是服务接口名)
    private Map<String, String> parameters;  // 参数
    
    /**
     * 获取参数
     * @param key 参数键
     * @param defaultValue 默认值
     */
    public String getParameter(String key, String defaultValue) {
        String value = parameters.get(key);
        return value == null ? defaultValue : value;
    }
    
    /**
     * 获取参数（整数）
     */
    public int getParameter(String key, int defaultValue) {
        Number n = getParameters(key);
        return n == null ? defaultValue : n.intValue();
    }
    
    /**
     * 获取URL的完整字符串表示
     */
    @Override
    public String toString() {
        // protocol://host:port/path?params
        return toFullString();
    }
}
```

### 5. ExtensionLoader - SPI加载器

**位置:** `dubbo-common/src/main/java/org/apache/dubbo/common/extension/ExtensionLoader.java`

```java
/**
 * Dubbo的SPI扩展加载器
 * 
 * 功能：
 * 1. 加载claspath下META-INF/dubbo/配置的实现
 * 2. 自动依赖注入
 * 3. 支持Wrapper包装
 * 4. 支持@Adaptive动态适配
 */
public class ExtensionLoader<T> {
    
    private final Class<T> type;
    private final Map<String, Class<?>> cachedClasses;
    private final Map<Class<?>, Object> extensionInstances;
    
    /**
     * 获取扩展实例
     * @param name 扩展名称
     * @return 扩展实例
     */
    public T getExtension(String name) {
        if (name == null) {
            throw new IllegalArgumentException("name is null");
        }
        
        // 1. 从缓存获取
        T instance = cachedInstances.get(name);
        if (instance != null) {
            return instance;
        }
        
        // 2. 加载
        synchronized (cachedInstances) {
            instance = cachedInstances.get(name);
            if (instance == null) {
                instance = createExtension(name);
                cachedInstances.put(name, instance);
            }
        }
        
        return instance;
    }
    
    /**
     * 创建扩展实例
     */
    private T createExtension(String name) throws RpcException {
        // 1. 获取实现类
        Class<?> clazz = getExtensionClasses().get(name);
        if (clazz == null) {
            throw new RpcException("Extension not found");
        }
        
        // 2. 创建实例
        try {
            T instance = (T) clazz.newInstance();
            
            // 3. 自动注入依赖
            injectExtension(instance);
            
            // 4. 应用Wrapper
            Set<Class<?>> wrappers = cachedWrapperClasses;
            if (wrappers != null) {
                for (Class<?> wrapper : wrappers) {
                    instance = (T) wrapper
                        .getConstructor(type)
                        .newInstance(instance);
                }
            }
            
            return instance;
        } catch (Exception e) {
            throw new RpcException(e);
        }
    }
    
    /**
     * 自动注入依赖
     */
    private void injectExtension(T instance) {
        // 1. 获取所有setter方法
        Method[] methods = instance.getClass()
            .getMethods();
        
        // 2. 查找并调用setter
        for (Method method : methods) {
            if (isSetter(method)) {
                // 3. 获取参数类型
                Class<?> paramType = method
                    .getParameterTypes()[0];
                
                // 4. 递归加载依赖扩展
                if (paramType.isInterface() && 
                    paramType.isAnnotationPresent(SPI.class)) {
                    
                    Object paramInstance = getExtension(
                        paramType);
                    
                    if (paramInstance != null) {
                        method.invoke(instance, paramInstance);
                    }
                }
            }
        }
    }
}
```

---

## 源代码示例

### 1. 完整的服务提供端实现

```java
// application.yml
dubbo:
  application:
    name: demo-provider
  protocol:
    name: triple
    port: 20880
  registry:
    address: nacos://localhost:8848

// 服务接口定义
public interface DemoService {
    String sayHello(String name);
}

// 服务实现
@Service(version = "1.0", group = "default")
public class DemoServiceImpl implements DemoService {
    
    @Override
    public String sayHello(String name) {
        return "Hello, " + name;
    }
}

// 启动类
@SpringBootApplication
@EnableDubbo
public class DemoProviderApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoProviderApplication.class, 
                             args);
    }
}
```

**背后的工作流程：**
```
1. Spring启动时扫描@Service注解
   ↓
2. ServiceConfig创建并配置
   ↓
3. ServiceConfig.export()调用
   ├─ 创建本地Invoker (DemoServiceImpl的代理)
   ├─ 应用Filter链
   ├─ Protocol.export() (Triple协议)
   ├─ 启动Netty服务器监听20880端口
   └─ Registry.register() (注册到Nacos)
   ↓
4. 服务暴露完成，等待消费者调用
```

### 2. 完整的服务消费端实现

```java
// application.yml
dubbo:
  application:
    name: demo-consumer
  registry:
    address: nacos://localhost:8848
  consumer:
    timeout: 5000

// 消费者应用
@Component
public class ConsumerService {
    
    @DubboReference(version = "1.0", group = "default")
    private DemoService demoService;
    
    public void callService() {
        // 调用远程服务
        String result = demoService.sayHello("World");
        System.out.println("Response: " + result);
    }
}

// 启动类
@SpringBootApplication
@EnableDubbo
public class DemoConsumerApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoConsumerApplication.class, 
                             args);
    }
}
```

**背后的工作流程：**
```
1. Spring启动时扫描@DubboReference注解
   ↓
2. ReferenceConfig创建并配置
   ↓
3. ReferenceConfig.get()调用
   ├─ Registry.subscribe() (订阅Nacos)
   ├─ 获取所有可用的Provider URLs
   ├─ RegistryDirectory创建并监听变化
   ├─ 为每个Provider创建Invoker
   ├─ ClusterInvoker包装
   ├─ ProxyFactory.getProxy() (生成动态代理)
   └─ 返回DemoService的代理对象
   ↓
4. 用户调用demoService.sayHello("World")时
   ├─ 代理拦截调用
   ├─ 构建Invocation
   ├─ 应用Consumer Filter链
   ├─ ClusterInvoker.invoke()
   │  ├─ Directory.list() (获取可用Invoker)
   │  ├─ Router.route() (应用路由)
   │  ├─ LoadBalance.select() (选择一个)
   │  └─ 调用选中的Invoker
   ├─ TripleInvoker.invoke()
   │  ├─ 序列化参数
   │  ├─ 构建请求
   │  ├─ 通过Netty发送
   │  └─ 等待响应
   ├─ 接收响应
   ├─ 反序列化结果
   └─ 返回给用户
```

### 3. 自定义Filter示例

```java
/**
 * 自定义Filter：记录每次调用的耗时
 */
@Activate(group = {"provider", "consumer"})
public class TimingFilter implements Filter {
    
    private static final Logger logger = 
        LoggerFactory.getLogger(TimingFilter.class);
    
    @Override
    public Result invoke(Invoker<?> invoker, 
                         Invocation invocation) 
        throws RpcException {
        
        long startTime = System.currentTimeMillis();
        
        try {
            // 调用下一个Filter或Invoker
            Result result = invoker.invoke(invocation);
            
            // 记录成功调用
            long duration = System.currentTimeMillis() 
                - startTime;
            logger.info("Method {} invoked in {}ms with result: {}", 
                       invocation.getMethodName(), 
                       duration, 
                       result);
            
            return result;
            
        } catch (RpcException e) {
            // 记录失败调用
            long duration = System.currentTimeMillis() 
                - startTime;
            logger.error("Method {} invoked in {}ms, failed with: {}", 
                        invocation.getMethodName(), 
                        duration, 
                        e.getMessage(), 
                        e);
            
            throw e;
        }
    }
}

// 配置文件: META-INF/dubbo/org.apache.dubbo.rpc.Filter
// timing=com.example.TimingFilter
```

### 4. 自定义LoadBalance示例

```java
/**
 * 自定义负载均衡：权重随机
 * 根据服务权重进行随机选择
 */
public class WeightedLoadBalance implements LoadBalance {
    
    @Override
    public <T> Invoker<T> select(List<Invoker<T>> invokers, 
                                  URL url, 
                                  Invocation invocation) {
        
        // 1. 计算总权重
        int totalWeight = 0;
        for (Invoker<T> invoker : invokers) {
            int weight = getWeight(invoker.getUrl());
            totalWeight += weight;
        }
        
        // 2. 生成随机数
        int random = new Random()
            .nextInt(totalWeight);
        
        // 3. 轮询找到对应权重的invoker
        int current = 0;
        for (Invoker<T> invoker : invokers) {
            int weight = getWeight(invoker.getUrl());
            current += weight;
            
            if (random <= current) {
                return invoker;
            }
        }
        
        // 4. 默认返回第一个
        return invokers.get(0);
    }
    
    private int getWeight(URL url) {
        int weight = url.getParameter("weight", 100);
        return weight < 0 ? 100 : weight;
    }
}

// 配置文件: META-INF/dubbo/org.apache.dubbo.rpc.cluster.LoadBalance
// weighted=com.example.WeightedLoadBalance

// 使用:
// <dubbo:reference interface="com.example.DemoService"
//                   loadbalance="weighted"/>
```

### 5. 自定义Router示例

```java
/**
 * 自定义路由：基于地区的就近路由
 */
@Activate(group = "consumer")
public class RegionRouter implements Router {
    
    private URL url;
    
    @Override
    public <T> List<Invoker<T>> route(List<Invoker<T>> invokers, 
                                       URL url, 
                                       Invocation invocation) {
        
        // 1. 获取消费者地区信息
        String consumerRegion = url
            .getParameter("region", "default");
        
        // 2. 优先选择同地区的提供者
        List<Invoker<T>> sameRegion = new ArrayList<>();
        List<Invoker<T>> otherRegion = new ArrayList<>();
        
        for (Invoker<T> invoker : invokers) {
            String providerRegion = 
                invoker.getUrl()
                   .getParameter("region", "default");
            
            if (consumerRegion.equals(providerRegion)) {
                sameRegion.add(invoker);
            } else {
                otherRegion.add(invoker);
            }
        }
        
        // 3. 返回优先级顺序
        if (!sameRegion.isEmpty()) {
            return sameRegion;
        } else {
            return otherRegion;
        }
    }
}
```

---

## 常见扩展点

### SPI扩展点一览表

| SPI接口 | 配置文件位置 | 常见实现 | 使用场景 |
|---------|------------|--------|--------|
| Protocol | META-INF/dubbo/internal/org.apache.dubbo.rpc.Protocol | triple, dubbo, rest, injvm | 选择通信协议 |
| Cluster | META-INF/dubbo/internal/org.apache.dubbo.rpc.cluster.Cluster | failover, failfast, failsafe, failback, forking | 选择容错策略 |
| LoadBalance | META-INF/dubbo/internal/org.apache.dubbo.rpc.cluster.LoadBalance | random, roundrobin, leastactive, consistent_hash | 选择负载均衡 |
| Router | META-INF/dubbo/internal/org.apache.dubbo.rpc.cluster.Router | condition, tag, service, app | 选择路由规则 |
| Registry | META-INF/dubbo/internal/org.apache.dubbo.registry.Registry | zookeeper, nacos, multicast, file | 选择注册中心 |
| Serialization | META-INF/dubbo/internal/org.apache.dubbo.common.serialize.Serialization | hessian2, fastjson2, protobuf | 选择序列化方式 |
| Filter | META-INF/dubbo/internal/org.apache.dubbo.rpc.Filter | generic, context, timeout, validation | 添加拦截逻辑 |
| Invoker | META-INF/dubbo/internal/org.apache.dubbo.rpc.Invoker | - | 自定义调用执行 |
| ProxyFactory | META-INF/dubbo/internal/org.apache.dubbo.rpc.ProxyFactory | javassist, bytebuddy | 选择代理生成方式 |
| ConfigCenter | META-INF/dubbo/internal/org.apache.dubbo.configcenter.ConfigCenter | apollo, nacos, zookeeper, file | 选择配置中心 |

---

## 性能优化点

### 1. 连接复用

```java
// Dubbo会复用连接，不是每次调用都创建新连接

// 配置最连接数
<dubbo:reference connections="10"/>

// 或在URL中
dubbo://host:port/service?connections=10
```

### 2. 线程池隔离

```java
// Provider线程池配置
<dubbo:protocol name="dubbo" 
                 threadpool="fixed" 
                 threads="100" 
                 queues="1000"/>

// Consumer线程池配置
<dubbo:reference threadpool="fixed" threads="100"/>
```

### 3. 异步调用

```java
// 启用异步调用，提高吞吐量
@DubboReference(async = true)
private DemoService demoService;

public void callAsync() {
    // 非阻塞调用
    demoService.sayHello("World");
    
    // 获取结果（可选）
    CompletableFuture<String> future = 
        RpcContext.getServiceContext()
           .getCompletableFuture();
    
    future.thenAccept(result -> {
        System.out.println("Async result: " + result);
    });
}
```

### 4. 参数缓存

```java
// 对于只读服务，可以启用缓存
<dubbo:reference interface="com.example.DemoService"
                  cache="lru"
                  cacheObjects="100"/>
```

### 5. 流式调用

```java
// Triple支持流式调用，减少往返次数

// 服务端：返回流
@Service
public class StreamService implements IStreamService {
    @Override
    public void sayHelloServerStream(
        String name, 
        StreamObserver<String> responseObserver) {
        
        // 分批发送数据流
        for (int i = 0; i < 10; i++) {
            responseObserver.onNext("Hello " + i);
        }
        responseObserver.onCompleted();
    }
}

// 消费端：接收流
@Component
public class StreamConsumer {
    @DubboReference
    private IStreamService streamService;
    
    public void callServerStream() {
        streamService.sayHelloServerStream(
            "World",
            new StreamObserver<String>() {
                @Override
                public void onNext(String value) {
                    System.out.println(value);
                }
                
                @Override
                public void onCompleted() {
                    System.out.println("Stream completed");
                }
                
                @Override
                public void onError(Throwable t) {
                    t.printStackTrace();
                }
            });
    }
}
```

---

## 故障排查

### 1. 常见问题诊断

#### 问题：服务找不到 (No provider available)

**可能原因：**
1. Provider没有启动或注册失败
2. Consumer订阅时Provider没有可用实例
3. 网络问题导致注册中心无法连接

**排查步骤：**
```java
// 1. 查看日志
// 日志中搜索: "subscribe" 和 "notify"

// 2. 检查Provider状态
// 访问Provider控制台或查看进程

// 3. 检查注册中心
// 登录Nacos/Zookeeper查看是否有Provider注册

// 4. 启用Dubbo QoS命令
// http://localhost:22222/api/services
// 查看已注册的服务

// 5. 增加日志级别
// logging.level.org.apache.dubbo = DEBUG
```

#### 问题：调用超时 (RpcException: Timeout)

**可能原因：**
1. Provider响应慢
2. 网络延迟
3. 超时时间设置太短
4. Provider处理请求阻塞

**排查步骤：**
```java
// 1. 增加超时时间
@DubboReference(timeout = 10000)  // 10秒

// 2. 查看Provider日志，看是否阻塞

// 3. 监控网络延迟

// 4. 增加Provider线程数
<dubbo:protocol name="dubbo" threads="200"/>

// 5. 检测是否存在死锁
```

#### 问题：序列化错误 (SerializationException)

**可能原因：**
1. 对象不可序列化
2. 序列化器不匹配
3. 类版本不一致

**排查步骤：**
```java
// 1. 实现Serializable接口
public class DemoData implements Serializable {
    private static final long serialVersionUID = 1L;
}

// 2. 检查序列化ID是否一致
// Provider和Consumer的serialVersionUID必须相同

// 3. 输出序列化错误堆栈跟踪
// logging.level.org.apache.dubbo.common.serialize = DEBUG
```

### 2. 监控和诊断

#### 使用Dubbo QoS

```bash
# Dubbo QoS是质量管理系统，提供命令接口
# 默认端口: 22222

# 查看所有服务
curl http://localhost:22222/api/services

# 查看指定服务的细节
curl http://localhost:22222/api/serviceDetail

# 查看invokers
curl http://localhost:22222/api/invokers

# 获取运行时配置
curl http://localhost:22222/api/configs

# 在线查询
telnet localhost 22222
> ls
> invoke
```

#### 使用Dubbo Admin控制台

```
Dubbo Admin是可视化监控工具

功能：
1. 查看注册的服务
2. 查看服务提供者和消费者
3. 查看接口级别的调用统计
4. 管理服务路由和配置规则
5. 灰度发布支持
```

#### 利用Metrics监控

```yaml
# 启用Prometheus导出
dubbo:
  metrics:
    enable: true
    export-url: prometheus://0.0.0.0:9090
    
# 访问 http://localhost:9090/metrics
# 查看 dubbo_* 开头的指标
```

### 3. 常用诊断命令

```java
// 1. 打印调用链
RpcContext context = RpcContext.getServiceContext();
System.out.println("Remote Host: " + 
    context.getRemoteHost());
System.out.println("Remote Port: " + 
    context.getRemotePort());
System.out.println("Local Host: " + 
    context.getLocalHost());
System.out.println("Local Port: " + 
    context.getLocalPort());

// 2. 查看invocation附件
Invocation invocation = context.getInvocation();
Map<String, Object> attachments = 
    invocation.getAttachments();

// 3. 添加自定义附件
context.setAttachment("key", "value");

// 4. 获取提供者信息
String url = context.getRemoteApplicationName();
```

---

**文档完毕** ✓

这份文档提供了：
1. **核心接口** - 理解Dubbo的基础
2. **关键实现类** - 了解底层原理
3. **实用代码示例** - 快速上手
4. **扩展点** - 自定义功能
5. **性能优化** - 提升应用性能
6. **故障排查** - 解决常见问题
