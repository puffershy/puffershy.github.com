---
title: Spring Boot异常处理详解
date: 2018-09-13 11:40:40
categories: Spring Boot
tags: 
- 
---

# 前言 #

- Spring Boot版本号1.5.9

本文将讲解在《Spring MVC异常处理》的基础上的Spring Boot做了哪些事情。下图列出Spring Boot中跟MVC异常相关的处理类。涉及到的类：
- org.springframework.boot.autoconfigure.web.DefaultErrorAttributes
- org.springframework.boot.autoconfigure.web.ErrorMvcAutoConfiguration
- org.springframework.boot.autoconfigure.web.ErrorMvcAutoConfiguration.SpelView
- org.springframework.boot.autoconfigure.web.BasicErrorController


# ErrorMvcAutoConfiguration #
```
	@Bean
	public ErrorPageCustomizer errorPageCustomizer() {
		return new ErrorPageCustomizer(this.serverProperties);
	}

	……

	/**
	 * {@link EmbeddedServletContainerCustomizer} that configures the container's error
	 * pages.
	 */
	private static class ErrorPageCustomizer implements ErrorPageRegistrar, Ordered {

		private final ServerProperties properties;

		protected ErrorPageCustomizer(ServerProperties properties) {
			this.properties = properties;
		}

		@Override
		public void registerErrorPages(ErrorPageRegistry errorPageRegistry) { 
			ErrorPage errorPage = new ErrorPage(this.properties.getServletPrefix()
					+ this.properties.getError().getPath());
			errorPageRegistry.addErrorPages(errorPage);
		}

		@Override
		public int getOrder() {
			return 0;
		}

	}

	public class ErrorProperties {
	
		/**
		 * Path of the error controller.
		 */
		@Value("${error.path:/error}")
		private String path = "/error";
	}
```
ErrorMvcAutoConfiguration类加载的时候，添加了一个错误页面`/error`。因为这项配置的存在，如果Spring MVC在处理过程抛出异常到Servlet容器，容器会重定向到这个错误页面`/error`。

怎么扩展呢？
>1. application.properties设置error.path配置异常页面的地址；


# DefaultErrorAttributes #

