---
title: RSA签名内存溢出OOM分析总结
date: 2018-08-28 15:43:40
categories: Java
tags: 
---

生产上应用了RSA算法进行数据加解密，运行一段时间之后出现OOM。后来分析是RSA签名的一个静态常量`javax.crypto.JceSecurity#verificationResults`不断新增导致的。

# 问题重现 #
JVM参数设置：
```
-Xmx100m
-Xms10m
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=E:/document/temp/oom/heapdump.hprof
```

测试代码如下：
```
    @Test
    public void testOOM() {
        try {
            int size = 10000000;
            for (int i = 0; i < size; i++) {
                Cipher cipher = Cipher.getInstance("RSA", new BouncyCastleProvider());
            }
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        } catch (NoSuchPaddingException e) {
            e.printStackTrace();
        }
    }
```

跑完单元测试，根据jvm参数设置，当OOM的时候打印heap dump到指定目录。分析heap dump文件常用的工具是MAT(Memory Analyzer Tool)。

# 分析内存 #
## 打开headpdump.hprof ##

1. 用MAT打开headpdump.hprof文件，看到占用内存最大的是饼图中的深蓝色部分，点击显示JceSecurity类，那么可以初步断定问题是由这个类导致的。

![](https://i.imgur.com/7kH8ygI.png)

2. 点击Actions下的Dominator Tree可以查看占用内存最大的对象，点击后如图2。可以看到有一个IdentityHashMap存储了大量的BouncyCastleProvider对象
![](https://i.imgur.com/zcG1CcP.jpg)

3. 点击JceSecurity-->List Objects-->with outgoing references显示如图3所示，可以看到是变量名为verificationResults的identityHashMap中存放了大量的BouncyCastleProvier，基本上就已经找到导致问题的原因了
![](https://i.imgur.com/Sl7Zqt6.jpg)

4. 查看源码，可以看到因为verificationResults是静态的，不会被GC，所以随着加密工具类调用的次数增加，verificationResults存储的BouncyCastle也越来越多，最终导致OOM。
javax.crypto.JceSecurity
```
 private static final Map<Provider, Object> verificationResults = new IdentityHashMap();
```

# 解决方案 #

```
    @Test
    public void testOOM() {
        BouncyCastleProvider bouncyCastleProvider = new BouncyCastleProvider();
        try {
            int size = 10000000;
            for (int i = 0; i < size; i++) {
                Cipher cipher = Cipher.getInstance("RSA", bouncyCastleProvider);
            }
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        } catch (NoSuchPaddingException e) {
            e.printStackTrace();
        }
    }
```