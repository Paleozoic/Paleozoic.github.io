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
目前卡住了，dubbo.xsd完全没有router的配置，what the fuck！
官方文档如下：
```
<dubbo:protocol router="xxx" />
<dubbo:provider router="xxx" /> <!-- 缺省值设置，当<dubbo:protocol>没有配置loadbalance时，使用此配置 -->
```

# 遇到问题
- 不修改消费者代码可以用新的提供者
- 不修改原来的提供者
- 通过增加数据库配置：key-rest url的KV对实现指定调度

# 基于REST的解决方案
##

## 缺点
- 需要指定IP，而IP并不是恒定不变的（虽然生产环境很少改变，但是dev/st/uat就不一定了）
- 负载均衡：需要通过nginx实现服务端的负载均衡
- nginx HA：需要引入nginx HA方案
- 分流：通过多个nginx实例分流减少IO压力

# 扩展dubbo路由规则
dubbo通过group，interface，version，三者决定是不是同一个服务。group暂不作考虑（目前没用到，那么整个服务注册便是一颗无根树）。**配置version="*"可以获取所有版本的提供者实例**
针对上面的需求，有2个方案：
- 方案一：使用一个版本号，然后dubbo拿到该version的所有Invoker，然后通过提供者、消费者的IP进行匹配。
这样子便解决了上面除IP之外的所有问题。并且不需要修改dubbo源码。
- 方案二：通过配置version="*"，让dubbo可以通过某个注解获取所有的版本的所有Invoker。然后再扩展路由过滤。

下面提供方案二的解决方案，事实上方案一可以说是属于方案二的一部分。能写方案二，方案一也是信手拈来的。
当然，走读一下dubbo源码是必不可少的。

## dubbo容错调度
![dubbo_fault_tolerance](/resources/img/rpc/dubbo_fault_tolerance.jpg)
- Invoker：这里的Invoker是Provider的一个可调用Service的抽象，Invoker封装了Provider地址及Service接口信息。
- Directory：(SPI, Prototype, ThreadSafe)集群目录服务，[Directory service](https://en.wikipedia.org/wiki/Directory_service)。Directory代表多个Invoker，可以把它看成List<Invoker>，但与List不同的是，它的值可能是动态变化的，比如注册中心推送变更。
- Cluster：Cluster将Directory中的多个Invoker伪装成一个Invoker，对上层透明，伪装过程包含了容错逻辑，调用失败后，重试另一个。
- Router：Router负责从多个Invoker中按路由规则选出子集，比如读写分离，应用隔离等。
- LoadBalance：LoadBalance负责从多个Invoker中选出具体的一个用于本次调用，选的过程包含了负载均衡算法，调用失败后，需要重选。
- 关于Router，[有以下应用](https://groups.google.com/forum/#!searchin/dubbo/router|sort:relevance/dubbo/j0gWIQI9B4g/3DBajDKr-boJ)：
> 服务路由是治理的核心功能，它可以动态调整集群间的访问关系：
  * 比如：按前后台应用分离，前台应用访问1,2,3机器，后台应用访问4,5机器。
  * 比如：a服务被b, c两个应用调用，而b应用很重要，就可以通过路由将a中的某几台机器只允许b访问，其它共享，这样如果c把其它机器都拖跨了，b不会受影响。
  * 比如：按读写分离find*访问1,2,3机器，update*访问4,5机器。
  * 比如：排除集群中的某台机器作为预发布机。

## dubbo源码解析
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

# 写在最后
dubbo的确是优秀的框架，一开始我的设计是重写负载均衡器，然后走读代码+官方文档，才发现dubbo早已考虑到了这些需求。
dubbo提供了router层，稍作变动问题便可以迎刃而解。
很感谢阿里的开源。：)

# 引用
[Dubbo用户指南](http://dubbo.io/User+Guide-zh.htm)
[Dubbo开发者指南](http://dubbo.io/Developer+Guide-zh.htm)
