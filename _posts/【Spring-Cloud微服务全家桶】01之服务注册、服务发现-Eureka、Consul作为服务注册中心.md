---
title: 【Spring Cloud微服务全家桶】01之服务注册、服务发现(Eureka、Consul作为服务注册中心)
date: 2017-08-27 16:09:54
tags: 
---

# 何为微服务？

Martin Fowler的《微服务》译文：<https://skyao.gitbooks.io/learning-microservice/content/definition/Martin-Fowler/microservices.html>

Martin Fowler的《微服务》原文：<https://martinfowler.com/articles/microservices.html>



# Spring Cloud简介

Spring Cloud官方文档：<http://projects.spring.io/spring-cloud/spring-cloud.html>

Spring Cloud是一个基于Spring Boot实现的云应用开发工具，它为基于JVM的云应用开发中涉及的配置管理、服务发现、断路器、智能路由、微代理、控制总线、全局锁、决策竞选、分布式会话和集群状态管理等操作提供了一种简单的开发方式。

Spring Cloud包含了多个子项目（针对分布式系统中涉及的多个不同开源产品），比如：Spring Cloud Config、Spring Cloud Netflix、Spring Cloud CloudFoundry、Spring Cloud AWS、Spring Cloud Security、Spring Cloud Commons、Spring Cloud Zookeeper、Spring Cloud CLI等项目。

Eureka：实际上在整个过程中维护者每个服务的生命周期。每一个服务都要被注册到Eureka服务器上，这里被注册到Eureka的服务又称为Client。Eureka通过心跳来确定服务是否正常。Eureka只做请求转发。同时Eureka是支持集群的呦！！！ （当然用其他的也是可以如consul,zookeeper）Zuul：类似于网关，反向代理。为外部请求提供统一入口。Ribbon/Feign：可以理解为调用服务的客户端。Hystrix：断路器，服务调用通常是深层的，一个底层服务通常为多个上层服务提供服务，那么如果底层服务失败则会造成大面积失败，Hystrix就是就调用失败后触发定义好的处理方法，从而更友好的解决出错。也是微服务的容错机制。



# 创建服务注册中心

{% asset_img 01.png  %}

上图简要描述了Eureka的基本架构，由3个角色组成：



- Eureka Server：提供服务注册和发现

- Service Provider：服务提供方，将自身服务注册到Eureka，从而使服务消费方能够找到

- Service Consumer*：服务消费方，从Eureka获取注册服务列表，从而能够消费服务。*

  ​

  *需要注意的是，上图中的3个角色都是逻辑角色。在实际运行中，这几个角色甚至可以是同一个实例，比如在我们项目中，Eureka Server和Service Provider就是同一个JVM进程。*

在创建服务注册中心前，我们先创建一个maven的pom父项目，然后在这个maven父项目里，使用Spring Initializr创建一个module。

{% asset_img 02.png  %}

发现在pom.xml已经自动生成了依赖，当然也可以创建一个普通的maven项目，将以下pom.xml的配置添加进去

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>1.5.6.RELEASE</version>
    <relativePath/>
</parent>

<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
    <java.version>1.8</java.version>
    <spring-cloud.version>Dalston.SR1</spring-cloud.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-eureka-server</artifactId>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>Dalston.SR1</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

修改application.properties配置

```properties
spring.application.name=registry-ms #服务名称
server.port=1001

eureka.instance.hostname=registry-ms.pay-inner.com #服务使用的主机名
eureka.client.register-with-eureka=false #eureka本身无需注册到注册中心
eureka.client.fetch-registry=false #eureka本身无需从注册中心获取服务注册实例
```

*通过eureka.client.register-with-eureka：false和fetch-registry：false来表明自己是一个eureka server*



启动服务注册中心

启动类上加上 @EnableEurekaServer注解，表示开启Eureka注册中心服务

```java
@EnableEurekaServer
@SpringBootApplication
public class RegistryApplication {
    public static void main( String[] args ) {
        SpringApplication.run(RegistryApplication.class,args);
    }
}
```



看到Instances currently registered with Eureka 还没有服务实例在线

{% asset_img 03.png  %}

# 创建一个简单的服务提供者

服务向注册中心注册完成后，服务的eureka client会告知注册中心它的元数据信息，例如ServiceId、主机名、URI、端口号等。Eureka Server接受每个Eureka客户端的心跳，当注册中心感知不到服务实例的存在时（比如心跳超时），那么该服务实例就会从注册中心中剔除。

创建一个SpringBoot应用，怎么创建的过程不再赘述，名为provider-service-ms，由于本例是服务注册、服务发现，可以写个打印所有服务实例服务方法。

创建完后，pom.xml

```java
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.5.4.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.choosefine</groupId>
    <artifactId>provider-ms</artifactId>
    <packaging>jar</packaging>

    <name>provider-ms</name>
    <url>http://maven.apache.org</url>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
        <spring-cloud.version>Dalston.SR1</spring-cloud.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-eureka</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>Dalston.SR1</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>

```

