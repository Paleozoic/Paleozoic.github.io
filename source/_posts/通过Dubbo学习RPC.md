---
title: 通过Dubbo学习RPC
date: 2017-05-09 23:00:17
categories: [RPC,Dubbo]
tags: [RPC,Dubbo]
---
<Excerpt in index | 首页摘要>
RPC基础，通过Dubbo的设计学习RPC框架的基本组成。<!-- more -->
<The rest of contents | 余下全文>

# Dubbo依赖关系
![Dubbo依赖关系](/resources/img/rpc/dubbo依赖关系.jpg)
由dubbo可以看到一个基本的RPC框架设计和依赖。
- Provider向注册中心注册服务(pub)
- Consumer订阅注册中心消息(sub)
- Provider向注册中心注册服务时会被注册中心推送至Consumer
- Consumer通过注册中心获取到服务的注册信息，比如调用地址等。Consumer通过调用地址列表做负载均衡（客户端负载均衡），然后调用Provider。数据之间需要序列化后通过网络传输
- 监控

# 注册中心（服务注册与发现）

# 序列化协议
|协议|优点|缺点|数据格式|可读性|
|--|--|--|--|--|
|Kyro|||||
|Avro|||||
|Xstream|||||
|Hessian|||||
|Jackson|||||
|JDK|||||

# 网络传输
|框架|JDK底层|传输协议|连接方式|优点|缺点|
|--|--|--|--|--|--|
|Netty|NIO|||||
|Mina|NIO|||||
|Grizzly|NIO|||||
|Twisted||||||
|REST类||||||

# 负载均衡
## 服务端负载均衡(nginx/zuul)
由网关统一管理应用请求的分发，好处是服务请求入口统一管理，方便做限流、权限控制等；缺点是所有负载均衡的分发压力（CPU和IO）全部归于网关。
## 客户端负载均衡(dubbo loadbalance/ribbon)
有客户端从配置中心获取服务实例列表，然后客户端根据服务列表做负载均衡的处理。好处是负载均衡的分发压力分摊给客户端，缺点是不方便做请求的统一管理。
## 负载均衡策略
一般来说，负载均衡的算法有3大类：轮询、哈希以及随机。

# 监控

# Dubbo支持协议(序列化以及网络传输)
![dubbo-protocol](/resources/img/rpc/dubbo-protocol.jpg)
dubbo将对象序列化，包括header(codec)（序列化编码方式，可选）和body(serialization)（对象序列化后的内容，二进制或者字符串）。
Client通过网络传输，将序列化内容发送给服务端。

