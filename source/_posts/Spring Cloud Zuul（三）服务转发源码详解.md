---
title: Spring Cloud Zuul（三）服务转发源码详解
date: 2019-07-11 11:11:11
categories: Spring Cloud
tags: 
- zuul
---

推荐阅读：https://www.jb51.net/article/128511.htm

超时：https://blog.csdn.net/qq_24267619/article/details/78653095
springcloud组件优化：https://www.cnblogs.com/justlove/p/8028361.html

官方超时文档：http://cloud.spring.io/spring-cloud-netflix/single/spring-cloud-netflix.html#_zuul_timeouts
高可用：https://www.cnblogs.com/ityouknow/archive/2018/01/31/8391593.html

### HttpClientRibbonCommandFactory ###

```
org.springframework.cloud.netflix.zuul.filters.route.apache.HttpClientRibbonCommandFactory
……

@Override
public HttpClientRibbonCommand create(final RibbonCommandContext context) {
	//获取所有ZuulFallbackProvider,即当Zuul调用失败的降级方法
	ZuulFallbackProvider zuulFallbackProvider = getFallbackProvider(context.getServiceId());

	final String serviceId = context.getServiceId();

	//创建处理请求转发类，该类会利用,Apache的Http client进行请求的转发
	final RibbonLoadBalancingHttpClient client = this.clientFactory.getClient(
			serviceId, RibbonLoadBalancingHttpClient.class);
	client.setLoadBalancer(this.clientFactory.getLoadBalancer(serviceId));

	//将降级方法、处理请求转发类、以及其他一些内容包装成HttpClientRibbonCommand(这个类继承了HystrixCommand)
	return new HttpClientRibbonCommand(serviceId, client, context, zuulProperties, zuulFallbackProvider,
			clientFactory.getClientConfig(serviceId));
}
```

### AbstractRibbonCommand ###

```
org.springframework.cloud.netflix.zuul.filters.route.support.AbstractRibbonCommand#run

	@Override
	protected ClientHttpResponse run() throws Exception {
		final RequestContext context = RequestContext.getCurrentContext();

		RQ request = createRequest();
		RS response = this.client.executeWithLoadBalancer(request, config);

		context.set("ribbonResponse", response);

		// Explicitly close the HttpResponse if the Hystrix command timed out to
		// release the underlying HTTP connection held by the response.
		//
		if (this.isResponseTimedOut()) {
			if (response != null) {
				response.close();
			}
		}

		return new RibbonHttpResponse(response);
	}
```
可以看到在run方法中会调用client的executeWithLoadBalancer方法，通过上面介绍我们知道client指的是RibbonLoadBalancingHttpClient，而RibbonLoadBalancingHttpClient里面并没有executeWithLoadBalancer方法。(这里面会最终调用它的父类AbstractLoadBalancerAwareClient的executeWithLoadBalancer方法。)

### AbstractLoadBalancerAwareClient ###

