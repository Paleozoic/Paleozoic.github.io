---
title: Dubbo之基于“版本”的服务调度(路由规则)
date: 2017-06-14 14:46:08
categories: [RPC,Dubbo]
tags: [RPC,Dubbo]
---
<Excerpt in index | 首页摘要>
通过扩展dubbo的路由规则，实现通过不同入参调用不同的服务实例。<!-- more -->
<The rest of contents | 余下全文>

# 写在前面
dubbo.xsd完全没有router的配置，what the fuck！
注解也没关相关配置！！
官方文档如下：
```
<dubbo:protocol router="xxx" />
<dubbo:provider router="xxx" /> <!-- 缺省值设置，当<dubbo:protocol>没有配置loadbalance时，使用此配置 -->
```
走读源码：
首先dubbo APP启动时，便会在`RegistryDirectory`初始化时，针对每个消费者加载路由规则（不过看源码routers传入的都是null）。
并且每当注册中心相关数据有改变时会调用`public synchronized void notify(List<URL> urls)`方法，从注册中心同步消费者路由规则。
内部会调用`protected void setRouters(List<Router> routers)`来设置路由规则，且会append MockInvokersSelector路由器：`routers.add(new MockInvokersSelector());`
即dubbo目前的路由规则是通过注册中心将表达式等推送到客户端（其实还有脚本、文件等方式，此处不展开讲）

# 遇到问题
- 不修改消费者代码可以用新的提供者
- 不修改原来的提供者
- 通过增加数据库配置：key-rest url的KV对实现指定调度

# 基于REST的解决方案
这种方案本质上和dubbo没啥关系了。我大致画个架构图出来，就可以很明显看到其实现。

DB里面提供类似如下的配置（结合nginx）：

| Key  | Nginx                   | Real IP          |
| ---- | ----------------------- | ---------------- |
| P1   | 192.168.1.1:8080/P1_ng/ | 192.168.2.1:8080 |
|      |                         | 192.168.2.2:8080 |
| P2   | 192.168.1.1:8080/P2_ng/ | 192.168.2.3:8080 |
|      |                         | 192.168.2.4:8080 |

代码里面根据Key的不同，通过HTTP REST去调用nginx，然后nginx分发到不同IP下的实例。

架构图（图中并未画出多个nginx分流的情况，自行脑补之）：

![REST_nginx_router](/resources/img/rpc/REST_nginx_router.png)





## 缺点
- 需要指定IP，而IP并不是恒定不变的（虽然生产环境很少改变，但是dev/st/uat就不一定了）
- 负载均衡：需要通过nginx实现服务端的负载均衡
- nginx HA：需要引入nginx HA方案
- 分流：通过多个nginx实例分流减少IO压力

# 基于扩展dubbo路由规则的解决方案
dubbo通过group，interface，version，三者决定是不是同一个服务。group暂不作考虑（目前没用到，那么整个服务注册便是一颗无根树）。**配置version="*"可以获取所有版本的提供者实例**
针对上面的需求，有3个方案：
- 方案一：使用一个版本号，然后dubbo拿到该version的所有Invoker，然后通过提供者、消费者的IP进行匹配。
  这样子便解决了上面除IP之外的所有问题。并且不需要修改dubbo源码。
- 方案二：修改dubbo原生路由规则，让其支持基于版本号的路由设定。
- 方案三：通过配置version="*"，让dubbo可以通过某个注解获取所有的版本的所有Invoker。然后再扩展路由过滤。

方案一只需要研究dubbo的表达式怎么写就可以了；

方案二需要扩展路由，而具体可以参考方案三的实现；

方案三我会具体介绍，基本上包含了方案二的全部实现，但是注意方案二是通过注册中心推送规则。而方案三是通过注解注入相关规则。从而导致方案三的缺点：**更新配置时需要更新所有实例的内存数据**

解决方案如下：（这些只是顺便一提，不展开讲了）

