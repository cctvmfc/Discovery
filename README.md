# Nepxion Discovery
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg?label=license)](https://github.com/Nepxion/Discovery/blob/master/LICENSE)
[![Maven Central](https://img.shields.io/maven-central/v/com.nepxion/discovery.svg?label=maven%20central)](http://search.maven.org/#search%7Cga%7C1%7Cg%3A%22com.nepxion%22%20AND%20discovery)
[![Javadocs](http://www.javadoc.io/badge/com.nepxion/discovery-plugin.svg)](http://www.javadoc.io/doc/com.nepxion/discovery-plugin)
[![Build Status](https://travis-ci.org/Nepxion/Discovery.svg?branch=master)](https://travis-ci.org/Nepxion/Discovery)

Nepxion Discovery是一款对Spring Cloud Discovery的服务注册增强插件，目前暂时只支持Eureka。现在Spring Cloud服务可以方便引入该插件，不需要对业务代码做任何修改，只需要修改配置即可

## 简介
支持如下功能

    1. 实现采用黑/白名单的IP地址过滤机制实现对客户端的禁止注册
    2. 实现通过多版本配置实现灰度访问控制
    3. 实现通过远程配置中心控制黑/白名单和灰度版本，实现动态通知
    4. 实现通过事件总线机制异步对接远程配置中心，提供使用者实现和扩展
    5. 实现支持本地配置和远程配置的选择

## 依赖
```xml
<dependency>
  <groupId>com.nepxion</groupId>
  <artifactId>discovery-plugin-starter</artifactId>
  <version>${discovery.plugin.version}</version>
</dependency>
```

## 示例
### 场景示例

    1. 黑/白名单的IP地址过滤
       开发环境的本地服务（例如IP地址为172.16.0.8）不小心注册到测试环境的服务注册发现中心，会导致调用出现问题，那么可以在配置中心维护一个黑/白名单的IP地址过滤（支持全局和局部的过滤）
       我们可以在远程配置中心配置对该服务名所对应的IP地址列表，包含前缀172.16，当是黑名单的时候，表示包含在IP地址列表里的所有服务都禁止注册到服务注册发现中心；当是白名单的时候，表示包含在IP地址列表里的所有服务都允许注册到服务注册发现中心
    2. 多版本配置实现灰度访问控制
       A服务调用B服务，而B服务有两个实例（B1、B2和B3），虽然三者相同的服务名，但功能上有差异，需求是在某个时刻，A服务只能调用B1，禁止调用B2和B3。在此场景下，我们在application.properties里为B1维护一个版本为1.0，为B2维护一个版本为1.1，以此类推
       我们可以在远程配置中心配置对于A服务调用某个版本的B服务，达到某种意义上的灰度控制，切换版本的时候，我们只需要改相关的远程配置中心的配置即可

### 配置文件
```xml
<?xml version="1.0" encoding="UTF-8"?>
<rule>
    <!-- 服务注册的黑/白名单过滤。白名单表示只允许指定IP地址前缀注册，黑名单表示不允许指定IP地址前缀注册。每个服务只能同时开启要么白名单，要么黑名单 -->
    <!-- filter-type，可选值BLACKLIST/WHITELIST，表示白名单或者黑名单 -->
    <!-- service-name，表示服务名 -->
    <!-- filter-value，表示黑/白名单的IP地址列表。IP地址一般用前缀来表示，如果多个用“,”分隔 -->
    <!-- 表示下面所有服务，不允许10.10和11.11为前缀的IP地址注册（全局过滤） -->
    <register filter-type="BLACKLIST" filter-value="10.10,11.11">
        <!-- 表示下面服务，不允许172.16和10.10和11.11为前缀的IP地址注册 -->
        <service service-name="discovery-springcloud-example-a" filter-value="172.16"/>
    </register>

    <!-- 服务发现下，服务多版本调用的控制 -->
    <!-- service-name，表示服务名 -->
    <!-- version-value，表示可供访问的版本，如果多个用“,”分隔 -->
    <discovery>
        <!-- 表示消费端服务a的1.0，允许访问提供端服务b的1.0和1.1版本 -->
        <service consumer-service-name="discovery-springcloud-example-a" provider-service-name="discovery-springcloud-example-b" consumer-version-value="1.0" provider-version-value="1.0,1.1"/>
    </discovery>
</rule>
```

### 配置策略
```xml
配置策略介绍
1. 标准配置，举例如下
   <service consumer-service-name="a" provider-service-name="b" consumer-version-value="1.0" provider-version-value="1.0,1.1"/> 表示消费端1.0版本，允许访问提供端1.0和1.1版本
2. 版本值不配置，举例如下
   <service consumer-service-name="a" provider-service-name="b" provider-version-value="1.0,1.1"/> 表示消费端任何版本，允许访问提供端1.0和1.1版本
   <service consumer-service-name="a" provider-service-name="b" consumer-version-value="1.0"/> 表示消费端1.0版本，允许访问提供端任何版本
   <service consumer-service-name="a" provider-service-name="b"/> 表示消费端任何版本，允许访问提供端任何版本
3. 版本值空字符串，举例如下
   <service consumer-service-name="a" provider-service-name="b" consumer-version-value="" provider-version-value="1.0,1.1"/> 表示消费端任何版本，允许访问提供端1.0和1.1版本
   <service consumer-service-name="a" provider-service-name="b" consumer-version-value="1.0" provider-version-value=""/> 表示消费端1.0版本，允许访问提供端任何版本
   <service consumer-service-name="a" provider-service-name="b" consumer-version-value="" provider-version-value=""/> 表示消费端任何版本，允许访问提供端任何版本
4. 版本对应关系未定义，默认消费端任何版本，允许访问提供端任何版本
特殊情况处理，在使用上需要极力避免该情况发生
1. 消费端的application.properties未定义版本号（即eureka.instance.metadataMap.version不存在），则该消费端可以访问提供端任何版本
2. 提供端的application.properties未定义版本号（即eureka.instance.metadataMap.version不存在），当消费端在xml里不做任何版本配置，才可以访问该提供端
```

### 跟远程配置中心整合
使用者可以跟携程Apollo，百度DisConf等远程配置中心整合

继承AbstractConfigLoader.java，实现配置文件获取的对接
```java
public class DiscoveryConfigLoader extends AbstractConfigLoader {
    // 通过application.properties里的spring.application.discovery.remote.config.enabled=true，来决定走远程配置中心，还是本地
    // 从远程配置中心获取XML内容
    @Override
    public InputStream getRemoteInputStream() {
        return null;
    }

    // 从本地获取XML内容
    @Override
    protected String getLocalContextPath() {
        // 配置文件放在resources目录下
        return "classpath:rule1.xml";

        // 配置文件放在工程根目录下
        // return "file:rule1.xml";
    }
}
```

实现接收远程配置中心推送过来的配置更新
```java
public class DiscoveryConfigSubscriber {
    @Autowired
    private ConfigPublisher configPublisher;

    public void subscribe() {
        // 订阅远程推送内容，再通过内置的发布订阅机制异步推送给缓存
        configPublisher.publish(inputStream);
    }
}
```

### 代码示例
#### B服务实现
B服务的两个实例B1和B2采用标准的Spring Cloud入口，参考discovery-springcloud-example-b1、discovery-springcloud-example-b2和discovery-springcloud-example-b3工程
唯一需要做的是在applicaiton.properties维护版本号，如下
```xml
eureka.instance.metadataMap.version=[version]
```

#### A服务实现
A服务需要引入discovery-plugin-starter，参考discovery-springcloud-example-a工程

application.properties
```xml
# Spring cloud config
spring.application.name=discovery-springcloud-example-a
server.port=4321
eureka.client.serviceUrl.defaultZone=http://10.0.75.1:9528/eureka/
eureka.instance.preferIpAddress=true
eureka.instance.metadataMap.version=1.0

# Plugin config
# 开启和关闭服务注册层面的控制。一旦关闭，服务注册的黑/白名单过滤功能将失效。缺失则默认为true
spring.application.register.control.enabled=true

# 开启和关闭服务发现层面的控制。一旦关闭，服务多版本调用的控制功能将失效，动态屏蔽指定IP地址的服务示例功能将失效。缺失则默认为true
spring.application.discovery.control.enabled=true

# 开启和关闭远程配置中心规则配置文件读取。一旦关闭，默认读取本地规则配置文件（例如：rule.xml）。缺失则默认为true
spring.application.discovery.remote.config.enabled=true

management.security.enabled=false
```
因为本中间件不跟任何远程配置中心系统绑定（需要使用者自行实现跟远程配置中心对接），故通过定时方式模拟获取远程配置内容的更新推送，参考DiscoveryConfigurationLoader.java和DiscoveryConfigurationSimulator.java

#### 黑/白名单的IP地址过滤运行效果
启动discovery-springcloud-example-a/DiscoveryApplication.java的时候，如果IP地址被过滤，那么程序将抛出无法注册到服务注册发现中心的异常，并终止程序

#### 多版本配置实现灰度访问控制运行效果
先运行discovery-springcloud-example-b1、discovery-springcloud-example-b2和discovery-springcloud-example-b3下的DiscoveryApplication.java，再运行discovery-springcloud-example-a/DiscoveryApplication.java，通过Postman访问
```xml
http://localhost:4321/instances
```
你可以看到通过A服务去获取B服务的被过滤的实例列表，虽然A服务定时器会更新不不同的配置，获取到的实例列表也随着变更

## 鸣谢
感谢Spring Cloud中国社区刘石明提供代码支持和建议