---
title: 【Spring Cloud微服务全家桶】02之高可用服务注册中心
date: 2017-09-01 14:03:59
tags: Spring Cloud
---

# 前言

在Spring Cloud微服务全家桶之服务注册、服务发现(Eureka、Consul作为服务注册中心)文中，使用的是单点eureka注册中心。在开发测试环境是可以的，但生产环境强烈不建议用单点eureka注册中心。

#  双eureka高可用
{% asset_img 01.png  %}
思路就是每个eureka server通过eureka.client.serviceUrl.defaultZone配置，将自己注册到对方的注册中心。
新建一个module项目，建的过程这里省略，名为registry-ha-ms,建完后pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.choosefine</groupId>
  <artifactId>registry-ha-ms</artifactId>
  <version>0.0.1-SNAPSHOT</version>
  <packaging>jar</packaging>

  <name>registry-ha-ms</name>
  <description>Demo project for Spring Boot</description>

  <parent>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-parent</artifactId>
      <version>1.5.6.RELEASE</version>
      <relativePath/> <!-- lookup parent from repository -->
  </parent>

  <properties>
      <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
      <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
      <java.version>1.8</java.version>
      <spring-cloud.version>Dalston.SR2</spring-cloud.version>
  </properties>

  <dependencies>
      <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-eureka-server</artifactId>
      </dependency>

      <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
      </dependency>
  </dependencies>

  <dependencyManagement>
      <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
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

使用@EnableEurekaServer注解启用Eureka注册中心

```java
@EnableEurekaServer
@SpringBootApplication
public class RegistryHaApplication {

  public static void main(String[] args) {
      SpringApplication.run(RegistryHaApplication.class, args);
  }
}
```



建两个properties配置文件，分别为application-peer1.properties，application-peer2.properties

* application-peer1.properties


```properties
server.port=1011
spring.application.name=registry-ha-ms

eureka.instance.hostname=eureka-peer1
eureka.client.serviceUrl.defaultZone=http://eureka-peer2:1012/eureka
```

* application-peer2.properties

```properties
server.port=1012
spring.application.name=registry-ha-ms

eureka.instance.hostname=eureka-peer2
eureka.client.serviceUrl.defaultZone=http://eureka-peer1:1011/eureka
```

配置本地Hosts域名解析

```properties
127.0.0.1 eureka-peer1
127.0.0.1 eureka-peer2
```

在项目目录下使用maven clean package -DskipTests打包，进入target目录，开启两个命令行窗口分别执行

```
java -jar registry-ha-ms-0.0.1-SNAPSHOT --spring.profiles.active=peer1

java -jar registry-ha-ms-0.0.1-SNAPSHOT --spring.profiles.active=peer2
```

或者在idea里建两个执行配置

{% asset_img 02.png %}
{% asset_img 03.png %}

这样就开启了两个eureka server。

我们在上一篇文章的基础上，将provider-ms的application.properties修改下

```propertis
server.port=2103
spring.application.name=provider-ms
#eureka.client.serviceUrl.defaultZone=http://registry-ms.pay-inner.com:1001/eureka
eureka.client.serviceUrl.defaultZone=http://eureka-peer1:1011/eureka,http://eureka-peer2:1012/eureka
```

然后再启动ProviderApplication，可以看到PROVIDER-MS 已经注册到eureka-peer1与eureka-peer2上了。且eureka-peer1的replicas节点列表里有eureka-peer2，eureka-peer2的replicas节点列表里有eureka-peer1

{%asset_img 04.png%}

{%asset_img 05.png%}

新建一个服务消费项目，名为consumer-ms建完后pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.choosefine</groupId>
  <artifactId>consumer-ms</artifactId>
  <version>0.0.1-SNAPSHOT</version>
  <packaging>jar</packaging>

  <name>consumer-ms</name>
  <description>Demo project for Spring Boot</description>

  <parent>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-parent</artifactId>
      <version>1.5.6.RELEASE</version>
      <relativePath/> <!-- lookup parent from repository -->
  </parent>

  <properties>
      <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
      <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
      <java.version>1.8</java.version>
      <spring-cloud.version>Dalston.SR2</spring-cloud.version>
  </properties>

  <dependencies>
      <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-eureka</artifactId>
      </dependency>

      <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
      </dependency>
  </dependencies>

  <dependencyManagement>
      <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
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

修改properties资源文件

```properties
server.port=2104
spring.application.name=consumer-ms

eureka.client.serviceUrl.defaultZone=http://eureka-peer1:1011/eureka,http://eureka-peer2:1012/eureka
```



@EnableDiscoveryClient启用服务发现客户端

```java
@EnableDiscoveryClient
@SpringBootApplication
public class ConsumerMsApplication {

  @Bean
  public RestTemplate restTemplate() {
      return new RestTemplate();
  }

  public static void main(String[] args) {
      SpringApplication.run(ConsumerMsApplication.class, args);
  }
}
```

写一个Controller消费provider-ms提供的服务

```java
@RestController
public class ConsumerDemoController {
    @Autowired
    private LoadBalancerClient loadBalancerClient;

    @Autowired
    private RestTemplate restTemplate;

    @RequestMapping("consume")
    public String consumeService(){
        ServiceInstance serviceInstance = loadBalancerClient.choose("provider-ms");
        String url = "http://"+serviceInstance.getHost()+":"+serviceInstance.getPort()+"/"+"services";
        System.out.println(serviceInstance.getUri().toString());
        String result = restTemplate.getForObject(url, String.class);
        return result;
    }
}
```



