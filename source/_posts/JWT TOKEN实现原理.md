---
title: JWT TOKEN 实现原理
date: 2018-06-01 11:02:40
categories: 微服务
tags: 
- JWT
- 
---
# 一.前言 #
用户中心的模块设计的时候，采用前后端分离架构，会碰到登录凭证的问题。经过一番了解决定采用JWT的方式生成token。

# 二.优点介绍 #
JSON Web Tokens简称jwt，是rest接口的一种安全策略。本身有很多的优势：
1. 解决跨域问题：这种基于Token的访问策略可以克服cookies的跨域问题。
2. 服务端无状态可以横向扩展，Token可完成认证，无需存储Session。
3. 系统解耦，Token携带所有的用户信息，无需绑定一个特定的认证方案，只需要知道加密的方法和密钥就可以进行加密解密，有利于解耦。
4. 防止跨站点脚本攻击，没有cookie技术，无需考虑跨站请求的安全问题。 

# 三.缺点介绍 #

# 四.原理简介 #
JSON Web Tokens的格式组成，jwt是一段被base64编码过的字符序列，用点号分隔，一共由三部分组成，头部header，消息体playload和签名sign。一个jwt类似的于这样：`xxxxx.yyyy.zzzz`
1. **jwt的头部Header是json格式：**
```
{
    "typ":"JWT",
    "alg":"HS256",
    "exp":1491066992916
}
```
其中typ是type的简写，代表该类型是JWT类型，加密方式声明是HS256，exp代表当前时间。

2. **jwt的消息体Playload**
Playload是JWT主要的信息存储部分，其中包含了许多中的声明（claims）。
Claims 的实体一般包含用户和一些元数据，这些 claims 分成三种类型：reserved, public, 和 private claims。
- （保留声明）reserved claims ：预定义的 一些声明，并不是强制的但是推荐，它们包括 iss (issuer), exp (expiration time), sub (subject),aud(audience) 等。
- （公有声明）public claims : 这个部分可以随便定义，但是要注意和 IANA JSON Web Token 冲突。
- （私有声明）private claims : 这个部分是共享被认定信息中自定义部分。

一个Playload可以是这样的：
```
{
    "userid":"123456",
    "iss":"companyName"
}
```
消息体的具体字段可根据业务需要自行定义和添加，只需在解密的时候注意拿字段的key值获取value。

3. **签名sign的生成**
最后是签名，签名的生成是把header和playload分别使用base64url编码,接着用’.‘把两个编码后的字符串连接起来，再把这拼接起来的字符串配合密钥进行HMAC SHA-256算法加密，最后再次base64编码下，这就拿到了签名sign. 最后把header和playload和sign用’.‘ 连接起来就生成了整个JWT。
一个使用 HMAC SHA256 的例子如下:
```
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret)
```


# 五.校验简介 #
整个jwt的结构是由header.playload.sign连接组成，只有sign是用密钥加密的，而所有的信息都在header和playload中可以直接获取，sign的作用只是校验header和playload的信息是否被篡改过，所以jwt不能保护数据，但以上的特性可以很好的应用在权限认证上。

1. 加密
比如要加密验证的是userid字段，首先按前面的格式组装json消息头header和消息体playload，按header.playload组成字符串，再根据密钥和HS256加密header.playload得到sign签名，最后得到jwtToken为header.playload.sign，在http请求中的url带上参数想后端服务请求认证。
2. 解密
后端服务校验jwtToken是否有权访问接口服务，进行解密认证，如校验访问者的userid，首先 
用将字符串按.号切分三段字符串，分别得到header和playload和sign。然后将header.playload拼装用密钥和HAMC SHA-256算法进行加密然后得到新的字符串和sign进行比对，如果一样就代表数据没有被篡改，然后从头部取出exp对存活期进行判断，如果超过了存活期就返回空字符串，如果在存活期内返回userid的值。

