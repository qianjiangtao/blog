---
title: Spring Boot拦截器不生效分析
date: 2018-08-24 11:27:25
categories:
- Spring问题收集
tags:
- Spring

---

![](https://gitee.com/qianjiangtao/my-image/raw/master/blog/2018-11-1-13-55.jpg)

<!--more-->

**问题说明:**

​	如题，最近在项目中使用拦截器判断用户是否登录，因为本项目后台的访问路径是`http://域名/user-center/pointxxx/...`，我想只拦截匹配/point，但是发现拦截器只有设置为`/**`的访问路径,拦截器才能生效，百度谷歌都没查到相关的说明(*可能有没找到*)，后来看源码分析发现是自己的写法有问题，下面就记录下详细分析的过程。



#### 1.最开始错误的写法

```java
@Configuration
public class WebConfiguration extends WebMvcConfigurerAdapter {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LoginInterceptor()).addPathPatterns("/user-center/point/**");
        super.addInterceptors(registry);
    }
}
```



#### 2.分析无法拦截的原因



拦截器无法拦截的原因无非就是两个原因:

- 1.拦截器没生效
- 2.拦截路径写的不正确，无法匹配



使用`/**`拦截器能生效说明拦截器写的没问题，那问题就出现在拦截器配的匹配路径写的有问题，所以检查`addPathPatterns`，发现确实有点问题，URL`/user-center/pointxxx/...`无法匹配`/point/**`,正确的写法应该是`/point*/**`代码如下:

```java
@Configuration
public class WebConfiguration extends WebMvcConfigurerAdapter {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LoginInterceptor()).addPathPatterns("/user-center/point*/**");
        super.addInterceptors(registry);
    }
}
```

再次测试发现还是不行，到底是哪不对，崩溃中.....，于是我想debug跟下源码看下spring拦截器为什么不匹配URL



**源码分析:**

1.首先看下Spring mvc的入口`DispatcherServlet`中的doDispatch()方法

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    ...
    //1.获取HandlerExecutionChain，这个对象中包含HandelMethod和interceptor集合
    mappedHandler = getHandler(processedRequest);
    if (mappedHandler == null || mappedHandler.getHandler() == null) {
        noHandlerFound(processedRequest, response);
        return;
    }
}
```



2.先调用`DispatcherServlet`中的getHandler()，然后代用`AbstractHandlerMapping`中的getHandler()如下:

```java
public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    ...
    //2.获取HandlerExecutionChain
    HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);
    if (CorsUtils.isCorsRequest(request)) {
        CorsConfiguration globalConfig = this.corsConfigSource.getCorsConfiguration(request);
        CorsConfiguration handlerConfig = getCorsConfiguration(handler, request);
        CorsConfiguration config = (globalConfig != null ? globalConfig.combine(handlerConfig) : handlerConfig);
        executionChain = getCorsHandlerExecutionChain(request, executionChain, config);
    }
    return executionChain;
}
```



```java
protected HandlerExecutionChain getHandlerExecutionChain(Object handler, HttpServletRequest request) {
    HandlerExecutionChain chain = (handler instanceof HandlerExecutionChain ?
                                   (HandlerExecutionChain) handler : new HandlerExecutionChain(handler));
    //3
    String lookupPath = this.urlPathHelper.getLookupPathForRequest(request);
    for (HandlerInterceptor interceptor : this.adaptedInterceptors) {
        if (interceptor instanceof MappedInterceptor) {
            MappedInterceptor mappedInterceptor = (MappedInterceptor) interceptor;
            if (mappedInterceptor.matches(lookupPath, this.pathMatcher)) {
                chain.addInterceptor(mappedInterceptor.getInterceptor());
            }
        }
        else {
            chain.addInterceptor(interceptor);
        }
    }
    return chain;
}
```

可以自己看下getLookupPathForRequest()这个方法，主要是通过项目配置的contextPath和请求url分析出lookupPath，如本项目contextPath为/user-center，请求url为：/user-center/pointProduct/index，那么得到的lookupPath为:/pointProduct/index,意思是spring自动把请求url前面的contextPath截取掉



3.使用`MappedInterceptor`的matches()匹配判断是否是需要拦截的

```java
public boolean matches(String lookupPath, PathMatcher pathMatcher) {
    PathMatcher pathMatcherToUse = (this.pathMatcher != null) ? this.pathMatcher : pathMatcher;
    if (this.excludePatterns != null) {
        for (String pattern : this.excludePatterns) {
            if (pathMatcherToUse.match(pattern, lookupPath)) {
                return false;
            }
        }
    }
    if (ObjectUtils.isEmpty(this.includePatterns)) {
        return true;
    }
    else {
        //这里pattern为:/user-center/point*/**,lookupPath通过上面分析为:/pointProduct/index,所以无法匹配
        for (String pattern : this.includePatterns) {
            if (pathMatcherToUse.match(pattern, lookupPath)) {
                return true;
            }
        }
        return false;
    }
}
```



#### 3.问题总结

通过上面的分析可以看出，spring mvc拦截器在拦截的时候会自动对请求路径进行处理，自动截取到请求url中包含contextPath，原因也很简单，springboot中如果配置了默认的contextPath那么本项目中所有的资源如果要访问的话必须加上contextPath,因此没比较再匹配都有的路径,所以正确的写法就很简单了如下:

```java
@Configuration
public class WebConfiguration extends WebMvcConfigurerAdapter {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LoginInterceptor()).addPathPatterns("/point*/**");
        super.addInterceptors(registry);
    }
}
```

