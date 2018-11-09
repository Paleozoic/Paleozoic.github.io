---
title: Spring Boot Starter封装之术
date: 2018-11-09 22:30:23
categories: [Spring Boot]
tags: [Spring Boot]
---
<Excerpt in index | 首页摘要>
We can just import two spring boot starters to easily build a project with shiro RBAC configuration and multi-datasources(also multi-dialects) configuration. 
What we should do is to implemente the interface `com.maxplus1.access.starter.config.shiro.rbac.service.IShiroService` and to edit the file `application.yml`. 
The starters depend on `Druid`,`Shiro` and `MyBatis`.<!-- more -->
<The rest of contents | 余下全文>
# 写在前面
- [spring-boot-starter-demo](https://github.com/Paleozoic/spring-boot-starter-demo)
- [mybatis-spring-boot-starter](https://github.com/Paleozoic/mybatis-spring-boot-starter)
- [shiro-spring-boot-starter](https://github.com/Paleozoic/shiro-spring-boot-starter)

# 整合配置文件
```yml
spring:
    autoconfigure:
      exclude: # 以下2个AutoConfiguration会自动读取Spring默认数据源，导致报错
        - org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
        - org.springframework.boot.autoconfigure.flyway.FlywayAutoConfiguration
    redis:
      database: 0
      host: 192.168.0.105
      port: 6379
      password: foobared
      lettuce:
        pool:
          max-active: 8
          max-wait: -1
          max-idle: 8
          min-idle: 0
      timeout: 100s
    maxplus1:
      druid: # 公共属性
        minIdle: 5
        maxActive: 20
        initialSize: 5
        timeBetweenEvictionRunsMillis: 3000
        minEvictableIdleTimeMillis: 300000
        validationQuery: SELECT 1 FROM DUAL
        testWhileIdle: true
        testOnBorrow: false
        testOnReturn: false
        data-sources: # 数据源个性化配置
          master: # 注册的DataSource名字为：masterDataSource
            username: paleo
            password: paleo
            url: jdbc:oracle:thin:@192.168.186.129:1521:maxplus
            driver-class-name: oracle.jdbc.OracleDriver
          slave:
            username: maxplus
            password: maxplus
            url: jdbc:mysql://192.168.186.128:3306/maxplus?useUnicode=true&characterEncoding=UTF-8
            driver-class-name: com.mysql.jdbc.Driver
      mybatis: # 公共属性
          type-aliases-package: com.maxplus1.demo.data.entity
          configuration:
              map-underscore-to-camel-case: true
              default-fetch-size: 100
              default-statement-timeout: 30
          data-sources:
            master: # 数据源+MyBatis个性化配置。默认第一个是主数据源，即@Primary
              mapper-locations: classpath:mapper/master/*.xml # 可以输入数组
              base-package: com.maxplus1.demo.data.dao.master
      pagehelper: # 分页参数，公共属性
        pageSizeZero: true
        rowBoundsWithCount: true
        offsetAsPageNum: true
        supportMethodsArguments: true
        data-sources:
          master:
            helperDialect: oracle
          slave:
            helperDialect: mysql
      shiro:
        tokenKey: uuusid
        loginUrl: /api/sys/login
        filterChain: | # 注意所有perms的权限都需要通过@RequirePermissions来实现，建议不要配置perms（动态URL问题）。PS:yaml配置map的key含有/时，无法识别/。
            /static/**=anon
            /api/sys/ssoLogin=anon
            /api/sys/login=anon
            /api/sys/testRedis=anon
            /api/**=authc
            /**=authc
            /api/sys/logout=logout
        globalSessionTimeout: 180000000 # 3,600,000 milliseconds = 1 hour
        sessionValidationInterval: 360000 # 会话有效校验扫描间隔
        redisCacheEnabled: false # 开启分布式session 需要引入Redis的相关配置，spring-data-redis
        testMode: false # 开启测试模式，测试模式下所有url的访问权限都是anon
        mockUser: # 测试模式模拟的用户
            userId: 'MOCK_USER0'
            userName: 'mock_yonghu0'
            deptId: 'MOCK_DEPT0'
            deptName: '模拟部门0'
            realName: '模拟用户0'
            status: '正常'
            password: 'MOCK_PASS0'
```

# Spring Bean装载过程
## Spring装配Bean的过程
1. 实例化;
2. 设置属性值;
3. 如果实现了BeanNameAware接口,调用setBeanName设置Bean的ID或者Name;
4. 如果实现BeanFactoryAware接口,调用setBeanFactory 设置BeanFactory;
5. 如果实现ApplicationContextAware,调用setApplicationContext设置ApplicationContext
6. 调用BeanPostProcessor的预先初始化方法;
7. 调用InitializingBean的afterPropertiesSet()方法;
8. 调用定制init-method方法；
9. 调用BeanPostProcessor的后初始化方法;


## Spring容器关闭过程
1. 调用DisposableBean的destroy();
2. 调用定制的destroy-method方法;



[引用自](https://www.cnblogs.com/fanguangdexiaoyuer/p/5886050.html)


# Spring Boot SPI 
Spring Boot SPI 可以代替包扫描的配置
Spring Boot SPI 类似于 Java SPI的加载机制。
META-INF/spring.factories配置：
```
# 如下：
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.mybatis.spring.boot.autoconfigure.MybatisAutoConfiguration
```


# BeanPostProcessor
- BeanPostProcessor包含了InitializingBean的前后拦截器

# Spring的监听器
[了解一下Spring Boot监听器的生命周期](https://www.cnblogs.com/senlinyang/p/8496099.html)，更具体的可以去看官方文档。


# flyway
数据库迁移工具：flyway支持。flyway企业版才支持Oracle 11。
可以尝试回退到4.2.0版本。flayway的思路是最新版本免费支持最新的Oracle~
在使用oracle的时候，如果报莫名错误，尝试如下sql：
```sql
truncate table "schema_version";
commit;
```

# 整合问题
我调试的过程和此文类似，就是为了搞清楚`BeanPostProcessor`的调用顺序。
[详情点击](https://blog.csdn.net/m0_37962779/article/details/78605478)
于是我在内部类`DruidDataSourceConfiguration.DruidDataSourceBeanPostProcessor`中，让DruidDataSourceBeanPostProcessor实现了`PriorityOrdered`接口，
让数据源可以优先处理。
否则，在整合我的2个自定义starter，Shiro和MyBatis造成冲突。（复杂的依赖关系）
```xml
<dependency>
    <groupId>com.maxplus1</groupId>
    <artifactId>shiro-spring-boot-starter</artifactId>
    <version>1.0.4</version>
</dependency>
<dependency>
    <groupId>com.maxplus1</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>1.0.3</version>
</dependency>
```
报错如下：
```log
2018-11-07 16:38:50,128:INFO main (DefaultLazyPropertyFilter.java:31) - Property Filter custom Bean not found with name 'encryptablePropertyFilter'. Initializing Default Property Filter
2018-11-07 16:38:53,291:INFO main (PostProcessorRegistrationDelegate.java:328) - Bean 'org.springframework.boot.context.properties.ConversionServiceDeducer$Factory' of type [org.springframework.boot.context.properties.ConversionServiceDeducer$Factory] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2018-11-07 16:38:54,310:INFO main (DefaultLazyPropertyResolver.java:31) - Property Resolver custom Bean not found with name 'encryptablePropertyResolver'. Initializing Default Property Resolver
2018-11-07 16:38:54,400:INFO main (DefaultLazyPropertyDetector.java:30) - Property Detector custom Bean not found with name 'encryptablePropertyDetector'. Initializing Default Property Detector
2018-11-07 16:38:55,551:INFO main (PostProcessorRegistrationDelegate.java:328) - Bean 'spring.maxplus1.shiro-com.maxplus1.access.starter.config.shiro.ShiroProperties' of type [com.maxplus1.access.starter.config.shiro.ShiroProperties] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2018-11-07 16:38:55,662:INFO main (PostProcessorRegistrationDelegate.java:328) - Bean 'com.maxplus1.access.starter.config.shiro.ShiroAutoConfiguration' of type [com.maxplus1.access.starter.config.shiro.ShiroAutoConfiguration] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2018-11-07 16:39:41,540:ERROR main (DruidDataSource.java:910) - {dataSource-1} init error
java.lang.NullPointerException: null
	at oracle.jdbc.driver.OracleDriver.connect(OracleDriver.java:420) ~[ojdbc6-11.2.0.4.jar:11.2.0.4.0]
	at com.alibaba.druid.pool.DruidAbstractDataSource.createPhysicalConnection(DruidAbstractDataSource.java:1558) ~[druid-1.1.10.jar:1.1.10]
	at com.alibaba.druid.pool.DruidAbstractDataSource.createPhysicalConnection(DruidAbstractDataSource.java:1623) ~[druid-1.1.10.jar:1.1.10]
	at com.alibaba.druid.pool.DruidDataSource.init(DruidDataSource.java:861) ~[druid-1.1.10.jar:1.1.10]
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[?:1.8.0_131]
```
调试也可以发现，没有进入DruidDataSourceBeanPostProcessor，Spring的自动装配顺序有点让人意外。
其中到底是Shiro Starter的哪个依赖提前加载了DataSource，影响了DruidDataSourceBeanPostProcessor，我已无法理清……
（头晕眼花的debug= =||，偷个懒，不调试了……）


# mybatis-spring-boot-starter
- pagination &  druid & mybatis for multi datasources(event different dialects),all in one strarter~
- 此工具通过一个starter直接开启多数据源（支持不同方言）的druid+mybatis+分页的工程，非常方便于复用~
- 写了大量的BeanDefinition，可能有缺少的属性。
>   如果@Bean可以一次性返回多个Bean并且注册到Spring容器，
则可以作出大量简化，并复用大量的starter。
比如：
```java
@Beans
public Map<String,Bean> manyBeans(){
    Map<String,Bean> beans = new HashMap<>();
    // 分别设置BeanName和Bean
    return beans;
}
```
详情看我提的 :

- [Spring Boot Issue](https://github.com/spring-projects/spring-boot/issues/14978)
- [Spring Issue](https://jira.spring.io/browse/SPR-17441)

## 注意
- 所有的bean，比如`PageInterceptor`  、`DruidDataSource` 、 `SqlSessionFactory` 、 `TransactionManager`等，所有注入的BeanName都是：数据源+类名。例如：`xxxTransactionManager`，`xxx`表示数据源的名字
- 但是`xxxTransactionManager`起了别名`xxx`，方便处理事务。用于`@Transactional('xxx')`

## PS
- 基本遵循mybatis-spring-boot-starter和druid-spring-boot-starter的配置格式
- 如果存在不支持属性，欢迎提issue

# shiro-spring-boot-starter
- it is easy to use shiro for rest api ~ quick start ~ base on spring boot 2.x
- 此starter封装并不只是为了使用完整的Shiro功能，目的是为大家提供一种封装思路，如果实际应用在业务场景，还需要具体情况具体分析
- 当然你也可以遵循目前的设计规范，如果有功能没有满足，不妨提个issue……
- 倘若你有自己的设计，fork过去尽情修改吧~

## JWT和Session的区别
- JWT：服务端无状态。所有状态存储在客户端的加密token里面。服务端只负责校验token的有效性。服务端无法强制下线。
- Session：服务端有状态。客户端只是存储一个sid，服务端存储sid对应的信息。多应用共享Session通常使用Redis之类的内存数据库存储。

## 登录鉴权方式
-  使用传统的session方式

## SessionId传递
- sessionId放在header或者cookie进行传递。header的sid具有高优先级。

## Session的序列化
- 由于Session都会实现Serializable，所以统一使用JDK的序列化方式，保证稳定性和兼容性。

## Redis配置
- Redis配置完全和spring-data-redis一致，且为默认的RedisTemplate
- 由于Shiro的序列化方式默认是JDK序列化，如果非Shiro需要使用其他序列化方式，需要单独配置，参考demo-redis。
- 多Redis数据源配置：此时`spring.redis`的配置依然需要配置，其他数据源单独配置。

## appId
- appId用于多系统交互时使用，如果系统不涉及此方面，可以忽略。
- appId可以通过配置文件配置，表示当前系统的app标识
- 也可以遵循协议，appId通过cookie或者header传递。header具有较高优先级（设计和Session一样）
- 为什么不放在Session？
    - 因为多系统可能公用一个Session，此时无法判断当前Session属于哪个系统。

## 配置文件示例
```yml
spring:
    redis:
      database: 0
      host: 192.168.0.105
      port: 6379
      password: foobared
      lettuce:
        pool:
          max-active: 8
          max-wait: -1
          max-idle: 8
          min-idle: 0
      timeout: 100s
    maxplus1:
      shiro:
          app:
            id: uuuappkey
            key: MaxPlus1     
          tokenKey: uuusid
          loginUrl: /api/sys/login
          filterChain: | # 注意所有perms的权限都需要通过@RequirePermissions来实现，建议不要配置perms（动态URL问题）。PS:yaml配置map的key含有/时，无法识别/。
              /static/**=anon
              /api/sys/ssoLogin=anon
              /api/sys/login=anon
              /api/sys/testRedis=anon
              /api/**=authc
              /**=authc
              /api/sys/logout=logout
          globalSessionTimeout: 180000000 # 3,600,000 milliseconds = 1 hour
          sessionValidationInterval: 360000 # 会话有效校验扫描间隔
          redisCacheEnabled: false # 开启分布式session 需要引入Redis的相关配置，spring-data-redis
          testMode: false # 开启测试模式，测试模式下所有url的访问权限都是anon
          mockUser: # 测试模式模拟的用户
              userId: 'MOCK_USER0'
              userName: 'mock_yonghu0'
              deptId: 'MOCK_DEPT0'
              deptName: '模拟部门0'
              realName: '模拟用户0'
              status: '正常'
              password: 'MOCK_PASS0'

```

## Filters与AuthorizingAnnotationMethodInterceptor
- `@RequiresPermissions`注解的处理是通过AOP实现拦截器`PermissionAnnotationMethodInterceptor`来处理；而`perms`是通过filter来实现的：`PermissionsAuthorizationFilter`。
- Filter与MethodInterceptor的区别，Filter是基于URL做的拦截，而MethodInterceptor是基于AOP直接对方法进行的拦截。对于REST URL，比如`/api/user/{id}`这样的url请求。
- Filter处理REST URL需要自己实现Filter并对url进行解析匹配（具体参考spring mvc的url匹配规则，略复杂）；或者直接使用MethodInterceptor。


## demo测试
- 启动demo or  demo-redis
- 登录测试（返回SessionId）：127.0.0.1:9000/demo/api/sys/login?userName=yonghu0&password=PASS0
- 访问测试（需要在cookie或者header带上uuusid=sessionId访问）：127.0.0.1:9000/demo/api/sys/testAccess
- 无权限访问测试（需要在cookie或者header带上uuusid=sessionId访问）：127.0.0.1:9000/demo/api/sys/testDeny
- 测试不同的Redis序列化方式：127.0.0.1:9000/demo/api/sys/testRedis

## HTTP状态码
-  未登录被拒绝：401 
-  未授权被拒绝：403 

## 注意
- 既然基于Shiro做了封装，那么可自定义的模块肯定减少，如果需要增加其他配置。可以：
    - 下载源码自行修改
    - 提issue到github
    - 使用原生Shiro进行个性化的封装
- `com.maxplus1.access.starter.config.shiro.interceptor.shiro.Perms` (已废弃)  
- 此包只适配了前后端分离的项目，没有对静态资源进行处理。且Shiro的异常以json形式返回。    

## deploy
- 更改parent下的pom.xml的`deploy2maven.url.snapshots`和`deploy2maven.url.releases`为自己私库的url
- 配置maven的settings.xml的server标签。如下：
```xml
<server>
    <id>user-snapshot</id>
    <username>your name</username>
    <password>your pass</password>
</server>
<server>
    <id>user-release</id>
    <username>your name</username>
    <password>your pass</password>
</server>
```
- 执行maven的deploy命令
    
## TODO
- Session的一级缓存，二级缓存  
- 动态更新权限，刷新缓存
- 同一账号多处登录问题
- 账户踢出接口
- [Shiro其他配置](http://shiro.apache.org/spring-boot.html#configuration-properties )   