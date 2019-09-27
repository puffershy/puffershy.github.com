---
title: Spring Cloud Consul源码详解 
date: 2019-09-26 22:43:40
categories: Spring Cloud
tags: 
- consul
---




## SpringCloud源码 ##

- 官方文档：https://www.springcloud.cc/spring-cloud-consul.html
- org.springframework.cloud.consul.serviceregistry.ConsulServiceRegistry#register
- 

## 推荐阅读 ##

网址|名称|备注
---|---
https://www.jb51.net/article/138767.htm|Spring Cloud学习教程之DiscoveryClient的深入探究|Springframework的LifeCycle接口<br><br>Discoveryclient实战之redis注册中心
https://www.jianshu.com/p/29efa6eb527b|SpringCloud使用Consul时，服务注销的操作方式|
https://blog.csdn.net/boling_cavalry/article/details/82668480|Spring Cloud源码分析之Eureka篇第三章：EnableDiscoveryClient与EnableEurekaClient的区别(Edgware版本)
https://blog.csdn.net/weixin_34406796/article/details/91425191|我认为SpringCloud中AbstractAutoServiceRegistration的“bug“
https://www.cnblogs.com/davidwang456/p/6734995.html|spring cloud集成 consul源码分析
https://www.springcloud.cc/spring-cloud-dalston.html|SpringCloud官方中文文档


## 源码 ##
1. org.springframework.cloud.consul.serviceregistry.ConsulServiceRegistryAutoConfiguration
2. org.springframework.cloud.consul.serviceregistry.ConsulAutoServiceRegistrationAutoConfiguration