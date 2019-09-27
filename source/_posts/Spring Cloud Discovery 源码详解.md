---
title: Spring Cloud Discovery源码详解 
date: 2019-09-26 22:43:40
categories: Spring Cloud
tags: 
- discovery
---

## Spring Cloud 版本说明 ##
Angle -> Brixton -> Camden -> Dalston -> Edgware -> Finchley -> Greenwich
## 注解EnableDiscoveryClient ##
在早期版本中为了将服务发布到注册中心，需要在配置类中增加`@EnableDiscoveryClient`。代码如下
```
@SpringBootApplication
@EnableDiscoveryClient
public class Application {
	private final static Logger logger = LoggerFactory.getLogger(Application.class);

	public static void main(String[] args) {
		SpringApplication.run(Application.class, args);
		logger.info("启动成功");
	}
}
```
- 版本Dalston
主要针对eureka，非eureka可以忽略。打开`spring.factories`：
```
org.springframework.cloud.client.discovery.EnableDiscoveryClient=\
org.springframework.cloud.netflix.eureka.EurekaDiscoveryClientConfiguration
```

- 版本Edgware
`@EnableDiscoveryClient`变成一个可选注释，我们不用这个注解，也不会影响服务注册发现，使用配置`spring.cloud.service-registry.auto-registration.enabled=false`即可禁止服务注册发现功能。见架包`spring-cloud-commons`的`spring.factories`：
```
 org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
……
org.springframework.cloud.client.serviceregistry.AutoServiceRegistrationAutoConfiguration
```
```
@Configuration
@Import(AutoServiceRegistrationConfiguration.class)
@ConditionalOnProperty(value = "spring.cloud.service-registry.auto-registration.enabled", matchIfMissing = true)
public class AutoServiceRegistrationAutoConfiguration {

	@Autowired(required = false)
	private AutoServiceRegistration autoServiceRegistration;

	@Autowired
	private AutoServiceRegistrationProperties properties;

	@PostConstruct
	protected void init() {
		if (autoServiceRegistration == null && this.properties.isFailFast()) {
			throw new IllegalStateException("Auto Service Registration has been requested, but there is no AutoServiceRegistration bean");
		}
	}
}
```
从配置中可以看出，spring容器在查询spring.factories的过程中，会校验服务发现实例，例如：eureka、consul,zookeeper。如果没有示例，则抛出异常：`Auto Service Registration has been requested, but there is no AutoServiceRegistration bean`。


## Discovery源码分析##
源码解析针对SpringCloud版本Edgware
- Discovery源码类链：`AutoServiceRegistrationAutoConfiguration`->`AutoServiceRegistration`->`AbstractAutoServiceRegistration`->`AbstractDiscoveryLifecycle`。类图如下：
![](https://i.imgur.com/xBwtLFm.jpg)

- 源码方法追踪链：
> org.springframework.cloud.client.discovery.AbstractDiscoveryLifecycle#start
org.springframework.cloud.client.serviceregistry.AbstractAutoServiceRegistration#register
 org.springframework.cloud.client.serviceregistry.ServiceRegistry#register

```

public abstract class AbstractDiscoveryLifecycle implements DiscoveryLifecycle,
		ApplicationContextAware, ApplicationListener<EmbeddedServletContainerInitializedEvent> {

	@Override
	public void start() {
		if (!isEnabled()) {
			if (logger.isDebugEnabled()) {
				logger.debug("Discovery Lifecycle disabled. Not starting");
			}
			return;
		}

		// only set the port if the nonSecurePort is 0 and this.port != 0
		if (this.port.get() != 0 && getConfiguredPort() == 0) {
			setConfiguredPort(this.port.get());
		}
		// only initialize if nonSecurePort is greater than 0 and it isn't already running
		// because of containerPortInitializer below
		if (!this.running.get() && getConfiguredPort() > 0) {
			//注册动作，方法为抽象法
			register();
			if (shouldRegisterManagement()) {
				registerManagement();
			}
			this.context.publishEvent(new InstanceRegisteredEvent<>(this,
					getConfiguration()));
			this.running.compareAndSet(false, true);
		}
	}

}


public abstract class AbstractAutoServiceRegistration<R extends Registration> extends AbstractDiscoveryLifecycle implements AutoServiceRegistration {
	@Override
	protected void register() {
		this.serviceRegistry.register(getRegistration());
	}
}


public interface ServiceRegistry<R extends Registration> {
	/**
	 * Register the registration. Registrations typically have information about
	 * instances such as: hostname and port.
	 * @param registration the registraion
	 */
	void register(R registration);
}
```

### 总结 ###
1. DiscoveryClient注册服务是利用了LifeCycle机制，在容器启动时会执行`ServiceRegistry#register()`方法；
2. 会自动寻找`DiscoveryClient`接口的实现用作服务发现；




## 推荐阅读 ##

网址|名称|备注
---|---
https://www.jb51.net/article/138767.htm|Spring Cloud学习教程之DiscoveryClient的深入探究|Springframework的LifeCycle接口<br><br>Discoveryclient实战之redis注册中心
https://www.jianshu.com/p/29efa6eb527b|SpringCloud使用Consul时，服务注销的操作方式|
https://blog.csdn.net/boling_cavalry/article/details/82668480|Spring Cloud源码分析之Eureka篇第三章：EnableDiscoveryClient与EnableEurekaClient的区别(Edgware版本)
https://blog.csdn.net/weixin_34406796/article/details/91425191|我认为SpringCloud中AbstractAutoServiceRegistration的“bug“
https://www.cnblogs.com/davidwang456/p/6734995.html|spring cloud集成 consul源码分析
https://www.springcloud.cc/spring-cloud-dalston.html|SpringCloud官方中文文档
