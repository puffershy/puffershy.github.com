---
title: shiro框架简介
date: 2018-04-11 22:43:40
categories: shiro
tags: 
---

# shiro过滤器简介 #
shiro内置过滤器，用来控制认证授权

过滤器名称|过滤器类名|描述
---|---|---
anon|org.apache.shiro.web.filter.authc.AnonymousFilter|例子/admins/**=anon 没有参数，表示可以匿名使用
authc|org.apache.shiro.web.filter.authc.FormAuthenticationFilter|例如/admins/user/**=authc表示需要认证(登录)才能使用，没有参数 
authcBasic|org.apache.shiro.web.filter.authc.BasicHttpAuthenticationFilter|例如/admins/user/**=authcBasic没有参数表示httpBasic认证 
perms|org.apache.shiro.web.filter.authz.PermissionsAuthorizationFilter|例子/admins/user/**=perms[user:add:*],参数可以写多个，多个时必须加上引号，并且参数之间用逗号分割，例如/admins/user/**=perms["user:add:*,user:modify:*"]，当有多个参数时必须每个参数都通过才通过，想当于isPermitedAll()方法
port|org.apache.shiro.web.filter.authz.PortFilter|例子/admins/user/**=port[8081],当请求的url的端口不是8081是跳转到schemal://serverName:8081?queryString,其中schmal是协议http或https等，serverName是你访问的host,8081是url配置里port的端口，queryString是你访问的url里的？后面的参数
rest|org.apache.shiro.web.filter.authz.HttpMethodPermissionFilter|例子/admins/user/**=rest[user],根据请求的方法，相当于/admins/user/**=perms[user:method] ,其中method为post，get，delete等
roles|org.apache.shiro.web.filter.authz.RolesAuthorizationFilter|例子/admins/user/**=roles[admin],参数可以写多个，多个时必须加上引号，并且参数之间用逗号分割，当有多个参数时，例如admins/user/**=roles["admin,guest"],每个参数通过才算通过，相当于hasAllRoles()方法
ssl|org.apache.shiro.web.filter.authz.SslFilter|例子/admins/user/**=ssl没有参数，表示安全的url请求，协议为https
user|org.apache.shiro.web.filter.authc.UserFilter|例如/admins/user/**=user没有参数表示必须存在用户，当登入操作时不做检查
logout|org.apache.shiro.web.filter.authc.LogoutFilter|登出

其中认证是否已登录FormAuthenticationFilter
认证是否授权（角色、权限)RolesAuthorizationFilter,PermissionsAuthorizationFilter(这里只介绍RolesAuthorizationFilter)。
二者的核心都是继承AccessControlFilter（其详细的类图，后面有介绍）。

# AccessControlFilter #
AccessControlFilter几个重要的方法：
1. onPreHandle
判断认证是否通过，以及后续处理。
```
    public boolean onPreHandle(ServletRequest request, ServletResponse response, Object mappedValue) throws Exception {
        return isAccessAllowed(request, response, mappedValue) || onAccessDenied(request, response, mappedValue);
    }
```

2. isAccessAllowed
判断认证是否通过（FormAuthenticationFilter中是认证是否已登录，RolesAuthorizationFilter是认证是否已授权）

3. onAccessDenied
认证失败的后续处理

4. isLoginRequest
判断是否是登录请求,登录地址在配置文件中配置
```
<!--未登录状态下访问authc则进入loginUrl配置的路径-->
 <property name="loginUrl" value="/login"></property>
 <!--登录成功后跳转的默认地址-->
 <property name="successUrl" value="/main"></property>
 <!--如果您请求的资源不再您的权限范围，则跳转到/400请求地址 -->
 <property name="unauthorizedUrl" value="400"></property>
```
```
    protected boolean isLoginRequest(ServletRequest request, ServletResponse response) {
        return pathsMatch(getLoginUrl(), request);
    }
```
5. saveRequest
保存当前请求
6. redirectToLogin
重定向到登录页面
7. saveRequestAndRedirectToLogin
保存当前请求并重定向到登录页面


# FormAuthenticationFilter #

![](https://i.imgur.com/vCJt1j8.jpg)

<center>FormAuthenticationFilter类图</center>

认证登录流程按照继承关系。
AccessControlFilter
```
public boolean onPreHandle(ServletRequest request, ServletResponse response, Object mappedValue) throws Exception {
       return isAccessAllowed(request, response, mappedValue) || onAccessDenied(request, response, mappedValue);
   }
```
AuthenticatingFilter（判断是否认证通过，将调用父类的isAccessAllowed）
```
@Override
protected boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) {
   return super.isAccessAllowed(request, response, mappedValue) ||
           (!isLoginRequest(request, response) && isPermissive(mappedValue));
}
```
AuthenticationFilter（在这里真正判断是否已登录）
```
protected boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) {
   Subject subject = getSubject(request, response);
   return subject.isAuthenticated();
}
```
登录的话，直接跳转，如果未登录：
FormAuthenticationFilter
```
protected boolean onAccessDenied(ServletRequest request, ServletResponse response) throws Exception {
   if (isLoginRequest(request, response)) {
       if (isLoginSubmission(request, response)) {
           if (log.isTraceEnabled()) {
               log.trace("Login submission detected.  Attempting to execute login.");
           }
           return executeLogin(request, response);
       } else {
           if (log.isTraceEnabled()) {
               log.trace("Login page view.");
           }
           //allow them to see the login page ;)
           return true;
       }
   } else {
       if (log.isTraceEnabled()) {
           log.trace("Attempting to access a path which requires authentication.  Forwarding to the " +
                   "Authentication url [" + getLoginUrl() + "]");
       }

       saveRequestAndRedirectToLogin(request, response);
       return false;
   }
}
```
首先判断是否是登录请求，如果是get形式登录页面跳转继续过滤器链，如果是post表单提交执行executeLogin方法，
将走realm中的doGetAuthenticationInfo方法。
ps：执行executeLogin方法过程中会生成AuthenticationToken，具体代码：
```
protected AuthenticationToken createToken(String username, String password,
                                         ServletRequest request, ServletResponse response) {
   boolean rememberMe = isRememberMe(request);
   String host = getHost(request);
   return createToken(username, password, rememberMe, host);
}
```
默认传username，password，所以前台页面name属性要一一对应。
登录成功或失败会走onLoginSuccess或onLoginFailure

如果不是登录请求则保存当前请求跳转至登录页面，如果想封装ajax请求的shiro处理，可自定义自己的FormAuthenticationFilter


# RolesAuthorizationFilter #

![](https://i.imgur.com/nZyMcjD.jpg)
<center>RolesAuthorizationFilter类图</center>

认证授权流程按照继承关系
AccessControlFilter
```
public boolean onPreHandle(ServletRequest request, ServletResponse response, Object mappedValue) throws Exception {
       return isAccessAllowed(request, response, mappedValue) || onAccessDenied(request, response, mappedValue);
   }
```
RolesAuthorizationFilter（认证是否有权限，shiro默认是需要所有角色才认证通过的）
```
public boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) throws IOException {

      Subject subject = getSubject(request, response);
      String[] rolesArray = (String[]) mappedValue;

      if (rolesArray == null || rolesArray.length == 0) {
          //no roles specified, so nothing to check - allow access.
          return true;
      }

      Set<String> roles = CollectionUtils.asSet(rolesArray);
      return subject.hasAllRoles(roles);
  }
```
AuthorizationFilter（认证角色未通过后的处理）
```
protected boolean onAccessDenied(ServletRequest request, ServletResponse response) throws IOException {

      Subject subject = getSubject(request, response);
      // If the subject isn't identified, redirect to login URL
      if (subject.getPrincipal() == null) {
          saveRequestAndRedirectToLogin(request, response);
      } else {
          // If subject is known but not authorized, redirect to the unauthorized URL if there is one
          // If no unauthorized URL is specified, just return an unauthorized HTTP status code
          String unauthorizedUrl = getUnauthorizedUrl();
          //SHIRO-142 - ensure that redirect _or_ error code occurs - both cannot happen due to response commit:
          if (StringUtils.hasText(unauthorizedUrl)) {
              WebUtils.issueRedirect(request, response, unauthorizedUrl);
          } else {
              WebUtils.toHttp(response).sendError(HttpServletResponse.SC_UNAUTHORIZED);
          }
      }
      return false;
  }
```
如果未登录，保存当前请求跳转至登录页面，否则判断是否有配置认证权限未通过须要跳转的请求，如果配置了则跳转，否则抛出错误码SC_UNAUTHORIZED（401），可在web.xml中捕获该错误码自己定义要跳转的页面。

## BasicHttpAuthenticationFilter过滤器 ##

所有请求都会先经过Filter,所以我们可以继承`BasicHttpAuthenticationFilter`，并且重写鉴权的方法。
代码的执行流程preHandle->isAccessAllowed->isLoginAttempt->executeLogin；

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