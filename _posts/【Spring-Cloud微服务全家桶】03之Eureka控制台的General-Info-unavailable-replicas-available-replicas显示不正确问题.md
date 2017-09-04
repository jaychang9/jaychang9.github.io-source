---
title: 【Spring Cloud微服务全家桶】03之Eureka控制台的General Info(unavailable-replicas
  available-replicas显示不正确问题)
date: 2017-09-01 17:36:30
tags: Spring Cloud
---

# 前言

之前【Spring Cloud微服务全家桶】之高可用服务注册中心 文章介绍了Eureka Server高可用



# 问题描述

{%asset_img 01.png%}

观察eureka-peer1，发现unavailable-replicas与available-replicas显示信息不正确，按理说所有eureka节点都启动后，eureka-peer3,eureka-peer2应该出现在available-replicas列表

{%asset_img 02.png%}

# 解决方案

修改下每个eureka节点的配置，把自身注册到注册中心

```properties
eureka.client.fetch-registry=true
eureka.client.register-with-eureka=true
eureka.instance.prefer-ip-address=false
```

* eureka-peer1节点的application.properties配置

```properties
server.port=1011
spring.application.name=registry-ha-ms

eureka.instance.hostname=eureka-peer1
eureka.client.serviceUrl.defaultZone=http://eureka-peer2:1012/eureka,http://eureka-peer3:1013/eureka

eureka.client.fetch-registry=true
eureka.client.register-with-eureka=true
eureka.instance.prefer-ip-address=false
```

* eureka-peer2节点的application.properties配置

```properties
server.port=1012
spring.application.name=registry-ha-ms

eureka.instance.hostname=eureka-peer2
eureka.client.serviceUrl.defaultZone=http://eureka-peer1:1011/eureka,http://eureka-peer3:1013/eureka

eureka.client.fetch-registry=true
eureka.client.register-with-eureka=true
eureka.instance.prefer-ip-address=false
```

* eureka-peer3节点的application.properties配置

```properties
server.port=1013
spring.application.name=registry-ha-ms

eureka.instance.hostname=eureka-peer3
eureka.client.serviceUrl.defaultZone=http://eureka-peer1:1011/eureka,http://eureka-peer2:1012/eureka

eureka.client.fetch-registry=true
eureka.client.register-with-eureka=true
eureka.instance.prefer-ip-address=false
```

重启每个eureka节点
发现available-replicas栏可以出现了

{%asset_img 03.png%}

{%asset_img 04.png%}

{%asset_img 05.png%}