```
com.netflix.client.AbstractLoadBalancerAwareClient

    public T executeWithLoadBalancer(final S request, final IClientConfig requestConfig) throws ClientException {
        LoadBalancerCommand<T> command = buildLoadBalancerCommand(request, requestConfig);

        try {
			//command.submit()方法主要是创建了一个Observable(RxJava),并且为这个Observable设置了重试次数
			//这个Observable最终会回调AbstractLoadBalancerAwareClient.this.execute()方法。
            return command.submit(
                new ServerOperation<T>() {
                    @Override
                    public Observable<T> call(Server server) {
                        URI finalUri = reconstructURIWithServer(server, request.getUri());
                        S requestForServer = (S) request.replaceUri(finalUri);
                        try {
                            return Observable.just(AbstractLoadBalancerAwareClient.this.execute(requestForServer, requestConfig));
                        } 
                        catch (Exception e) {
                            return Observable.error(e);
                        }
                    }
                })
                .toBlocking()
                .single();
        } catch (Exception e) {
            Throwable t = e.getCause();
            if (t instanceof ClientException) {
                throw (ClientException) t;
            } else {
                throw new ClientException(e);
            }
        }
        
    }

    protected LoadBalancerCommand<T> buildLoadBalancerCommand(final S request, final IClientConfig config) {
		//创建一个RetryHandler，这个很重要它是用来决定利用RxJava的Observable是否进行重试的标准。
		RequestSpecificRetryHandler handler = getRequestSpecificRetryHandler(request, config);

		//创建一个LoadBalancerCommand，这个类用来创建Observable,以及根据RetryHandler来判断是否进行重试操作。
		LoadBalancerCommand.Builder<T> builder = LoadBalancerCommand.<T>builder()
				.withLoadBalancerContext(this)
				.withRetryHandler(handler)
				.withLoadBalancerURI(request.getUri());
		customizeLoadBalancerCommandBuilder(request, config, builder);
		return builder.build();
	}
```
下面针对于每一块内容做详细说明：
首先getRequestSpecificRetryHandler(request, requestConfig);这个方法其实调用的是RibbonLoadBalancingHttpClient的getRequestSpecificRetryHandler方法，这个方法主要是返回一个RequestSpecificRetryHandler

```
org.springframework.cloud.netflix.ribbon.apache.RibbonLoadBalancingHttpClient#getRequestSpecificRetryHandler

	@Override
	public RequestSpecificRetryHandler getRequestSpecificRetryHandler(
			RibbonApacheHttpRequest request, IClientConfig requestConfig) {

		//这个很关键，请注意该类构造器中的前两个参数的值,正因为一开始我也忽略了这两个值，所以后续给我造成一定的干扰。
		return new RequestSpecificRetryHandler(false, false, RetryHandler.DEFAULT,
				requestConfig);
	}

```
接下来创建LoadBalancerCommand并将上一步获得的RequestSpecificRetryHandler作为参数内容。
最后调用LoadBalancerCommand的submit方法。该方法内容太长具体代码细节就不在这里贴出了，按照我个人的理解，只贴出相应的伪代码：

