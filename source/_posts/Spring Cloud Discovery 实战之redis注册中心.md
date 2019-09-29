---
title: Spring Cloud Discovery源码详解 
date: 2019-09-26 22:43:40
categories: Spring Cloud
tags: 
- discovery
---

## 导读 ##
上一篇文章《Spring Cloud Discovery 源码详解》介绍了Spring Cloud Discovery的原理，接下来就可以使用这个原理，模拟注册中心。

## 架包依赖 ##
```
<!-- spring cloud 公共架包依赖-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-commons</artifactId>
        </dependency>

        <!--spring redis依赖-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
```

