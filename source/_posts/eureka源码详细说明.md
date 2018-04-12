---
title: Eureka源码详解
date: 2018-04-11 22:43:40
categories: Spring Cloud
tags: 
- redis
- memcache
- tair
---

----------
![](https://i.imgur.com/xPl8okD.png)
Eureka Server是对等集群，其他Server也需要同步这一注册信息。与zookeeper相比，Eureka并不追求很强的一致性，而是认为A（可用性）和P（分区容错性）更重要。

# 1. eureka server 启动 #
spring eureka server启动类org.springframework.cloud.netflix.eureka.server.EurekaServerInitializerConfiguration原样覆盖了eureka servlet启动逻辑。
eureka servlet启动主要做以下几件事：
1. 从相邻eureka节点复制注册列表
2. 生成evictionTimer（定时器）进行Renew（服务续约），默认30秒发送心跳，1分钟就是2次，并设置eureka server状态为up(上线)
3. 注册所有监控统计监听
4. 触发事件EurekaRegistryAvailableEvent（Eureka注册中心启动事件），EurekaServerStartedEvent(Server启动事件)

其中spring并没有去实现EurekaRegistryAvailableEvent（Eureka注册中心启动事件）和EurekaServerStartedEvent(Server启动事件)，所以在可以针对这两事件进行业务扩展。譬如发邮件等通知信息等。
后期扩展如下：
```
	@Component
	public class EurekaEventListener{

		@EventListener
		public void listen(EurekaRegistryAvailableEvent event){
			 InstanceInfo instanceInfo = event.getInstanceInfo();  
	         System.out.println(instanceInfo);  
		}

	}
	
```

# 2. Eureka Server事件介绍 #
监听eureka服务中心的一些状态，就需要通过eureka server的事件通知。eureka server一共有5个事件。位于架包spring-cloud-netflix-eureka-server.1.4.2.RELEASE.jar的org.springframework.context.ApplicationEvent下：
1. EurekaInstanceCanceledEvent 服务下线事件
2. EurekaInstanceRegisteredEvent 服务注册事件
3. EurekaInstanceRenewedEvent 服务续约事件
4. EurekaRegistryAvailableEvent Eureka注册中心启动事件
5. EurekaServerStartedEvent Eureka Server启动事件



# 3. Register（服务注册） #
应用向服务中心注册服务的时候是通过调用**com.netflix.eureka.resources.ApplicationResource.addInstance**来应用注册。同时，当应用的服务状态发生变化时，也会调用来更新服务状态。
接口**addInstance**参数*isReplication*用来判断是来自应用的注册（null），还是Eureka Server的注册（true）。

注册服务过程主要做以下几件事：
1. 获取服务失效时间leaseDuration，默认时间90秒（该判断服务器失效时间由客户端传值）
2. 触发应用注册事件EurekaInstanceRegisteredEvent（服务注册事件）
3. 应用注册到服务列表
4. 复制注册信息到其他节点，并排除当前节点

接口核心代码逻辑如下：
1. ApplicationResource类接收Http服务请求，调用PeerAwareInstanceRegistryImpl的register方法。
2. PeerAwareInstanceRegistryImpl完成服务注册后，调用replicateToPeers向其它Eureka Server节点（Peer）做状态同步

![](http://nobodyiam.com/images/2016-06-25/eureka-server-register.png)
服务注册列表保存在**com.netflix.eureka.registry.AbstractInstanceRegistry**的一个嵌套的hash map中：
```
private final ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>> registry
            = new ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>>();
```
- 第一层hash map的key是app name,也就是应用名称
- 第二层hash map的map的key是instance name,也就是实例名字

# 4. Renew（服务续约） #
renew操作由应用定期调用，类似发送心跳heartbeat。主要是告诉服务注册中心应用还活着，避免服务被剔除掉。接入实现如下图：
![](http://nobodyiam.com/images/2016-06-25/eureka-server-renew.png)
通过调用服务注册中心的**com.netflix.eureka.resources.InstanceResource.renewLease**来续约。续约过程主要做一下几件事情：
1. 刷新当前节点的续约时间
2. 触发EurekaInstanceRenewedEvent事件（服务续约事件）
2. 向其他注册中心发送续约请求，并排除当前节点

# 5. Cancel（服务下线） #
cancel（服务下线）操作一般在应用*shut down*的时候调用，用来把自身的服务从Eureka Server中删除，以防客户端调用不存在的服务。接口调用情况如下：
![](http://nobodyiam.com/images/2016-06-25/eureka-server-cancel.png)
服务下线过程主要做以下几件事情：
1. 触发EurekaInstanceCanceledEvent事件（服务下线事件）
2. 当前节点服务下线
3. 向其他注册中心发送服务下线请求，并排除当前节点

# 6. Fetch Registries（获取服务注册列表） #
Fetch Registries操作由服务消费调用，用来获取Eureka Server上注册的服务。为了提高性能，服务列表在Eureka Server会缓存一份，同时每30秒更新一次。
接口`com.netflix.eureka.resources.ApplicationsResource#getContainers()`
![](http://nobodyiam.com/images/2016-06-25/eureka-server-fetch.png)

# 7. Eviction（失效服务剔除） #
Eviction（失效服务剔除）用来定期在Eureka Server检测失效的服务，检测标准就是超过一定时间没有Renew的服务。

默认失效时间为90秒，也就是如果有服务超过90秒没有向Eureka Server发起Renew请求的话，就会被当做失效服务剔除掉。

失效时间可以通过eureka.instance.leaseExpirationDurationInSeconds进行配置，定期扫描时间可以通过eureka.server.evictionIntervalTimerInMs进行配置。

接口实现逻辑见下图：
![](http://nobodyiam.com/images/2016-06-25/eureka-server-evict.png)


# 8. How Peer Replicates(如何复制服务列表) #
在前面的Register、Renew、Cancel接口实现中，我们看到了都会有replicateToPeers操作，这个就是用来做Peer之间的状态同步。

通过这种方式，Service Provider只需要通知到任意一个Eureka Server后就能保证状态会在所有的Eureka Server中得到更新。

具体实现方式其实很简单，就是接收到Service Provider请求的Eureka Server，把请求再次转发到其它的Eureka Server，调用同样的接口，传入同样的参数，除了会在header中标记isReplication=true，从而避免重复的replicate。

# 9.How Peer Nodes are Discovered #
那大家可能会有疑问，Eureka Server是怎么知道有多少Peer的呢？

Eureka Server在启动后会调用EurekaClientConfig.getEurekaServerServiceUrls来获取所有的Peer节点，并且会定期更新。定期更新频率可以通过eureka.server.peerEurekaNodesUpdateIntervalMs配置。

这个方法的默认实现是从配置文件读取，所以如果Eureka Server节点相对固定的话，可以通过在配置文件中配置来实现。

如果希望能更灵活的控制Eureka Server节点，比如动态扩容/缩容，那么可以override getEurekaServerServiceUrls方法，提供自己的实现，比如我们的项目中会通过数据库读取Eureka Server列表。

具体实现如下图所示：
![](http://nobodyiam.com/images/2016-06-25/eureka-server-peer-discovery.png)

# 10.How New Peer Initializes #
最后再来看一下一个新的Eureka Server节点加进来，或者Eureka Server重启后，如何来做初始化，从而能够正常提供服务。

具体实现如下图所示，简而言之就是启动时把自己当做是Service Consumer从其它Peer Eureka获取所有服务的注册信息。然后对每个服务，在自己这里执行Register，isReplication=true，从而完成初始化。
![](http://nobodyiam.com/images/2016-06-25/eureka-server-peer-init.png)

# 11. Service Provider实现细节 #
现在来看下Service Provider的实现细节，主要就是Register、Renew、Cancel这3个操作。

## 11.1 Register ##

Service Provider要对外提供服务，一个很重要的步骤就是把自己注册到Eureka Server上。

这部分的实现比较简单，只需要在启动时和实例状态变化时调用Eureka Server的接口注册即可。需要注意的是，需要确保配置eureka.client.registerWithEureka=true。
![](http://nobodyiam.com/images/2016-06-25/service-provider-register.png)

## 11.2 Renew ##
Renew操作会在Service Provider端定期发起，用来通知Eureka Server自己还活着。 这里有两个比较重要的配置需要注意一下：

1. eureka.instance.leaseRenewalIntervalInSecondsRenew频率。默认是30秒，也就是每30秒会向Eureka Server发起Renew操作。

2. eureka.instance.leaseExpirationDurationInSeconds服务失效时间。默认是90秒，也就是如果Eureka Server在90秒内没有接收到来自Service Provider的Renew操作，就会把Service Provider剔除。

具体实现如下：
![](http://nobodyiam.com/images/2016-06-25/service-provider-renew.png)

## 11.3 Cancel ##
在Service Provider服务shut down的时候，需要及时通知Eureka Server把自己剔除，从而避免客户端调用已经下线的服务。

逻辑本身比较简单，通过对方法标记@PreDestroy，从而在服务shut down的时候会被触发。
![](http://nobodyiam.com/images/2016-06-25/service-provider-cancel.png)

## 11.4 How Eureka Servers are Discovered ##
这里大家疑问又来了，Service Provider是怎么知道Eureka Server的地址呢？

其实这部分的主体逻辑和3.3.7 How Peer Nodes are Discovered几乎是一样的。

也是默认从配置文件读取，如果需要更灵活的控制，可以通过override getEurekaServerServiceUrls方法来提供自己的实现。定期更新频率可以通过eureka.client.eurekaServiceUrlPollIntervalSeconds配置。

![](http://nobodyiam.com/images/2016-06-25/client-discover-eureka-server.png)

# 12.Service Consumer实现细节 #
Service Consumer这块的实现相对就简单一些，因为它只涉及到从Eureka Server获取服务列表和更新服务列表。
## 12.1 Fetch Service Registries ##
Service Consumer在启动时会从Eureka Server获取所有服务列表，并在本地缓存。需要注意的是，需要确保配置eureka.client.shouldFetchRegistry=true。
![](http://nobodyiam.com/images/2016-06-25/service-consumer-fetch-registries.png)
## 12.2 Update Service Registries ##
由于在本地有一份缓存，所以需要定期更新，定期更新频率可以通过eureka.client.registryFetchIntervalSeconds配置。
![](http://nobodyiam.com/images/2016-06-25/service-consumer-update-registries.png)

## 12.3  How Eureka Servers are Discovered ##
Service Consumer和Service Provider一样，也有一个如何知道Eureka Server地址的问题。

其实由于Service Consumer和Service Provider本质上使用的是同一个Eureka客户端，所以这部分逻辑是一样的，这里就不再赘述了。详细信息见3.4.4节。



# 参数配置 #
|参数名|说明|
|:-------------------:|:------------------:|
|xxxx|应用没有renew（服务续约）的情况服务失效时间，单位秒，客户端配置|
|eureka.server.responseCacheAutoExpirationInSeconds|设置时间对象没有被写访问则对象从内存中删除的时间，默认180s，底层构建缓存的时候用到com.netflix.eureka.registry.ResponseCacheImpl.ResponseCacheImpl|



# Eureka Server本地缓存原理 #
Eureka Server会本地缓存一份服务列表，默认是30秒。主要实现类`com.netflix.eureka.registry.ResponseCacheImpl`。缓存结构划分成两级：

    private final ConcurrentMap<Key, Value> readOnlyCacheMap = new ConcurrentHashMap<Key, Value>();
	private final LoadingCache<Key, Value> readWriteCacheMap;

readWriteCacheMap使用guava的缓存框架，具体实例化如下：

        this.readWriteCacheMap =
                CacheBuilder.newBuilder().initialCapacity(1000)
						//设置时间对象没有被写访问则对象从内存中删除
                        .expireAfterWrite(serverConfig.getResponseCacheAutoExpirationInSeconds(), TimeUnit.SECONDS)
                        .removalListener(new RemovalListener<Key, Value>() {
                            @Override
                            public void onRemoval(RemovalNotification<Key, Value> notification) {
                                Key removedKey = notification.getKey();
                                if (removedKey.hasRegions()) {
                                    Key cloneWithNoRegions = removedKey.cloneWithoutRegions();
                                    regionSpecificKeys.remove(cloneWithNoRegions, removedKey);
                                }
                            }
                        })
						//实现自动加载
                        .build(new CacheLoader<Key, Value>() {
                            @Override
                            public Value load(Key key) throws Exception {
								//从实例中
                                if (key.hasRegions()) {
                                    Key cloneWithNoRegions = key.cloneWithoutRegions();
                                    regionSpecificKeys.put(cloneWithNoRegions, key);
                                }
                                Value value = generatePayload(key);
                                return value;
                            }
                        });
首先从readOnlyCacheMap获取，如果没有命中，则从调用readWriteCacheMap中获取。
这里涉及到常量配置：

参数名|说明|备注
---|---|----
eureka.server.useReadOnlyResponseCache|Eureka Server服务列表是否使用缓存|ResponseCacheImpl.ResponseCacheImpl()，如果设置为true，则会生成一个刷新缓存的任务
eureka.server.responseCacheAutoExpirationInSeconds|Eureka Server服务列表缓存久未读失效时间|ResponseCacheImpl.ResponseCacheImpl()
eureka.server.disableTransparentFallbackToOtherRegion|是否获取其他区的服务列表|AbstractInstanceRegistry.getApplications()
eureka.server.responseCacheUpdateIntervalMs|Eureka Server刷新服务列表缓存时间，默认30s| ResponseCacheImpl#ResponseCacheImpl()


# Eureka Server 的坑 #
## Eureka 缓存 ##
Eureka的wiki上有一句话，大意是一个服务启动后最长可能需要2分钟时间才能被其它服务感知到，但是文档并没有解释为什么会有这2分钟。其实这是由三处缓存 + 一处延迟造成的。
首先，Eureka对HTTP响应做了缓存。在Eureka的”控制器”类`ApplicationResource`的109行可以看到有一行

    String payLoad = responseCache.get(cacheKey);
的调用，该代码所在的`getApplication()`方法的功能是响应客户端查询某个服务信息的HTTP请求：
    
    String payLoad = responseCache.get(cacheKey); // 从cache中拿响应数据
    
    if (payLoad != null) {
       logger.debug("Found: {}", appName);
       return Response.ok(payLoad).build();
    } else {
       logger.debug("Not Found: {}", appName);
       return Response.status(Status.NOT_FOUND).build();
    }

上面的代码中，responseCache引用的是ResponseCache类型，该类型是一个接口，其get()方法首先会去缓存中查询数据，如果没有则生成数据返回（即真正去查询注册列表），且缓存的有效时间为30s。也就是说，客户端拿到Eureka的响应并不一定是即时的，大部分时候只是缓存信息。

其次，Eureka Client对已经获取到的注册信息也做了30s缓存。即服务通过eureka客户端第一次查询到可用服务地址后会将结果缓存，下次再调用时就不会真正向Eureka发起HTTP请求了。

**再次， 负载均衡组件Ribbon也有30s缓存。**Ribbon会从上面提到的Eureka Client获取服务列表，然后将结果缓存30s。

最后，如果你并不是在Spring Cloud环境下使用这些组件(Eureka, Ribbon)，你的服务启动后并不会马上向Eureka注册，而是需要等到第一次发送心跳请求时才会注册。心跳请求的发送间隔也是30s。（Spring Cloud对此做了修改，服务启动后会马上注册）

以上这四个30秒正是官方wiki上写服务注册最长需要2分钟的原因。

## 服务注册信息不会被二次传播 ##
如果Eureka A的peer指向了B, B的peer指向了C，那么当服务向A注册时，B中会有该服务的注册信息，但是C中没有。也就是说，如果你希望只要向一台Eureka注册其它所有实例都能得到注册信息，那么就必须把其它所有节点都配置到当前Eureka的peer属性中。这一逻辑是在`PeerAwareInstanceRegistryImpl#replicateToPeers()`方法中实现的：

	private void replicateToPeers(Action action, String appName, String id,
                                  InstanceInfo info /* optional */,
                                  InstanceStatus newStatus /* optional */, boolean isReplication) {
        Stopwatch tracer = action.getTimer().start();
        try {
            if (isReplication) {
                numberOfReplicationsLastMin.increment();
            }
            // 如果这条注册信息是其它Eureka同步过的则不会再继续传播给自己的peer节点
            if (peerEurekaNodes == Collections.EMPTY_LIST || isReplication) {
                return;
            }

            for (final PeerEurekaNode node : peerEurekaNodes.getPeerEurekaNodes()) {
                // 不要向自己发同步请求
                if (peerEurekaNodes.isThisMyUrl(node.getServiceUrl())) {
                    continue;
                }
                replicateInstanceActionsToPeers(action, appName, id, info, newStatus, node);
            }
        } finally {
            tracer.stop();
        }
    }