这里引入了LoadBalancerClient，通过choose方法负载均衡选择一个“provider-ms”的服务实例，再通过restTemplate调用，服务实例的信息存储在ServiceInstance对象中，可以获取到主机名，端口号等信
启动ConsumerMsApplication，观察eureka-peer1,eureka-peer2，consumer-ms已经注册了
{%asset_img 06.png%}

{%asset_img 07.png%}

此时调用刚才写的consume接口，发现已经能够将注册中心的两个服务实例展示出来了，说明服务间调用成功了

{% asset_img 08.png %}

# 三eureka高可用

{% asset_img 09.png %}

虽然上面我们以双节点作为例子，但是实际上因负载等原因，我们往往可能需要在生产环境构建多于两个的Eureka Server节点。那么对于如何配置serviceUrl来让集群中的服务进行同步，需要我们更深入的理解节点间的同步机制来做出决策。Eureka Server的同步遵循着一个非常简单的原则：只要有一条边将节点连接，就可以进行信息传播与同步。什么意思呢？不妨我们通过下面的实验来看看会发生什么。场景一：假设我们有3个注册中心，我们将eureka-peer1、eureka-peer2、eureka-peer3各自都将eureka.client.serviceUrl.defaultZone指向另外两个节点。换言之，peer1、peer2、peer3是两两互相注册的。启动三个服务注册中心，并将provider-ms的serviceUrl指向eureka-peer1并启动，可以获得如下图所示的集群效果。

{% asset_img 10.png %}

我们在上述双eureka节点高可用的基础上，增加一个application-peer3.properties,并修改3个properties配置文件，内容如下：

```properties
application-peer1.properties

server.port=1011spring.application.name=registry-ha-ms

eureka.instance.hostname=eureka-peer1eureka.client.serviceUrl.defaultZone=http://eureka-peer2:1012/eureka,http://eureka-peer3:1013/eureka
```



application-peer2.properties

```properties
server.port=1012spring.application.name=registry-ha-ms

eureka.instance.hostname=eureka-peer2eureka.client.serviceUrl.defaultZone=http://eureka-peer1:1011/eureka,http://eureka-peer3:1013/eureka
```

application-peer3.properties

```properties
server.port=1013spring.application.name=registry-ha-ms

eureka.instance.hostname=eureka-peer3eureka.client.serviceUrl.defaultZone=http://eureka-peer1:1011/eureka,http://eureka-peer2:1012/eureka
```



复制一个，改名RegistryHaApplication-peer3 ，其中Program arguments: --spring.profiles.active=peer3![img](file:///C:/Users/Administrator/AppData/Local/Temp/enhtmlclip/Image(10).png)

配置本地Hosts域名解析

```
127.0.0.1 eureka-peer1127.0.0.1 eureka-peer2127.0.0.1 eureka-peer3
```



分别启动RegistryHaApplication-peer1,RegistryHaApplication-peer2,RegistryHaApplication-peer3

{% asset_img 11.png %}

观察eureka-peer1,eureka-peer2,eureka-peer3,每个eureka都已经两两互相注册，且目前没有服务实例注册在注册中心（Instances currently registered with Eureka,No instance available）
{% asset_img 12.png %}

{% asset_img 13.png %}

{% asset_img 14.png %}

试验下单边注册一个服务实例，即provider-ms的serviceUrl指向eureka-peer1修改provider-ms项目的application.properties配置，仅仅配置将provider-ms注册到eureka-peer1,然后启动ProviderApplication
server.port=2103spring.application.name=provider-ms#eureka.client.serviceUrl.defaultZone=http://registry-ms.pay-inner.com:1001/eurekaeureka.client.serviceUrl.defaultZone=http://eureka-peer1:1011/eureka
观察eureka-peer1,eureka-peer2,eureka-peer3发现provider-ms服务，已经通过eureka-peer1将服务实例信息同步给了eureka-peer2,eureka-peer3

{% asset_img 15.png %}

{% asset_img 16.png %}

{% asset_img 17.png %}

进一步，我们再将consumer-ms进行eureka-peer2单边注册

{% asset_img 18.png %}

观察eureka-peer1,eureka-peer2,eureka-peer3,此时eureka-peer2同步consumer-ms的服务实例信息到eureka-perr1,eureka-peer3
{% asset_img 19.png %}

{% asset_img 20.png %}

{% asset_img 21.png %}



然后我们调用下consumer-ms的consume接口，没有问题

{% asset_img 22.png %}

将eureka-peer1 shutdownprovider-ms开始报错，访问不到eureka-peer1

{% asset_img 23.png %}



但这并不影响，consumer-ms调用provider-ms

{% asset_img 24.png %}

将eureka-peer2 shutdownconsumer-ms开始报错，访问不到eureka-peer2

{% asset_img 25.png %}

但丝毫不影响,consumer-ms调用provider-ms,做到了服务注册中心的高可用！（这里有点不严谨，因为即使把所有eureka节点都shutdown，还是可以调用的，服务消费方都会本地缓存服务提供方的地址的，但是如果eureka节点全部shutdown，那么新加入的服务实例就无法感知了）

{% asset_img 26.png %}

不过，不建议每个服务配置单边注册到某个eureka节点，强烈建议，每个服务配置所有eureka节点即provider-ms,consumer-ms的application.properties中的*eureka.client.serviceUrl.defaultZone*配成所有eureka节点

```properties
eureka.client.serviceUrl.defaultZone=http://eureka-peer1:1011/eureka,http://eureka-peer2:1012/eureka,http://eureka-peer3:1013/eureka
```

这个配置的试验就不再赘述，可以自行尝试



示例代码：<https://git.oschina.net/jaychang/spring-cloud-demo.git>