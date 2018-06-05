---
title: shiro框架简介
date: 2018-04-11 22:43:40
categories: shiro
tags: 
---

https://blog.csdn.net/u011535541/article/details/49178309/

# shiro组件介绍 #

组件名称|描述
---|---
DefaultWebSecurityManager|
DefaultWebSubjectFactory|工厂方法
DefaultAdvisorAutoProxyCreator|如果想要shiro支持注解，则必须声明
LifecycleBeanPostProcessor|声明周期处理器，用于保证shiro内部lifecycle函数的bean执行


## DefaultWebSecurityManager ##
SecurityManager作为shiro的主要入口之一。几乎所有相关的权限操作，都由他代理了。
1. 所有配置入口；
2. 一个接口就可以实现，验证的操作（登录，退出）、授权（授权访问指定资源，角色）、session管理，相当于这些操作的门面（门面模式，也叫外观模式）；

![](https://i.imgur.com/iy3ZyaY.jpg)

类层级关系：

类名|说明
---|---
CachingSecurityManager|该抽象实现支持缓存
RealmSecurityManager|该抽象实现支持Realm(真正的数据交互、处理操作)
AuthenticatingSecurityManager|该抽象实现验证器方法
AuthorizingSecurityManager|该抽象实现授权方法
SessionsSecurityManager|该抽象实现session管理
DefaultSecurityManager|默认SecurityManager具体实现类
DefaultWebSecurityManager|



## DefaultAdvisorAutoProxyCreator ##
```
@Bean
@DependsOn("lifecycleBeanPostProcessor")
public DefaultAdvisorAutoProxyCreator defaultAdvisorAutoProxyCreator() {
        DefaultAdvisorAutoProxyCreator defaultAdvisorAutoProxyCreator = new DefaultAdvisorAutoProxyCreator();
		// 强制使用cglib，防止重复代理和可能引起代理出错的问题
        defaultAdvisorAutoProxyCreator.setProxyTargetClass(true);
        return defaultAdvisorAutoProxyCreator;
}
```


**BasicHttpAuthenticationFilter过滤器**

所有请求都会先经过Filter,所以我们可以继承`BasicHttpAuthenticationFilter`，并且重写鉴权的方法。
代码的执行流程preHandle->isAccessAllowed->isLoginAttempt->executeLogin；
重要方法介绍：

方法名|作用
---|---
onPreHandle|判断认证是否通过，以及后续处理上代码。AccessControlFilter
isAccessAllowed|是否允许访问，用于访问权限限制。判断认证是否通过（FormAuthenticationFilter中是认证是否已登录，RolesAuthorizationFilter是认证是否已授权。AuthenticatingFilter
onAccessDenied|认证失败的后续处理
isLoginRequest|判断是否是登录请求
preHandle|
isLoginAttempt|判断用户是否想要登入，默认检测header是否包含Authorization字段
executeLogin|执行登录，其中getSubject(request, response).login(token);这一步就是提交给了realm进行处理
onLoginSuccess|登录成功，则走该方法
onLoginFailure|登录失败，则走该方法