```
com.netflix.loadbalancer.reactive.LoadBalancerCommand#submit

public Observable<T> submit(final ServerOperation<T> operation) {
    final ExecutionInfoContext context = new ExecutionInfoContext();
    
    if (listenerInvoker != null) {
        try {
            listenerInvoker.onExecutionStart();
        } catch (AbortExecutionException e) {
            return Observable.error(e);
        }
    }

	//相同server的重试次数(去除首次请求)
    final int maxRetrysSame = retryHandler.getMaxRetriesOnSameServer();
	//集群内其他Server的重试个数
    final int maxRetrysNext = retryHandler.getMaxRetriesOnNextServer();

    // Use the load balancer
	//创建一个Observable(RxJava),selectServer()方法是利用Ribbon选择一个Server，并将其包装成Observable
    Observable<T> o = 
            (server == null ? selectServer() : Observable.just(server))
            .concatMap(new Func1<Server, Observable<T>>() {
                @Override
                // Called for each server being selected
                public Observable<T> call(Server server) {
                    context.setServer(server);
                    final ServerStats stats = loadBalancerContext.getServerStats(server);
                    
                    // Called for each attempt and retry
					//这里会回调submit方法入参ServerOperation类的call方法，
                    Observable<T> o = Observable
                            .just(server)
                            .concatMap(new Func1<Server, Observable<T>>() {
                                @Override
                                public Observable<T> call(final Server server) {
                                    context.incAttemptCount();
                                    loadBalancerContext.noteOpenConnection(stats);
                                    
                                    if (listenerInvoker != null) {
                                        try {
                                            listenerInvoker.onStartWithServer(context.toExecutionInfo());
                                        } catch (AbortExecutionException e) {
                                            return Observable.error(e);
                                        }
                                    }
                                    
                                    final Stopwatch tracer = loadBalancerContext.getExecuteTracer().start();
                                    
                                    return operation.call(server).doOnEach(new Observer<T>() {
                                        private T entity;
                                        @Override
                                        public void onCompleted() {
                                            recordStats(tracer, stats, entity, null);
                                            // TODO: What to do if onNext or onError are never called?
                                        }

                                        @Override
                                        public void onError(Throwable e) {
                                            recordStats(tracer, stats, null, e);
                                            logger.debug("Got error {} when executed on server {}", e, server);
                                            if (listenerInvoker != null) {
                                                listenerInvoker.onExceptionWithServer(e, context.toExecutionInfo());
                                            }
                                        }

                                        @Override
                                        public void onNext(T entity) {
                                            this.entity = entity;
                                            if (listenerInvoker != null) {
                                                listenerInvoker.onExecutionSuccess(entity, context.toExecutionInfo());
                                            }
                                        }                            
                                        
                                        private void recordStats(Stopwatch tracer, ServerStats stats, Object entity, Throwable exception) {
                                            tracer.stop();
                                            loadBalancerContext.noteRequestCompletion(stats, entity, exception, tracer.getDuration(TimeUnit.MILLISECONDS), retryHandler);
                                        }
                                    });
                                }
                            });
                    
                    if (maxRetrysSame > 0) 
                        o = o.retry(retryPolicy(maxRetrysSame, true));
                    return o;
                }
            });
        
    if (maxRetrysNext > 0 && server == null) 
        o = o.retry(retryPolicy(maxRetrysNext, false));
    
    return o.onErrorResumeNext(new Func1<Throwable, Observable<T>>() {
        @Override
        public Observable<T> call(Throwable e) {
			//转发请求失败时，会进入此方法。通过此方法进行判断是否超过重试次数maxRetrysSame、maxRetrysNext。			

            if (context.getAttemptCount() > 0) {
                if (maxRetrysNext > 0 && context.getServerAttemptCount() == (maxRetrysNext + 1)) {
                    e = new ClientException(ClientException.ErrorType.NUMBEROF_RETRIES_NEXTSERVER_EXCEEDED,
                            "Number of retries on next server exceeded max " + maxRetrysNext
                            + " retries, while making a call for: " + context.getServer(), e);
                }
                else if (maxRetrysSame > 0 && context.getAttemptCount() == (maxRetrysSame + 1)) {
                    e = new ClientException(ClientException.ErrorType.NUMBEROF_RETRIES_EXEEDED,
                            "Number of retries exceeded max " + maxRetrysSame
                            + " retries, while making a call for: " + context.getServer(), e);
                }
            }
            if (listenerInvoker != null) {
                listenerInvoker.onExecutionFailed(e, context.toFinalExecutionInfo());
            }
            return Observable.error(e);
        }
    });
}

```
operation.call()方法最终会调用RibbonLoadBalancingHttpClient的execute方法，该方法内容如下：

```
org.springframework.cloud.netflix.ribbon.apache.RibbonLoadBalancingHttpClient#execute

	@Override
	public RibbonApacheHttpResponse execute(RibbonApacheHttpRequest request,
			final IClientConfig configOverride) throws Exception {
		final RequestConfig.Builder builder = RequestConfig.custom();
		IClientConfig config = configOverride != null ? configOverride : this.config;
		builder.setConnectTimeout(
				config.get(CommonClientConfigKey.ConnectTimeout, this.connectTimeout));
		builder.setSocketTimeout(
				config.get(CommonClientConfigKey.ReadTimeout, this.readTimeout));
		builder.setRedirectsEnabled(
				config.get(CommonClientConfigKey.FollowRedirects, this.followRedirects));

		final RequestConfig requestConfig = builder.build();
		request = getSecureRequest(request, configOverride);
		final HttpUriRequest httpUriRequest = request.toRequest(requestConfig);
		final HttpResponse httpResponse = this.delegate.execute(httpUriRequest);
		return new RibbonApacheHttpResponse(httpResponse, httpUriRequest.getURI());
	}
```