通过注解@EnableDiscoveryClient 表明自己是一个Eureka Client

```java
@EnableDiscoveryClient
@SpringBootApplication
public class ProviderApplication {
    public static void main( String[] args ) {
        SpringApplication.run(ProviderApplication.class,args);
    }
}

```

写一个Rest Controller用于查看服务注册中心中注册的服务实例列表，DiscoveryClient提供了，获取服务列表的方法。

```java
@RestController
public class ProviderDemoController {
    @Autowired
    private DiscoveryClient discoveryClient;

    @RequestMapping("services")
    public List<String> clientList(){
        List<String> serviceIds = discoveryClient.getServices();
        for(String serviceId : serviceIds) {
            List<ServiceInstance> instances = discoveryClient.getInstances(serviceId);
            for (ServiceInstance instance : instances) {
                System.out.println("ServiceId:" + instance.getServiceId() + ",Host:" + instance.getHost() + ",Port:" + instance.getPort() + ",URI:" + instance.getUri() + ",MetaData:" + instance.getMetadata());
            }
        }
        return serviceIds;
    }
}

```



然后对application.properties做一些配置工作

spring.application.name是很重的一个配置，该名称会作为注册中心中服务实例的id(即ServiceId)，当其他服务需要调用此服务的时候，就需要知道此服务的ServiceId，eureka.client.serviceUrl.defaultZone 指明注册中心的地址。

```properties
server.port=2104
spring.application.name=provider-ms
eureka.client.serviceUrl.defaultZone=http://registry-ms.pay-inner.com:1001/eureka
```

启动ProviderApplication，观察当前注册到eureka的服务实例，看到此时已经有名为PROVIDER-MS的服务实例上线了（Status为UP）。

{% asset_img 04.png  %}

也可以通过访问http://127.0.0.1:2103/services（provider-ms.pay-inner.com是为了访问方便，配了本地hosts域名解析），查看注册到eureka的服务实例。

{% asset_img 05.png  %}

控制台也打印了 ServiceId、Host、Port、URI这些信息

{% asset_img 06.png  %}



#将eureka改用consul实现服务注册

- Consul是什么

  Consul 是一个支持多数据中心分布式高可用的服务发现和配置共享的服务软件,由 HashiCorp 公司用 Go 语言开发, 基于 Mozilla Public License 2.0 的协议进行开源. Consul 支持健康检查,并允许 HTTP 和 DNS 协议调用 API 存储键值对. 命令行超级好用的虚拟机管理软件 vgrant 也是 HashiCorp 公司开发的产品. 一致性协议采用 Raft 算法,用来保证服务的高可用. 使用 GOSSIP 协议管理成员和广播消息, 并且支持 ACL 访问控制.

- Consul有什么优势

使用 Raft 算法来保证一致性, 比复杂的 Paxos 算法更直接. 相比较而言, zookeeper 采用的是 Paxos, 而 etcd 使用的则是 Raft.

​     支持多数据中心，内外网的服务采用不同的端口进行监听。 多数据中心集群可以避免单数据中心的单点故障,而其部署则需要考虑网络延迟, 分片等情况等. zookeeper 和 etcd 均不提供多数据中心功能的支持.

​     支持健康检查. etcd 不提供此功能

​     支持 http 和 dns 协议接口. zookeeper 的集成较为复杂, etcd 只支持 http 协议** **     官方提供web管理界面, etcd 无此功能 

只需要更换服务治理的依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-consul-discovery</artifactId>
</dependency>

```

再将application.properties配置修改下，将eureka地址的配置注释掉

```properties
server.port=2103
spring.application.name=provider-ms
#eureka.client.serviceUrl.defaultZone=http://registry-ms.pay-inner.com:1001/eureka
spring.cloud.consul.host=localhost
spring.cloud.consul.port=8500

```

由于Spring Cloud 对于DiscoveryClient已经做了一层抽象，我们无需关心eureka与consul不同的实现细节。而我们需要做的只是更改springcloud封装的不同服务治理依赖，以及在配置文件中配置不同的属性

将consul用dev模式启动。由于consul已经为我们提供了服务，无需像之前创建eureka那样创建服务注册中心，直接从consul官网下载服务端即可,<https://www.consul.io/>

{% asset_img 07.png  %}

重新启动ProviderApplication

观察Consul UI（http://127.0.0.1:8500/ui/），是否有服务上线，可以看到PROVIDER-MS已经注册到consul了

{% asset_img 08.png  %}

通过访问http://127.0.0.1:2103/services

{% asset_img 09.png  %}

控制台也打印了 ServiceId、Host、Port、URI这些信息，这里也会把consul打印，因为默认是将consul本身也注册到服务注册中心

{% asset_img 10.png  %}

注意：这里看到http://XXXXXX:2103/health 404原因是少依赖了，pom.xml添加以下依赖即可

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

```



示例代码：<https://git.oschina.net/jaychang/spring-cloud-demo.git>