![](https://i.imgur.com/qzyjZ3e.jpg)

```
org.springframework.boot.autoconfigure.web.ErrorMvcAutoConfiguration

	@Bean
	@ConditionalOnMissingBean(value = ErrorAttributes.class, search = SearchStrategy.CURRENT)
	public DefaultErrorAttributes errorAttributes() {
		return new DefaultErrorAttributes();
	}
```
DefaultErrorAttributes主要功能：
1. 实现ErrorAttributes，提供`org.springframework.boot.autoconfigure.web.ErrorAttributes#getErrorAttributes`，当处理/error错误页面时，需要从该bean中读取错误信息以返回。
2. 实现HandlerExceptionResolver接口，并具有最高优先级。即DispatcherServlet在doDispatch过程中有异常抛出时，先由DefaultErrorAttributes处理。从下面代码中可以发现，DefaultErrorAttributes在处理过程中，是讲ErrorAttributes保存到了request中。事实上，这是DefaultErrorAttributes能够在后面返回Error Attributes的原因，实现HandlerExceptionResolver接口，是DefaultErrorAttributes实现ErrorAttributes接口的手段。

```
	@Override
	public int getOrder() {
		//最高优先级
		return Ordered.HIGHEST_PRECEDENCE;
	}

	@Override
	public ModelAndView resolveException(HttpServletRequest request,
			HttpServletResponse response, Object handler, Exception ex) {
		storeErrorAttributes(request, ex);
		return null;
	}

	private void storeErrorAttributes(HttpServletRequest request, Exception ex) {
		//保存异常信息
		request.setAttribute(ERROR_ATTRIBUTE, ex);
	}
```
## 如何配置？ ##
我们可以继承DefaultErrorAttributes，修改ErrorAttributes。
```
    @Bean
    public DefaultErrorAttributes errorAttributes(){
        return new DefaultErrorAttributes(){
            @Override
            public Map<String, Object> getErrorAttributes(RequestAttributes requestAttributes, boolean includeStackTrace) {
                // return super.getErrorAttributes(requestAttributes, includeStackTrace);

                Map<String, Object> result = Maps.newHashMap();
                result.put("code", "404");
                result.put("display", "未找到");
                return result;
            }
        };
    }
```
我们可以自己实现ErrorAttributes接口，实现自己的Error Attributes方案, 只要配置一个类型为ErrorAttributes的bean，默认的DefaultErrorAttributes就不会被配置。

# BasicErrorController和ErrorView #

```
org.springframework.boot.autoconfigure.web.ErrorMvcAutoConfiguration

	//配置异常控制器
	@Bean
	@ConditionalOnMissingBean(value = ErrorController.class, search = SearchStrategy.CURRENT)
	public BasicErrorController basicErrorController(ErrorAttributes errorAttributes) {
		return new BasicErrorController(errorAttributes, this.serverProperties.getError(),
				this.errorViewResolvers);
	}

	//配置ErrorView
	@Configuration
	@ConditionalOnProperty(prefix = "server.error.whitelabel", name = "enabled", matchIfMissing = true)
	@Conditional(ErrorTemplateMissingCondition.class)
	protected static class WhitelabelErrorViewConfiguration {

		private final SpelView defaultErrorView = new SpelView(
				"<html><body><h1>Whitelabel Error Page</h1>"
						+ "<p>This application has no explicit mapping for /error, so you are seeing this as a fallback.</p>"
						+ "<div id='created'>${timestamp}</div>"
						+ "<div>There was an unexpected error (type=${error}, status=${status}).</div>"
						+ "<div>${message}</div></body></html>");

		@Bean(name = "error")
		@ConditionalOnMissingBean(name = "error")
		public View defaultErrorView() {
			return this.defaultErrorView;
		}

		// If the user adds @EnableWebMvc then the bean name view resolver from
		// WebMvcAutoConfiguration disappears, so add it back in to avoid disappointment.
		@Bean
		@ConditionalOnMissingBean(BeanNameViewResolver.class)
		public BeanNameViewResolver beanNameViewResolver() {
			BeanNameViewResolver resolver = new BeanNameViewResolver();
			resolver.setOrder(Ordered.LOWEST_PRECEDENCE - 10);
			return resolver;
		}

	}
```
BasicErrorController根据Accept头的内容，输出不同格式的错误响应。比如针对浏览器的请求生成html页面，针对其它请求生成json格式的返回。代码如下：
```
	@RequestMapping(produces = "text/html")
	public ModelAndView errorHtml(HttpServletRequest request,
			HttpServletResponse response) {
		HttpStatus status = getStatus(request);
		Map<String, Object> model = Collections.unmodifiableMap(getErrorAttributes(
				request, isIncludeStackTrace(request, MediaType.TEXT_HTML)));
		response.setStatus(status.value());
		ModelAndView modelAndView = resolveErrorView(request, response, status, model);
		return (modelAndView == null ? new ModelAndView("error", model) : modelAndView);
	}

	@RequestMapping
	@ResponseBody
	public ResponseEntity<Map<String, Object>> error(HttpServletRequest request) {
		Map<String, Object> body = getErrorAttributes(request,
				isIncludeStackTrace(request, MediaType.ALL));
		HttpStatus status = getStatus(request);
		return new ResponseEntity<Map<String, Object>>(body, status);
	}
```
WhitelabelErrorView则提供了一个默认的白板错误页面。

## 如何配置？ ##
1. 可以提供自己的名字为error的view，以替换掉默认的白板页面，提供自己想要的样式。
2. 可以继承BasicErrorController或者干脆自己实现ErrorController接口，用来响应/error这个错误页面请求，可以提供更多类型的错误格式等。