|协议|访问地址|
|--|--|
|dubbo协议|[dubbo协议](http://dubbo.io/User+Guide-zh.htm#UserGuide-zh-dubbo%3A%2F%2F)|
|rmi协议|[rmi协议](http://dubbo.io/User+Guide-zh.htm#UserGuide-zh-rmi%3A%2F%2F)|
|hessian协议|[hessian协议](http://dubbo.io/User+Guide-zh.htm#UserGuide-zh-hessian%3A%2F%2F)|
|http协议|[http协议](http://dubbo.io/User+Guide-zh.htm#UserGuide-zh-http%3A%2F%2F)|
|webservice协议|[webservice协议](http://dubbo.io/User+Guide-zh.htm#UserGuide-zh-webservice%3A%2F%2F)|
|thrift协议|[thrift协议](http://dubbo.io/User+Guide-zh.htm#UserGuide-zh-thrift%3A%2F%2F)|
|memcached协议|[memcached协议](http://dubbo.io/User+Guide-zh.htm#UserGuide-zh-memcached%3A%2F%2F)|
|redis协议|[redis协议](http://dubbo.io/User+Guide-zh.htm#UserGuide-zh-redis%3A%2F%2F)|

# Dubbo服务集群容错
## Failover 失败转移
失败转移，当出现失败，重试其它服务器，通常用于读操作，但重试会带来更长延迟。
以下是源码，只保留关键部分。
[`com.alibaba.dubbo.rpc.cluster.support.FailoverClusterInvoker`](https://github.com/alibaba/dubbo/blob/master/dubbo-cluster/src/main/java/com/alibaba/dubbo/rpc/cluster/support/FailoverClusterInvoker.java)
```java
/**
 * @param invocation Invocation. (API, Prototype, NonThreadSafe) 此对象保存了方法名、参数类型、参数值、Attachment以及当前上下文的Invoker
 * @param invokers Invoker. (API/SPI, Prototype, ThreadSafe) 当前上下文的Invoker。Invoker用于执行Invocation。此处的List<Invoker<T>> invokers是注册中心获取到的所有实例
 * @param loadbalance 负载均衡器，提供select方法选择客户端调用的实例
 * @return
 * @throws RpcException
 */
public Result doInvoke(Invocation invocation, final List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    	List<Invoker<T>> copyinvokers = invokers;
    	checkInvokers(copyinvokers, invocation); //此方法用于检查invokers的大小是否为0，为0抛出异常
        // 通过方法名、RETRIES_KEY、DEFAULT_RETRIES获取重试次数
        int len = getUrl().getMethodParameter(invocation.getMethodName(), Constants.RETRIES_KEY, Constants.DEFAULT_RETRIES) + 1;
        if (len <= 0) { //如果配置了负数，那是不行的。至少也得重试1次。
            len = 1;
        }
        // retry loop.
        RpcException le = null; // last exception.
        List<Invoker<T>> invoked = new ArrayList<Invoker<T>>(copyinvokers.size()); // invoked invokers. 存储已经执行过的Invoker
        Set<String> providers = new HashSet<String>(len); // 存储提供者，其实就是提供者的实例地址，URL
        for (int i = 0; i < len; i++) {
        	//重试时，进行重新选择，避免重试时invoker列表已发生变化.
        	//注意：如果列表发生了变化，那么invoked判断会失效，因为invoker实例已经改变
        	if (i > 0) {
        		checkWhetherDestroyed(); //检查AbstractClusterInvoker的destroyed是否为true
        		copyinvokers = list(invocation); //根据invocation拿到它自己的提供者列表
        		//重新检查一下
        		checkInvokers(copyinvokers, invocation); //检查一下提供者是否存在
        	}
            Invoker<T> invoker = select(loadbalance, invocation, copyinvokers, invoked); //通过负载均衡器选择实例
            invoked.add(invoker);
            RpcContext.getContext().setInvokers((List)invoked); //更新上下文已经被执行过的实例
            try {
                Result result = invoker.invoke(invocation);
                if (le != null && logger.isWarnEnabled()) {
                    // le是上次调用失败保留的异常信息
                }
                return result;
            } catch (RpcException e) {
                if (e.isBiz()) { // biz exception.
                    throw e;
                }
                le = e;
            } catch (Throwable e) {
                le = new RpcException(e.getMessage(), e);
            } finally {
                providers.add(invoker.getUrl().getAddress());
            }
        }
        throw new RpcException(...);
    }
```

## Failfast 快速失败
快速失败，只发起一次调用，失败立即报错，通常用于非幂等性的写操作。
以下是源码，只保留关键部分。
[`com.alibaba.dubbo.rpc.cluster.support.FailfastClusterInvoker`](https://github.com/alibaba/dubbo/blob/master/dubbo-cluster/src/main/java/com/alibaba/dubbo/rpc/cluster/support/FailfastClusterInvoker.java)
```java
public Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    checkInvokers(invokers, invocation);
    Invoker<T> invoker = select(loadbalance, invocation, invokers, null);
    try {
        return invoker.invoke(invocation);
    } catch (Throwable e) {
        if (e instanceof RpcException && ((RpcException)e).isBiz()) { // biz exception.
            throw (RpcException) e;
        }
        throw new RpcException(...); //调用失败，直接抛出RpcException
    }
}
```

## Failsafe 失败安全
失败安全，出现异常时，直接忽略，通常用于写入审计日志等操作。
以下是源码，只保留关键部分。
[`com.alibaba.dubbo.rpc.cluster.support.FailfastClusterInvoker`](https://github.com/alibaba/dubbo/blob/master/dubbo-cluster/src/main/java/com/alibaba/dubbo/rpc/cluster/support/FailfastClusterInvoker.java)
```java
public Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    try {
        checkInvokers(invokers, invocation);
        Invoker<T> invoker = select(loadbalance, invocation, invokers, null);
        return invoker.invoke(invocation);
    } catch (Throwable e) {
        logger.error("Failsafe ignore exception: " + e.getMessage(), e);
        return new RpcResult(); // ignore
    }
}
```

## Failback 失败自动恢复
失败自动恢复，后台记录失败请求，定时重发，通常用于消息通知操作。
以下是源码，只保留关键部分。
[`com.alibaba.dubbo.rpc.cluster.support.FailbackClusterInvoker`](https://github.com/alibaba/dubbo/blob/master/dubbo-cluster/src/main/java/com/alibaba/dubbo/rpc/cluster/support/FailbackClusterInvoker.java)
```java
protected Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    try {
        checkInvokers(invokers, invocation);
        Invoker<T> invoker = select(loadbalance, invocation, invokers, null);
        return invoker.invoke(invocation);
    } catch (Throwable e) {
        addFailed(invocation, this); //调用失败，后台线程定时重发调用
        return new RpcResult(); // ignore
    }
}

private void addFailed(Invocation invocation, AbstractClusterInvoker<?> router) {
    if (retryFuture == null) {
        synchronized (this) {
            if (retryFuture == null) {
                retryFuture = scheduledExecutorService.scheduleWithFixedDelay(new Runnable() {

                    public void run() {
                        // 收集统计信息
                        try {
                            retryFailed();
                        } catch (Throwable t) { // 防御性容错
                            logger.error("Unexpected error occur at collect statistic", t);
                        }
                    }
                }, RETRY_FAILED_PERIOD, RETRY_FAILED_PERIOD, TimeUnit.MILLISECONDS);
            }
        }
    }
    failed.put(invocation, router);
}
```

## Forking 并行调用
并行调用，只要一个成功即返回，全部异常则返回最后一个异常。通常用于实时性要求较高的操作，但需要浪费更多服务资源。
以下是源码，只保留关键部分。
[`com.alibaba.dubbo.rpc.cluster.support.ForkingClusterInvoker`](https://github.com/alibaba/dubbo/blob/master/dubbo-cluster/src/main/java/com/alibaba/dubbo/rpc/cluster/support/ForkingClusterInvoker.java)
```java
public Result doInvoke(final Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    checkInvokers(invokers, invocation);
    final List<Invoker<T>> selected;
    final int forks = getUrl().getParameter(Constants.FORKS_KEY, Constants.DEFAULT_FORKS);
    final int timeout = getUrl().getParameter(Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
    if (forks <= 0 || forks >= invokers.size()) { //如果配置的并行调用数量小于0或者大于最大提供者实例数量，则设置为最大提供者实例数量
        selected = invokers;
    } else {
        selected = new ArrayList<Invoker<T>>();
        for (int i = 0; i < forks; i++) { //轮询forks次
            //在invoker列表(排除selected)后,如果没有选够,则存在重复循环问题.见select实现.
            Invoker<T> invoker = select(loadbalance, invocation, invokers, selected);
            if(!selected.contains(invoker)){//防止重复添加invoker，因为负载均衡器可能选择到相同的实例。如果选择到相同的实例，会导致并行数量减少
                selected.add(invoker);
            }
        }
    }
    RpcContext.getContext().setInvokers((List)selected);
    final AtomicInteger count = new AtomicInteger(); // 异常数量计数器
    final BlockingQueue<Object> ref = new LinkedBlockingQueue<Object>(); //用来存储提供者返回的结果或者是调用后抛出的异常
    for (final Invoker<T> invoker : selected) {
        executor.execute(new Runnable() { //Executors.newCachedThreadPool线程池调度线程，异步调用提供者
            public void run() {
                try {
                    Result result = invoker.invoke(invocation);
                    ref.offer(result); //
                } catch(Throwable e) {
                    int value = count.incrementAndGet();
                    if (value >= selected.size()) {//如果异常次数已经等于提供者数量，证明所有提供者调用失败，将异常放入队列
                        ref.offer(e);
                    }
                }
            }
        });
    }
    try {
        Object ret = ref.poll(timeout, TimeUnit.MILLISECONDS); //阻塞获取BlockingQueue的结果（可能是正确结果或者异常）
        if (ret instanceof Throwable) {
            Throwable e = (Throwable) ret;
            throw new RpcException(e instanceof RpcException ? ((RpcException)e).getCode() : 0, "Failed to forking invoke provider " + selected + ", but no luck to perform the invocation. Last error is: " + e.getMessage(), e.getCause() != null ? e.getCause() : e);
        }
        return (Result) ret;
    } catch (InterruptedException e) {
        throw new RpcException("Failed to forking invoke provider " + selected + ", but no luck to perform the invocation. Last error is: " + e.getMessage(), e);
    }
}
```

## Broadcast 广播调用
轮询所有提供者实例，只返回最后一个提供者实例的结果。任意一个实例抛出异常则整个RPC过程异常。
通常用于通知所有提供者更新缓存或日志等本地资源信息。
以下是源码，只保留关键部分。
[`com.alibaba.dubbo.rpc.cluster.support.BroadcastClusterInvoker`](https://github.com/alibaba/dubbo/blob/master/dubbo-cluster/src/main/java/com/alibaba/dubbo/rpc/cluster/support/BroadcastClusterInvoker.java)
```java
public Result doInvoke(final Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    checkInvokers(invokers, invocation);
    RpcContext.getContext().setInvokers((List)invokers);
    RpcException exception = null;
    Result result = null;
    for (Invoker<T> invoker: invokers) {
        try {
            result = invoker.invoke(invocation);
        } catch (RpcException e) {
            exception = e;
            logger.warn(e.getMessage(), e);
        } catch (Throwable e) {
            exception = new RpcException(e.getMessage(), e);
            logger.warn(e.getMessage(), e);
        }
    }
    if (exception != null) {
        throw exception;
    }
    return result;
}
```

## Mergeable 合并调用
调用多个实例，并调用合并器Merger合并所有的返回结果
这个代码比较长，略。
[`com.alibaba.dubbo.rpc.cluster.support.MergeableClusterInvoker`](https://github.com/alibaba/dubbo/blob/master/dubbo-cluster/src/main/java/com/alibaba/dubbo/rpc/cluster/support/MergeableClusterInvoker.java)


# Dubbo负载均衡算法
## Random LoadBalance(随机均衡算法)
根据权重进行实例的随机选择，即每个实例的随机选中的概率是根据权重的决定的。
```java
protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
  int length = invokers.size(); // 总个数
  int totalWeight = 0; // 总权重
  boolean sameWeight = true; // 权重是否都一样
  for (int i = 0; i < length; i++) {
      int weight = getWeight(invokers.get(i), invocation);
      totalWeight += weight; // 累计总权重
      if (sameWeight && i > 0
              && weight != getWeight(invokers.get(i - 1), invocation)) {
          sameWeight = false; // 计算所有权重是否一样
      }
  }
  if (totalWeight > 0 && ! sameWeight) {
      // 如果权重不相同且权重大于0则按总权重数随机
      int offset = random.nextInt(totalWeight);
      // 并确定随机值落在哪个片断上
      for (int i = 0; i < length; i++) {
          offset -= getWeight(invokers.get(i), invocation);
          if (offset < 0) {
              return invokers.get(i);
          }
      }
  }
  // 如果权重相同或权重为0则均等随机
  return invokers.get(random.nextInt(length));
}
```
## RoundRobin LoadBalance(权重轮循均衡算法)
```java
protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
	String key = invokers.get(0).getUrl().getServiceKey() + "." + invocation.getMethodName();
	int length = invokers.size(); // 总个数
	int maxWeight = 0; // 最大权重
	int minWeight = Integer.MAX_VALUE; // 最小权重
	final LinkedHashMap<Invoker<T>, IntegerWrapper> invokerToWeightMap = new LinkedHashMap<Invoker<T>, IntegerWrapper>();
	int weightSum = 0;
	for (int i = 0; i < length; i++) {
		int weight = getWeight(invokers.get(i), invocation);
		maxWeight = Math.max(maxWeight, weight); // 累计最大权重
		minWeight = Math.min(minWeight, weight); // 累计最小权重
		if (weight > 0) {
			invokerToWeightMap.put(invokers.get(i), new IntegerWrapper(weight));
			weightSum += weight;
		}
	}
	AtomicPositiveInteger sequence = sequences.get(key);
	if (sequence == null) {
		sequences.putIfAbsent(key, new AtomicPositiveInteger());
		sequence = sequences.get(key);
	}
	int currentSequence = sequence.getAndIncrement();
	if (maxWeight > 0 && minWeight < maxWeight) { // 权重不一样
		int mod = currentSequence % weightSum;
		for (int i = 0; i < maxWeight; i++) {
			for (Map.Entry<Invoker<T>, IntegerWrapper> each : invokerToWeightMap.entrySet()) {
				final Invoker<T> k = each.getKey();
				final IntegerWrapper v = each.getValue();
				if (mod == 0 && v.getValue() > 0) {
					return k;
				}
				if (v.getValue() > 0) {
					v.decrement();
					mod--;
				}
			}
		}
	}
	// 取模轮循
	return invokers.get(currentSequence % length);
}
```

## LeastAction LoadBalance(最少活跃调用数均衡算法)
如果一个实例被调用的次数较少，则会优先调用该实例。
```java
protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
    int length = invokers.size(); // 总个数
    int leastActive = -1; // 最小的活跃数
    int leastCount = 0; // 相同最小活跃数的个数
    int[] leastIndexs = new int[length]; // 相同最小活跃数的下标
    int totalWeight = 0; // 总权重
    int firstWeight = 0; // 第一个权重，用于于计算是否相同
    boolean sameWeight = true; // 是否所有权重相同
    for (int i = 0; i < length; i++) {
    	Invoker<T> invoker = invokers.get(i);
        int active = RpcStatus.getStatus(invoker.getUrl(), invocation.getMethodName()).getActive(); // 活跃数
        int weight = invoker.getUrl().getMethodParameter(invocation.getMethodName(), Constants.WEIGHT_KEY, Constants.DEFAULT_WEIGHT); // 权重
        if (leastActive == -1 || active < leastActive) { // 发现更小的活跃数，重新开始
            leastActive = active; // 记录最小活跃数
            leastCount = 1; // 重新统计相同最小活跃数的个数
            leastIndexs[0] = i; // 重新记录最小活跃数下标
            totalWeight = weight; // 重新累计总权重
            firstWeight = weight; // 记录第一个权重
            sameWeight = true; // 还原权重相同标识
        } else if (active == leastActive) { // 累计相同最小的活跃数
            leastIndexs[leastCount ++] = i; // 累计相同最小活跃数下标
            totalWeight += weight; // 累计总权重
            // 判断所有权重是否一样
            if (sameWeight && i > 0
                    && weight != firstWeight) {
                sameWeight = false;
            }
        }
    }
    // assert(leastCount > 0)
    if (leastCount == 1) {
        // 如果只有一个最小则直接返回
        return invokers.get(leastIndexs[0]);
    }
    if (! sameWeight && totalWeight > 0) {
        // 如果权重不相同且权重大于0则按总权重数随机
        int offsetWeight = random.nextInt(totalWeight);
        // 并确定随机值落在哪个片断上
        for (int i = 0; i < leastCount; i++) {
            int leastIndex = leastIndexs[i];
            offsetWeight -= getWeight(invokers.get(leastIndex), invocation);
            if (offsetWeight <= 0)
                return invokers.get(leastIndex);
        }
    }
    // 如果权重相同或权重为0则均等随机
    return invokers.get(leastIndexs[random.nextInt(leastCount)]);
}
```

## ConsistentHash LoadBalance(一致性Hash均衡算法)
根据一致性哈希算法，
一致性哈希选择器依赖的参数是：
- virtualInvokers：每个哈希槽对应的Invoker
- replicaNumber：理解为哈希槽的数量，由hash.nodes指定，默认值是160
- identityHashCode：哈希码，根据invokers生成
- argumentIndex：参数索引数组，int[]类型。由hash.arguments指定，默认值是0。会根据该参数来确定选择那些输入参数作为key生成的依据。然后MD5之后做Hash。
**综上，可以认为，同一个方法的调用中，如果参数的哈希值相同则会调用同一个实例。**
[源码解析](http://blog.csdn.net/revivedsun/article/details/71022871)
```java
protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
    String key = invokers.get(0).getUrl().getServiceKey() + "." + invocation.getMethodName();
    int identityHashCode = System.identityHashCode(invokers);
    ConsistentHashSelector<T> selector = (ConsistentHashSelector<T>) selectors.get(key);
    if (selector == null || selector.getIdentityHashCode() != identityHashCode) {
        selectors.put(key, new ConsistentHashSelector<T>(invokers, invocation.getMethodName(), identityHashCode));
        selector = (ConsistentHashSelector<T>) selectors.get(key); //ConsistentHashSelector 一致性哈希选择器
    }
    return selector.select(invocation);
}
```

# 引用
[Dubbo开发者指南](http://dubbo.io/Developer+Guide-zh.htm)
[Dubbo架构设计详解](http://shiyanjun.cn/archives/325.html)
