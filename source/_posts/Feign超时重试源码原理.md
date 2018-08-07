---
title: Feign超时源码
date: 2018-08-06 15:43:40
categories: Spring Cloud
tags: 
- feign
---

----------

feign简而言之就是spring cloud体系里对http的封装，方便使用。对于使用者，不用关心服务的IP地址，以及相关调用逻辑。
Feign集成Ribbon的负载均衡。

## Feign执行入口 ##
feign默认整合ribbon的逻辑。如图
![](https://i.imgur.com/nkS6Avx.png)
feign逻辑执行入口`feign.SynchronousMethodHandler.invoke`，feign的代理类都会由此方法进来（三种代理类包括 原生的feign代理类，整合了ribbon的feign代理类，整合了sleuth）。
![](https://i.imgur.com/bnPPslt.png)
跟进`executeAndDecode`方法到`LoadBalancerFeignClient#execute`方法：
![](https://i.imgur.com/qU4LtL8.png)
每个请求都会获取相应的ClientConfig对象，继续跟进getClientConfig方法会发现一个重要对象:`SpringClientFacotry`
![](https://i.imgur.com/Dukpvqj.png)
SpringClientFactory是一个bean容器，来获取每个feign对应的properties和loadBalancer。仔细查看获取LB(LoadBalance)实例和获取properties的方法，发现LB实例最终从父类NamedContextFactory获取Bean。
![](https://i.imgur.com/ykitu2J.png)
至此，一个关键的配置类被发现了：RibbonClientConfiguration，整个逻辑也就清晰了：RibbonClientConfiguration类是ribbon包的配置类，在feign请求的时候动态配置的。
在每个服务第一次请求的时候，会到抽象工厂类NamedContextFactory中获取当前服务对被调用服务配置，通过getContext方法获取，该方法会先从map中获取:

    private Map<String, AnnotationConfigApplicationContext> contexts = new ConcurrentHashMap<>();

获取不到会执行createContext方法动态加载配置。该方法中用到了spring的注解容器：AnnotationConfigApplicationContext

该容器主要配合@Configuration和注解使用。上述代码中向该容器中注入RibbonClientConfiguration配置类，调用context.refresh()刷新容器，也就拿到了每个服务的配置。

到这里也明白了，为什么feign调用，第二次调用的时间会比第一次用时略少，因为每个feign第一次根据serviceId。
> 在实际开发过程当中，发现服务都成功启动的时候，第一次访问会有报错的情况发生，但是之后又恢复正常访问。主要是Ribbon进行客户端负载均衡的Client并不是在服务启动的时候就初始化好的，而是在调用的时候才会去创建相应的Client，所以第一次调用的耗时不仅仅包含发送HTTP请求的时间，还包含了创建RibbonClient的时间，这样一来如果创建时间速度较慢，同时设置的超时时间又比较短的话，很容易就会出现上面所描述的显现。

加载对应的ribbon配置。具体的ribbon配置有兴趣可以看下ribbon源码。
每个服务第一次调用的时候控制台会打印一下信息，loadBalancer初始化...
```
2017-11-21 17:31:59.787  INFO 4892 --- [   hystrix-ak-1] c.n.l.DynamicServerListLoadBalancer      : DynamicServerListLoadBalancer for client ak initialized: DynamicServerListLoadBalancer:{NFLoadBalancer:name=ak,current list of Servers=[192.168.57.237:8888],Load balancer stats=Zone stats: {},Server stats: [[Server:192.168.57.237:8888;	Zone:defaultZone;	Total Requests:0;	Successive connection failure:0;	Total blackout seconds:0;	Last connection made:Thu Jan 01 08:00:00 CST 1970;	First connection made: Thu Jan 01 08:00:00 CST 1970;	Active Connections:0;	total failure count in last (1000) msecs:0;	average resp time:0.0;	90 percentile resp time:0.0;	95 percentile resp time:0.0;	min resp time:0.0;	max resp time:0.0;	stddev resp time:0.0]
]}ServerList:org.springframework.cloud.netflix.ribbon.eureka.DomainExtractingServerList@51b887dc
```


在我们进行feign这是ribbon的一种饥饿加载机制，ribbon也提供了修改配置：
```
ribbon.eager-load.enabled=true
#指定需要饥饿加载的服务器名
ribbon.eager-load.clients=hello-service, user-service
```

理论上，只要hystrix超时时间合理的话，是可以避免服务启动后第一次调用超时的问题，所以ribbion的饥饿加载机制是可以关闭的，等到真正需要调用的时候再进行初始化对应的ribbon配置。

## feign clien超时设置 ##
上面说明了feign的执行入口`org.springframework.cloud.netflix.feign.ribbon.LoadBalancerFeignClient.execute`传参`options`就是超时配置
![](https://i.imgur.com/6uLhkta.png)
根据源码分析，client请求默认连接超时时间是10秒，读取超时时间是60秒。
可以通过全局修改Options
```
@Bean
public Options getOptions(){
	return new Options(20*1000,30*10000);
}
```
一般不采用这种方式修改修改超时时间，而是通过服务单独配置。
```
feign:
  client:
    config:
      remote-service:           #服务名，填写default为所有服务
        connectTimeout: 1000
        readTimeout: 12000
```

## feign client 重试原理 ##
FeignClient默认不会开启重试机制，需要自定义配置。
如果定义重试次数为N次，那么在所有满足重试的场景，有可能请求的最大次数为N次，因为retryer在构造的时候，初始化attempt为1，把正常请求也算进去了。
FeignClient重试控制类`Retryer`如下：
```
  public static class Default implements Retryer {

	//最大重试次数    
	private final int maxAttempts;
    private final long period;
    private final long maxPeriod;
    int attempt;
    long sleptForMillis;

    public Default() {
      this(100, SECONDS.toMillis(1), 5);
    }

    public Default(long period, long maxPeriod, int maxAttempts) {
      this.period = period;
      this.maxPeriod = maxPeriod;
      this.maxAttempts = maxAttempts;
	  //初始化重试次数
      this.attempt = 1;
    }

	……
}

```
Retryer重试逻辑如下
```
    public void continueOrPropagate(RetryableException e) {
      if (attempt++ >= maxAttempts) {
        throw e;
      }

      long interval;
      if (e.retryAfter() != null) {
		//如果有设置下次重试时间

		//计算下次重试时间和当前时间的差值
        interval = e.retryAfter().getTime() - currentTimeMillis();
        if (interval > maxPeriod) {
		  //如果差值，超过了最大重试周期，则取最大重试周期
          interval = maxPeriod;
        }
        if (interval < 0) {
		  //如果差值小于0，说明已经过设定的重试时间，不用休眠，直接充值
          return;
        }
      } else {
		//没有设置下次重试时间，则获取休眠时间
        interval = nextMaxInterval();
      }
      try {
        Thread.sleep(interval);
      } catch (InterruptedException ignored) {
        Thread.currentThread().interrupt();
      }
      sleptForMillis += interval;
    }

    long nextMaxInterval() {
	  //计算时间间隔，重试最小周期*（1.5（重试次数-1）次方）
      long interval = (long) (period * Math.pow(1.5, attempt - 1));
      return interval > maxPeriod ? maxPeriod : interval;
    }
```
Feign重试在早期的spring Cloud中使用`feign.Retryer.Default#Default()`,重试5次。但feign整合了ribbon，ribbon也有重试能力，此时，就可能会导致行为的混乱。
spring Cloud意识到此问题，因此做了改进，将feign的重试改为`feign.Retryer#NEVER_RETRY`，如果使用Feign的重试，只需要Ribbon的重试配置即可。因此Camden以及以后的版本，Feign的重试可使用如下属性配置：
```
ribbon:
 MaxAutoRetries: 1
 MaxAutoRetriesNextServer: 2
 OkToRetryOnAllOperations: false
```
其ribbon的重试，会在另外一个文章介绍。

FeignClient如果超时，则抛出的异常为`RetryableException`
> FeignClient重试机制，如果服务方的接口不是支持幂等，存在重复请求的情况，所以重试机制的开启要慎重。
> 