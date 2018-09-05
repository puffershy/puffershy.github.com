---
title: RxJava 从入门到放弃再到不离不弃
date: 2018-08-06 15:43:40
categories: Java
tags: 
---

## RxJava是什么 ##
>a library for composing asynchronous and event-based programs using observable sequences for the Java VM
>解释：一个对于构成使用的Java虚拟机观察序列异步和基于事件的程序库

RxJava 是一个响应式编程框架，采用观察者设计模式。所以自然少不了 Observable 和 Subscriber 这两个东东了。
RxJava 是一个开源项目，地址：https://github.com/ReactiveX/RxJava
RxAndroid，用于 Android 开发，添加了 Android 用的接口。地址： https://github.com/ReactiveX/RxAndroid

## 基本概念 ##
- Observable：发射源，在观察者模式中称为“被观察者”或“可观察对象”；
- Observer：接收源，观察者模式中的“观察者”，可接收`Observable`、`Subject`发射的数据；
- Subject：`Subject`是一个比较特殊的对象，即可充当发射源，也可充当接收源。
- Subscriber：“订阅者”，也是接收源，那它跟Observer有什么区别呢？Subscriber实现了Observer接口，比Observer多了一个最重要的方法unsubscribe( )，用来取消订阅，当你不再想接收数据了，可以调用unsubscribe( )方法停止接收，Observer 在 subscribe() 过程中,最终也会被转换成 Subscriber 对象，一般情况下，建议使用Subscriber作为接收源；
- Subscription：Observable调用subscribe( )方法返回的对象，同样有unsubscribe( )方法，可以用来取消订阅事件；
- Action0：RxJava中的一个接口，它只有一个无参call（）方法，且无返回值，同样还有Action1，Action2…Action9等，Action1封装了含有 1 个参的call（）方法，即call（T t），Action2封装了含有 2 个参数的call方法，即call（T1 t1，T2 t2），以此类推；
- Func0：与Action0非常相似，也有call（）方法，但是它是有返回值的，同样也有Func0、Func1…Func9;


## 推荐阅读 ##
[《RxJava 从入门到放弃再到不离不弃》](https://www.daidingkang.cc/2017/05/19/Rxjava/ "《RxJava 从入门到放弃再到不离不弃》")


