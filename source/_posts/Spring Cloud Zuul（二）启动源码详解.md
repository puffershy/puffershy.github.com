---
title: Spring Cloud Zuul（二）启动源码详解
date: 2019-07-10 22:43:40
categories: Spring Cloud
tags: 
- zuul
---

spring zuul启动加载原理


org.springframework.cloud.netflix.zuul.web.ZuulHandlerMapping
org.springframework.cloud.netflix.zuul.web.ZuulController
com.netflix.zuul.http.ZuulServlet
com.netflix.zuul.ZuulRunner
com.netflix.zuul.FilterProcessor
com.netflix.zuul.FilterLoader
com.netflix.zuul.filters.FilterRegistry
org.springframework.cloud.netflix.zuul.ZuulFilterInitializer
org.springframework.cloud.netflix.zuul.ZuulServerAutoConfiguration.ZuulFilterConfiguration#zuulFilterInitializer