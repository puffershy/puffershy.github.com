---
title: shiro DefaultWebSecurityManager 源码解析
date: 2018-04-11 22:43:40
categories: shiro
tags: 
---

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