- 配置所有实例共享，显然需要跨进程缓存：Redis、Zookeeper之类的。还可以利用watch实时更新
- 更新配置时，调用所有节点的方法更新配置数据

## dubbo容错调度
![dubbo_fault_tolerance](/resources/img/rpc/dubbo_fault_tolerance.jpg)
- Invoker：这里的Invoker是Provider的一个可调用Service的抽象，Invoker封装了Provider地址及Service接口信息。
- Directory：(SPI, Prototype, ThreadSafe)集群目录服务，[Directory service](https://en.wikipedia.org/wiki/Directory_service)。Directory代表多个Invoker，可以把它看成List<Invoker>，但与List不同的是，它的值可能是动态变化的，比如注册中心推送变更。
- Cluster：Cluster将Directory中的多个Invoker伪装成一个Invoker，对上层透明，伪装过程包含了容错逻辑，调用失败后，重试另一个。
- Router：Router负责从多个Invoker中按路由规则选出子集，比如读写分离，应用隔离等。
- LoadBalance：LoadBalance负责从多个Invoker中选出具体的一个用于本次调用，选的过程包含了负载均衡算法，调用失败后，需要重选。
- 关于Router，有以下应用：
> 服务路由是治理的核心功能，它可以动态调整集群间的访问关系：
  * 排除预发布机：`=> host != 172.22.3.91`
  * 白名单：(注意：一个服务只能有一条白名单规则，否则两条规则交叉，就都被筛选掉了)，`host != 10.20.153.10,10.20.153.11 =>`
  * 黑名单：`host = 10.20.153.10,10.20.153.11 =>`
  * 服务寄宿在应用上，只暴露一部分的机器，防止整个集群挂掉：`=> host = 172.22.3.1*,172.22.3.2*`
  * 为重要应用提供额外的机器：`application != kylin => host != 172.22.3.95,172.22.3.96`
  * 读写分离：`method != find*,list*,get*,is* => host = 172.22.3.97,172.22.3.98` or `method = find*,list*,get*,is* => host = 172.22.3.94,172.22.3.95,172.22.3.96`
  * 前后台分离：`application = bops => host = 172.22.3.91,172.22.3.92,172.22.3.93` or `application != bops => host = 172.22.3.94,172.22.3.95,172.22.3.96`
  * 隔离不同机房网段：`host != 172.22.3.* => host != 172.22.3.*`
  * 提供者与消费者部署在同集群内，本机只访问本机的服务：`=> host = $host`
      而路由规则的配置通常是通过监控中心or治理中心的页面完成，也可以通过`RegistryFactory`写入。

## dubbo源码解析（可以不看）

**以下可以概括为源码乱读= =||**

### ReferenceConfig
```java
//注意urls.add(url.addParameterAndEncoded(Constants.REFER_KEY, StringUtils.toQueryString(map)));这行代码，根据map的参数生成字符串。isInjvm同进程调用。
//所以在map里面添加注解配置的routers参数
private T createProxy(Map<String, String> map) {
		URL tmpUrl = new URL("temp", "localhost", 0, map);
		final boolean isJvmRefer;
        if (isInjvm() == null) {
            if (url != null && url.length() > 0) { //指定URL的情况下，不做本地引用
                isJvmRefer = false;
            } else if (InjvmProtocol.getInjvmProtocol().isInjvmRefer(tmpUrl)) {
                //默认情况下如果本地有服务暴露，则引用本地服务.
                isJvmRefer = true;
            } else {
                isJvmRefer = false;
            }
        } else {
            isJvmRefer = isInjvm().booleanValue();
        }

		if (isJvmRefer) {
			URL url = new URL(Constants.LOCAL_PROTOCOL, NetUtils.LOCALHOST, 0, interfaceClass.getName()).addParameters(map);
			invoker = refprotocol.refer(interfaceClass, url);
            if (logger.isInfoEnabled()) {
                logger.info("Using injvm service " + interfaceClass.getName());
            }
		} else {
            if (url != null && url.length() > 0) { // 用户指定URL，指定的URL可能是对点对直连地址，也可能是注册中心URL
                String[] us = Constants.SEMICOLON_SPLIT_PATTERN.split(url);
                if (us != null && us.length > 0) {
                    for (String u : us) {
                        URL url = URL.valueOf(u);
                        if (url.getPath() == null || url.getPath().length() == 0) {
                            url = url.setPath(interfaceName);
                        }
                        if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                            urls.add(url.addParameterAndEncoded(Constants.REFER_KEY, StringUtils.toQueryString(map)));
                        } else {
                            urls.add(ClusterUtils.mergeUrl(url, map));
                        }
                    }
                }
            } else { // 通过注册中心配置拼装URL
            	List<URL> us = loadRegistries(false);
            	if (us != null && us.size() > 0) {
                	for (URL u : us) {
                	    URL monitorUrl = loadMonitor(u);
                        if (monitorUrl != null) {
                            map.put(Constants.MONITOR_KEY, URL.encode(monitorUrl.toFullString()));
                        }
                	    urls.add(u.addParameterAndEncoded(Constants.REFER_KEY, StringUtils.toQueryString(map)));
                    }
            	}
            	if (urls == null || urls.size() == 0) {
                    throw new IllegalStateException("No such any registry to reference " + interfaceName  + " on the consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion() + ", please config <dubbo:registry address=\"...\" /> to your spring config.");
                }
            }

            if (urls.size() == 1) {
                invoker = refprotocol.refer(interfaceClass, urls.get(0));
            } else {
                List<Invoker<?>> invokers = new ArrayList<Invoker<?>>();
                URL registryURL = null;
                for (URL url : urls) {
                    invokers.add(refprotocol.refer(interfaceClass, url));
                    if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                        registryURL = url; // 用了最后一个registry url
                    }
                }
                if (registryURL != null) { // 有 注册中心协议的URL
                    // 对有注册中心的Cluster 只用 AvailableCluster
                    URL u = registryURL.addParameter(Constants.CLUSTER_KEY, AvailableCluster.NAME);
                    invoker = cluster.join(new StaticDirectory(u, invokers));
                }  else { // 不是 注册中心的URL
                    invoker = cluster.join(new StaticDirectory(invokers));
                }
            }
        }

        Boolean c = check;
        if (c == null && consumer != null) {
            c = consumer.isCheck();
        }
        if (c == null) {
            c = true; // default true
        }
        if (c && ! invoker.isAvailable()) {
            throw new IllegalStateException("Failed to check the status of the service " + interfaceName + ". No provider available for the service " + (group == null ? "" : group + "/") + interfaceName + (version == null ? "" : ":" + version) + " from the url " + invoker.getUrl() + " to the consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion());
        }
        if (logger.isInfoEnabled()) {
            logger.info("Refer dubbo service " + interfaceClass.getName() + " from url " + invoker.getUrl());
        }
        // 创建服务代理
        return (T) proxyFactory.getProxy(invoker);
    }
```    
留意：init()的以下代码：
```java
checkDefault(); //初始化默认的ConsumerConfig
appendProperties(this);//追加当前配置到ConsumerConfig
```
但是protected static void appendProperties(AbstractConfig config)是AbstractConfig的方法，入参也是AbstractConfig里面，所以appendProperties不能追加ReferenceConfig的参数。
```java
private void checkDefault() {
		if (consumer == null) {
				consumer = new ConsumerConfig();
		}
		//追加AbstractConfig的参数
		appendProperties(consumer);
}
```

### AnnotationBean

`AnnotationBean extends AbstractConfig`用于包装处理`com.alibaba.dubbo.config.annotation.Reference`和`com.alibaba.dubbo.config.annotation.Service`注解。

通过`AnnotationBean`的方法`private Object refer(Reference reference, Class<?> referenceClass)`解析`AnnotationBean`成`private final ConcurrentMap<String, ReferenceBean<?>> referenceConfigs = new ConcurrentHashMap<String, ReferenceBean<?>>();`

在`refer`方法中，对consumer的解析执行了2次（此处甚是迷惑）：

```java
private Object refer(Reference reference, Class<?> referenceClass) { //method.getParameterTypes()[0]
    String interfaceName;
    if (! "".equals(reference.interfaceName())) {
        interfaceName = reference.interfaceName();
    } else if (! void.class.equals(reference.interfaceClass())) {
        interfaceName = reference.interfaceClass().getName();
    } else if (referenceClass.isInterface()) {
        interfaceName = referenceClass.getName();
    } else {
        throw new IllegalStateException("The @Reference undefined interfaceClass or interfaceName, and the property type " + referenceClass.getName() + " is not a interface.");
    }
    String key = reference.group() + "/" + interfaceName + ":" + reference.version();
    ReferenceBean<?> referenceConfig = referenceConfigs.get(key);
    if (referenceConfig == null) {
        referenceConfig = new ReferenceBean<Object>(reference);
        if (void.class.equals(reference.interfaceClass())
                && "".equals(reference.interfaceName())
                && referenceClass.isInterface()) {
            referenceConfig.setInterface(referenceClass);
        }
        if (applicationContext != null) {
            referenceConfig.setApplicationContext(applicationContext);
            if (reference.registry() != null && reference.registry().length > 0) {
                List<RegistryConfig> registryConfigs = new ArrayList<RegistryConfig>();
                for (String registryId : reference.registry()) {
                    if (registryId != null && registryId.length() > 0) {
                        registryConfigs.add((RegistryConfig)applicationContext.getBean(registryId, RegistryConfig.class));
                    }
                }
                referenceConfig.setRegistries(registryConfigs);
            }
            if (reference.consumer() != null && reference.consumer().length() > 0) { //
                referenceConfig.setConsumer((ConsumerConfig)applicationContext.getBean(reference.consumer(), ConsumerConfig.class));
            }
            if (reference.monitor() != null && reference.monitor().length() > 0) {
                referenceConfig.setMonitor((MonitorConfig)applicationContext.getBean(reference.monitor(), MonitorConfig.class));
            }
            if (reference.application() != null && reference.application().length() > 0) {
                referenceConfig.setApplication((ApplicationConfig)applicationContext.getBean(reference.application(), ApplicationConfig.class));
            }
            if (reference.module() != null && reference.module().length() > 0) {
                referenceConfig.setModule((ModuleConfig)applicationContext.getBean(reference.module(), ModuleConfig.class));
            }
            if (reference.consumer() != null && reference.consumer().length() > 0) {
                referenceConfig.setConsumer((ConsumerConfig)applicationContext.getBean(reference.consumer(), ConsumerConfig.class));
            }
            try {
                referenceConfig.afterPropertiesSet();
            } catch (RuntimeException e) {
                throw (RuntimeException) e;
            } catch (Exception e) {
                throw new IllegalStateException(e.getMessage(), e);
            }
        }
        referenceConfigs.putIfAbsent(key, referenceConfig);
        referenceConfig = referenceConfigs.get(key);
    }
    return referenceConfig.get();
}
```

### AbstractDirectory

AbstractDirectory是增加router的Directory。
[com.alibaba.dubbo.rpc.cluster.directory.AbstractDirectory](https://github.com/alibaba/dubbo/blob/master/dubbo-cluster/src/main/java/com/alibaba/dubbo/rpc/cluster/directory/AbstractDirectory.java)
```java
//获取所有的Invoker，并遍历路由规则过滤求Invokers子集
public List<Invoker，并根据路由规则遍历过滤<T>> list(Invocation invocation) throws RpcException {
    if (destroyed){
        throw new RpcException("Directory already destroyed .url: "+ getUrl());
    }
    List<Invoker<T>> invokers = doList(invocation);
    List<Router> localRouters = this.routers; // local reference
    if (localRouters != null && localRouters.size() > 0) {
        for (Router router: localRouters){
            try {
                if (router.getUrl() == null || router.getUrl().getParameter(Constants.RUNTIME_KEY, true)) {
                    invokers = router.route(invokers, getConsumerUrl(), invocation);
                }
            } catch (Throwable t) {
                logger.error("Failed to execute router: " + getUrl() + ", cause: " + t.getMessage(), t);
            }
        }
    }
    return invokers;
}
```

### RegistryDirectory

RegistryDirectory: 注册目录服务，通过此类可以获取消费者消费的提供者的所有实例Invokers
[com.alibaba.dubbo.registry.integration.RegistryDirectory](https://github.com/alibaba/dubbo/blob/master/dubbo-registry/dubbo-registry-api/src/main/java/com/alibaba/dubbo/registry/integration/RegistryDirectory.java)

```java
//从注册中心获取消费者消费的提供者的所有实例Invokers
public List<Invoker<T>> doList(Invocation invocation) {
    if (forbidden) {
        throw new RpcException(RpcException.FORBIDDEN_EXCEPTION, "Forbid consumer " +  NetUtils.getLocalHost() + " access service " + getInterface().getName() + " from registry " + getUrl().getAddress() + " use dubbo version " + Version.getVersion() + ", Please check registry access list (whitelist/blacklist).");
    }
    List<Invoker<T>> invokers = null;
    Map<String, List<Invoker<T>>> localMethodInvokerMap = this.methodInvokerMap; // local reference
    if (localMethodInvokerMap != null && localMethodInvokerMap.size() > 0) {
        String methodName = RpcUtils.getMethodName(invocation);
        Object[] args = RpcUtils.getArguments(invocation);
        if(args != null && args.length > 0 && args[0] != null
                && (args[0] instanceof String || args[0].getClass().isEnum())) {
            invokers = localMethodInvokerMap.get(methodName + "." + args[0]); // 可根据第一个参数枚举路由
        }
        if(invokers == null) {
            invokers = localMethodInvokerMap.get(methodName);
        }
        if(invokers == null) {
            invokers = localMethodInvokerMap.get(Constants.ANY_VALUE);
        }
        if(invokers == null) {
            Iterator<List<Invoker<T>>> iterator = localMethodInvokerMap.values().iterator();
            if (iterator.hasNext()) {
                invokers = iterator.next();
            }
        }
    }
    return invokers == null ? new ArrayList<Invoker<T>>(0) : invokers;
}
```

### ConditionRouter

ConditionRouter：条件路由，关于条件路由的定义以及正则匹配，具体见[这里](http://dubbo.io/User+Guide-zh.htm#UserGuide-zh-%E6%9D%A1%E4%BB%B6%E8%B7%AF%E7%94%B1%E8%A7%84%E5%88%99)，源码是：`Map<String, MatchPair> parseRule(String rule)`
[com.alibaba.dubbo.rpc.cluster.router.condition.ConditionRouter](https://github.com/alibaba/dubbo/blob/master/dubbo-cluster/src/main/java/com/alibaba/dubbo/rpc/cluster/router/condition/ConditionRouter.java)

```java
/**
  * 路由规则逻辑
  * 1. 必须同时配置 消费端匹配 和 提供端过滤
  * 2. 消费端不匹配 返回所有 invoker
  *
  * @param invokers
  * @param url refer url
  * @param invocation
  * @param <T>
  * @return
  * @throws RpcException
  */
 public <T> List<Invoker<T>> route(List<Invoker<T>> invokers, URL url, Invocation invocation)
         throws RpcException {
     if (invokers == null || invokers.size() == 0) {
         return invokers;
     }
     try {
         //消费端匹配 如果不匹配 直接返回所有invoker
         if (!matchWhen(url)) {
             return invokers;
         }
         List<Invoker<T>> result = new ArrayList<Invoker<T>>();
         if (!hasThenCondition()) {
             logger.warn("The current consumer in the service blacklist. consumer: " + NetUtils.getLocalHost() + ", service: " + url.getServiceKey());
             return result;
         }
         for (Invoker<T> invoker : invokers) {
             if (matchThen(invoker.getUrl(), url)) {
                 result.add(invoker);
             }
         }
         if (result.size() > 0) {
             return result;
         } else if (force) {
             logger.warn("The route result is empty and force execute. consumer: " + NetUtils.getLocalHost() + ", service: " + url.getServiceKey() + ", router: " + url.getParameterAndDecoded(Constants.RULE_KEY));
             return result;
         }
     } catch (Throwable t) {
         logger.error("Failed to execute condition router rule: " + getUrl() + ", invokers: " + invokers + ", cause: " + t.getMessage(), t);
     }
     return invokers;
 }
```

## 修改源码（此步才是关键）

### dubbo运行流程

先理清一下dubbo的运行流程：

- Spring扫描注解得到所有的`AnnotationBean`。即`@Refernce`和`@Service`注解的引用
- 对每一个`AnnotationBean`执行：`private Object refer(Reference reference, Class<?> referenceClass)`，得到`ConcurrentMap<String, ReferenceBean<?>> referenceConfigs`和`Set<ServiceConfig<?>> serviceConfigs = new ConcurrentHashSet<ServiceConfig<?>>()`。这两个都是`AnnotationBean`的属性。（在这里查找对注解的解析代码，层层引用，可以定位到：`com.alibaba.dubbo.config.AbstractConfig#appendAnnotation`）
- 从`ReferenceConfig<T>`调用`get()`方法，会进行`init()`操作。ReferenceConfig会被缓存，所以消费者代理也会被缓存（消费者代理是ReferenceConfig的属性）。里面有下面一段源码（留意我写的注释）：

```java
//attributes是？？？
StaticContext.getSystemContext().putAll(attributes);//attributes通过系统context进行存储.
//map是时间戳、PID、revision、methods等参数，这些参数编码之后作为url的refer参数的值。所以不能将router放在map里面。后面会说routers是如何识别的
ref = createProxy(map); //创建代理
```

- 通过带来调用服务提供者

### 注解配置解析
首先，路由器应该是可配置的，那么就要添加解析路由器的相关代码。
dubbo是客户端负载均衡，理论上路由的判断应放在客户端执行，所以这里我本来打算修改消费者相关的代码。
[com.alibaba.dubbo.config.ReferenceConfig ](https://github.com/dangdangdotcom/dubbox/blob/master/dubbo-config/dubbo-config-api/src/main/java/com/alibaba/dubbo/config/ReferenceConfig.java)的`private void init()`方法初始化并解析消费者的配置。
但注意：如果在`com.alibaba.dubbo.config.AbstractReferenceConfig` 抽象类添加router属性，会有点问题：**（`AbstractInterfaceConfig`属于提供者和消费者共同的配置接口，而且从注册中心加载URL的方法`protected List<URL> loadRegistries(boolean provider)`由它提供，所以扩展路由代码写在`AbstractInterfaceConfig`里面）**：

```java
// 路由
protected String router;

public String getRouter() {
  return router;
}

public void setRouter(String router) {
  this.router = router;
}
```

先修改注解：
增加routers的配置参数：
[com.alibaba.dubbo.config.annotation.Reference]()
```java
String[] routers() default {};
```

注意到：在`ReferenceConfig`的构造方法里会调用`appendAnnotation`来解析注解（注：对于数组合会并成','分隔的字符串）。
```java
public ReferenceConfig(Reference reference) {
    appendAnnotation(Reference.class, reference);
}
```
修改`com.alibaba.dubbo.config.AbstractConfig#appendAnnotation`:
```java
Object value = method.invoke(annotation, new Object[0]);
if (value != null && ! value.equals(method.getDefaultValue())) {
    Class<?> parameterType = ReflectUtils.getBoxedClass(method.getReturnType());
    if ("filter".equals(property) || "listener".equals(property) || "router".equals(property)) { //增加路由处理
        parameterType = String.class;
        value = StringUtils.join((String[]) value, ","); //将数组处理成字符串
    } else if ("parameters".equals(property)) {
        parameterType = Map.class;
        value = CollectionUtils.toStringMap((String[]) value);
    }
    try {
        //此处的setter方法属于AbstractReferenceConfig
        Method setterMethod = getClass().getMethod(setter, new Class<?>[] { parameterType }); //根据参数类型和参数，通过反射调用set方法设置指。
        setterMethod.invoke(this, new Object[] { value }); //调用set方法
    } catch (NoSuchMethodException e) {
        // ignore
    }
}
```

经过了上面的一步，注解的参数已经被解析到`AbstractConfig`里面了。




### router设置

注意：dubbo的router扩展如下(这种方式是JDK SPI)：

- 写2个类，分别实现Router和RouterFactory。RouterFactory假设是`com.alibaba.dubbo.rpc.cluster.router.xxxRouterFactory`
- 然后让dubbo可以识别routerName是xxx的路由。

```
src
 |-main
    |-java
        |-com
            |-xxx
                |-xxxRouterFactory.java (实现RouterFactory接口)
                |-xxxRouter.java (实现Router接口)
    |-resources
        |-META-INF
            |-dubbo
                |-com.alibaba.dubbo.rpc.cluster.RouterFactory (纯文本文件，内容为：xxx=com.alibaba.dubbo.rpc.cluster.router.xxxRouterFactory)
```

- 设置路由，通常在初始化`AbstractDirectory`时设置，或由`RegistryDirectory`的`notify`通知更新。

```java
protected void setRouters(List<Router> routers){
    // copy list
    routers = routers == null ? new  ArrayList<Router>() : new ArrayList<Router>(routers);
    // append url router
    String routerkey = url.getParameter(Constants.ROUTER_KEY); //看是否存在routerName，URL会通过category=routers&router=xxxRouterName&dynamic=false传入router相关配置 注：Constants.ROUTER_KEY="router"
    if (routerkey != null && routerkey.length() > 0) {
        RouterFactory routerFactory = ExtensionLoader.getExtensionLoader(RouterFactory.class).getExtension(routerkey);//getExtensionLoader获取ExtensionLoader然后根据routerName获取RouterFactory实例
        routers.add(routerFactory.getRouter(url));
    }
    // append mock invoker selector
    routers.add(new MockInvokersSelector());
    Collections.sort(routers);
    this.routers = routers;
}
```

那么问题来了，URL是怎么生成的呢？

```java
else { // 通过注册中心配置拼装URL，其他两种情况是本地调用和指定的点对点调用。可不做考虑
  List<URL> us = loadRegistries(false);//URL从注册中心来
  if (us != null && us.size() > 0) {
    for (URL u : us) {
      URL monitorUrl = loadMonitor(u);
      if (monitorUrl != null) {
        map.put(Constants.MONITOR_KEY, URL.encode(monitorUrl.toFullString()));
      }
      urls.add(u.addParameterAndEncoded(Constants.REFER_KEY, StringUtils.toQueryString(map)));
    }
  }
  if (urls == null || urls.size() == 0) {
    throw new IllegalStateException("No such any registry to reference " + interfaceName  + " on the consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion() + ", please config <dubbo:registry address=\"...\" /> to your spring config.");
  }
}
```

到此为止，在URL添加上router即可，如下：

```java
protected List<URL> loadRegistries(boolean provider) {
  checkRegistry();
  List<URL> registryList = new ArrayList<URL>();
  if (registries != null && registries.size() > 0) {
    for (RegistryConfig config : registries) {
      String address = config.getAddress();
      if (address == null || address.length() == 0) {
        address = Constants.ANYHOST_VALUE;
      }
      String sysaddress = System.getProperty("dubbo.registry.address");
      if (sysaddress != null && sysaddress.length() > 0) {
        address = sysaddress;
      }
      if (address != null && address.length() > 0
          && ! RegistryConfig.NO_AVAILABLE.equalsIgnoreCase(address)) {
        Map<String, String> map = new HashMap<String, String>();
        appendParameters(map, application);
        appendParameters(map, config);
        map.put("path", RegistryService.class.getName());
        map.put("dubbo", Version.getVersion());
        map.put(Constants.TIMESTAMP_KEY, String.valueOf(System.currentTimeMillis()));
        if (ConfigUtils.getPid() > 0) {
          map.put(Constants.PID_KEY, String.valueOf(ConfigUtils.getPid()));
        }
        if (! map.containsKey("protocol")) {
          if (ExtensionLoader.getExtensionLoader(RegistryFactory.class).hasExtension("remote")) {
            map.put("protocol", "remote");
          } else {
            map.put("protocol", "dubbo");
          }
        }
        List<URL> urls = UrlUtils.parseURLs(address, map);
        for (URL url : urls) {
          url = url.addParameter(Constants.REGISTRY_KEY, url.getProtocol());
          if(this.router!=null&&this.router.length()>0){//router存在
            url = url.addParameter(Constants.ROUTER_KEY, this.router);//添加路由参数
          }
          url = url.setProtocol(Constants.REGISTRY_PROTOCOL);
          if ((provider && url.getParameter(Constants.REGISTER_KEY, true))
              || (! provider && url.getParameter(Constants.SUBSCRIBE_KEY, true))) {
            registryList.add(url);
          }
        }
      }
    }
  }
  return registryList;
}
```

###  修改的具体代码
[Github查看](https://github.com/Paleozoic/dubbox/commit/c10416a67aafc8fac9665d5100f5ff8dadfaf02f)
PS：一看提交记录，其实才改了几行代码，但是其中需要做的准备工作感觉很多。这也算是Coding的乐趣之一了:)

### 使用示例
[项目示例见Github](https://github.com/Paleozoic/spring_boot_dubbox_demo)
[关键代码](https://github.com/Paleozoic/spring_boot_dubbox_demo/commit/f25ae9ea48bbbf73f5ef942a3914db5390c81345)
- [KeyRouter](https://github.com/Paleozoic/spring_boot_dubbox_demo/tree/master/31-ms-consumer/src/main/java/com/alibaba/dubbo/rpc/cluster/router)：包括工厂类和实例类
- [JDK SPI](https://github.com/Paleozoic/spring_boot_dubbox_demo/tree/master/31-ms-consumer/src/main/resources/META-INF/dubbo)：注意文件名和位置
- 注解引入router：
```java
@Reference(version = "*",router = "keyRouter")
private HelloRest helloRest3;
```

## 关于XML配置文件

这里并没有扩展XML配置文件，有兴趣可以自行扩展。或许有一天我会补上这儿的代码。

其实原理上大体差不多。区别应该是初始化的时候读的是XML。

那么关注的类是：

- `NamespaceHandler`
- `BeandefinitionParser`
- `DubboBeanDefinitionParser`
- `DubboNamespaceHandler`



## TODO

从注册中心推送的routers是否会影响现有的routers？覆盖/被覆盖/叠加/抛异常？待研究。


# 写在最后
dubbo的确是优秀的框架，一开始我的设计是重写负载均衡器，然后走读代码+官方文档，才发现dubbo早已考虑到了这些需求。
dubbo提供了router层，稍作变动问题便可以迎刃而解。
很感谢阿里的开源。：)

# 引用
[Dubbo用户指南](http://dubbo.io/User+Guide-zh.htm)
[Dubbo开发者指南](http://dubbo.io/Developer+Guide-zh.htm)