# 六.JSON Web Token工作流程 #
在用户使用证书或者账号密码登入的时候一个 JSON Web Token 将会返回，同时可以把这个 JWT 存储在local storage、或者 cookie 中，用来替代传统的在服务器端创建一个 session 返回一个 cookie。
![](https://i.imgur.com/fkkuzZz.jpg)
当用户想要使用受保护的路由时候，应该要在请求得时候带上 JWT ，一般的是在 header 的 Authorization 使用 Bearer 的形式。
这是一种无状态的认证机制，用户的状态从来不会存在服务端，在访问受保护的路由时候回校验 HTTP header 中 Authorization 的 JWT，同时 JWT 是会带上一些必要的信息，不需要多次的查询数据库。

这种无状态的操作可以充分的使用数据的 APIs，甚至是在下游服务上使用，这些 APIs 和哪服务器没有关系，因此，由于没有 cookie 的存在，所以在不存在跨域（CORS, Cross-Origin Resource Sharing）的问题。

# 七.代码示例 #
java代码的加密解密示例
```
package api.test.util;

import java.io.UnsupportedEncodingException;
import java.security.InvalidKeyException;
import java.security.NoSuchAlgorithmException;

import javax.crypto.Mac;
import javax.crypto.SecretKey;
import javax.crypto.spec.SecretKeySpec;

import org.apache.commons.codec.binary.Base64;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import lombok.extern.slf4j.Slf4j;
import net.sf.json.JSONObject;

/**
 * jwt加解密实现
 * 
 * @author zhengsc
 */
@Slf4j
public class TokenUtil {

    private String ISSUER = "companyName"; // 机构

    private String APP_SECRET_KEY = "secret"; // 密钥

    private long MAX_TOKEN_AGE = 1800; // 存活期

    /**
     * 生成userId的accessToken
     * 
     * @param userid
     * @return
     */
    public String generateAccessToken(String userid) {
        JSONObject claims = new JSONObject();
        claims.put("iss", ISSUER);
        claims.put("userid", userid);
        String accessToken = sign(claims, APP_SECRET_KEY);
        return accessToken;
    }

    /**
     * 解密程序返回userid
     * 
     * @param token
     * @return
     */
    public String verifyToken(String token) {
        String userid = "";
        try {
            String[] splitStr = token.split("\\.");
            String headerAndClaimsStr = splitStr[0] + "." +splitStr[1];
            String veryStr = signHmac256(headerAndClaimsStr, APP_SECRET_KEY);
            // 校验数据是否被篡改
            if (veryStr.equals(splitStr[2])) {
                String header = new String(Base64.decodeBase64(splitStr[0]),"UTF-8");
                JSONObject head = JSONObject.fromObject(header);
                long expire = head.getLong("exp") * 1000L;
                long currentTime = System.currentTimeMillis();
                if (currentTime <= expire){ // 验证accessToken的有效期
                    String claims = new String(Base64.decodeBase64(splitStr[1]),"UTF-8");
                    JSONObject claim = JSONObject.fromObject(claims);
                    userid = (String) claim.get("userid");
                }
            }
        } catch (UnsupportedEncodingException e) {
            log.error(e.getMessage(), e);
        }

        return userid;
    }

    /**
     * 组装加密结果jwt返回
     * 
     * @param claims
     * @param appSecretKey
     * @return
     */
    private String sign(JSONObject claims, String appSecretKey) {
        String headerAndClaimsStr = getHeaderAndClaimsStr(claims);
        String signed256 = signHmac256(headerAndClaimsStr, appSecretKey);
        return headerAndClaimsStr + "." + signed256;
    }

    /**
     * 拼接请求头和声明
     * 
     * @param claims
     * @return
     */
    private String getHeaderAndClaimsStr(JSONObject claims) {
        JSONObject header = new JSONObject();
        header.put("alg", "HS256");
        header.put("typ", "JWT");
        header.put("exp", System.currentTimeMillis() + MAX_TOKEN_AGE * 1000L);
        String headerStr = header.toString();
        String claimsStr = claims.toString();
        String headerAndClaimsStr = Base64.encodeBase64URLSafeString(headerStr.getBytes()) + "."
                + Base64.encodeBase64URLSafeString(claimsStr.getBytes());
        return headerAndClaimsStr;
    }

    /**
     * 将headerAndClaimsStr用SHA1加密获取sign
     * 
     * @param headerAndClaimsStr
     * @param appSecretKey
     * @return
     */
    private String signHmac256(String headerAndClaimsStr, String appSecretKey) {
        SecretKey key = new SecretKeySpec(appSecretKey.getBytes(), "HmacSHA256");
        String result = null;
        try {
            Mac mac;
            mac = Mac.getInstance(key.getAlgorithm());
            mac.init(key);
            result = Base64.encodeBase64URLSafeString(mac.doFinal(headerAndClaimsStr.getBytes()));
        } catch (NoSuchAlgorithmException | InvalidKeyException e) {
            log.error(e.getMessage(), e);
        }
        return result;
    }

}
```
# 八.扩展阅读 #
1. [基于Token的WEB后台认证机制](https://www.cnblogs.com/xiekeli/p/5607107.html "基于Token的WEB后台认证机制")；
2. [JWT 在前后端分离中的应用与实践](https://cnodejs.org/topic/557844a8e3cc2f192486a8ff "JWT 在前后端分离中的应用与实践")；