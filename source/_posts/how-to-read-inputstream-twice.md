---
title: 解决 HttpServletRequest 流数据不可重复读
tags:
  - Spring Boot
  - IO流
categories:
  - 技术
copyright: true
date: 2020-05-28 19:22:26
---

在某些业务中可能会需要多次读取 HTTP 请求中的参数，比如说前置的 API 签名校验。这个时候我们可能会在拦截器或者过滤器中实现这个逻辑，但是尝试之后就会发现，如果在拦截器中通过 `getInputStream()` 读取过参数后，在 Controller 中就无法重复读取了，会抛出以下几种异常：
```
HttpMessageNotReadableException: Required request body is missing
IllegalStateException: getInputStream() can't be called after getReader()
```
这个时候需要我们将请求的数据缓存起来。本文会从 ServletRequest 数据封装原理开始详细讲讲如何解决这个问题。如果不想看原理的，可直接阅读 [最佳解决方案](# 最佳解决方案)。

<!-- more -->


## ServletRequest 数据封装原理

平时我们接受 HTTP 请求的参数时，基本是通过 SpringMVC 的包装。

- POST form-data 参数时，直接用实体类，或者直接在 Controller 的方法上把参数填上就可以了，手动则可以通过 `request.getParameter()` 来获取。
- POST json 时，会在实体类上添加 `@RequestBody` 参数或者直接调用 `request.getInputStream()` 获取流数据。

我们可以发现在获取不同数据格式的数据时调用的方法是不同的，但是阅读源码可以发现，其实底层他们的数据来源都是一样的，只是 SpringMVC 帮我们做了一下处理。下面我们就来讲讲 ServletRequest 数据封装的原理。

实际上我们通过 HTTP 传输的参数都会存在 Request 对象的 InputStream 中，这个 Request 对象也就是 ServletRequest 最终的实现，是由 tomcat 提供的。然后针对于不同的数据格式，会在不同的时刻对 InputStream 中的数据进行封装。

### Spring MVC 对不同类型数据的封装

- **GET 请求的数据一般是 Query String，直接在 url 的后面，不需要特殊处理**

- **通过例如 POST、PUT 发送 `multipart/form-data` 格式的数据**

```java
// 源码中适当去除无关代码
// 对于这类数据，SpringMVC 在 DispatchServlet 的 doDispatch() 方法中就会进行处理。具体处理流程如下：
// org.springframework.web.servlet.DispatcherServlet.java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    HttpServletRequest processedRequest = request;
    HandlerExecutionChain mappedHandler = null;
    boolean multipartRequestParsed = false;
    WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
    processedRequest = checkMultipart(request);
    multipartRequestParsed = (processedRequest != request);
    // Determine handler for the current request.
    // other code...
}
// 1. 调用 checkMultipart(request)，当前请求的数据类型是否为 multipart/form-data
protected HttpServletRequest checkMultipart(HttpServletRequest request) throws MultipartException {
    if (this.multipartResolver != null && this.multipartResolver.isMultipart(request)) {
		return this.multipartResolver.resolveMultipart(request);
    }
    return request;
}
//2. 如果是，调用 multipartResolver 的 resolveMultipart(request)，返回一个 StandardMultipartHttpServletRequest 对象。
// org.springframework.web.multipart.support.StandardMultipartHttpServletRequest.java
public StandardMultipartHttpServletRequest(HttpServletRequest request) throws MultipartException {
    this(request, false);
}
public StandardMultipartHttpServletRequest(HttpServletRequest request, boolean lazyParsing) throws MultipartException {
    super(request);
    if (!lazyParsing) {
        parseRequest(request);
    }
}
// 3. 在构造 StandardMultipartHttpServletRequest 对象时，会调用 parseRequest(request)，将 InputStream 中是数据流进行进一步的封装。
// 不贴源码了，主要是对 form-data 数据的封装，包含字段和文件。
```

- **通过例如 POST、PUT 发送 `application/x-www-form-urlencoded` 格式的数据**

```java
// 非 form-data 的数据，会存储在 HttpServletRequest 的 InputStream 中。
// 在第一次调用 getParameterNames() 或 getParameter() 时,
// 会调用 parseParameters() 方法对参数进行封装，从 InputStream 中读取数据，并封装到 Map 中。

//org.apache.catalina.connector.Request.java
public String getParameter(String name) {
    if (!this.parametersParsed) {
        this.parseParameters();
    }
    return this.coyoteRequest.getParameters().getParameter(name);
}
```

- **通过例如 POST、PUT 发送 ` application/json` 格式的数据**

```java
// 数据会直接会存储在 HttpServletRequest 的 InputStream 中，通过 request.getInputStream() 或 getReader() 获取。
```

### 读取参数时出现的问题

现在我们基本已经对 SpringMVC 是如何封装 HTTP 请求参数有了一定的认识。根据之前描述的，我们如果要在拦截器中和 Controller 中重复读取参数时，会出现以下异常：

```
HttpMessageNotReadableException: Required request body is missing
IllegalStateException: getInputStream() can't be called after getReader()
```

这是由于 InputStream 这个流数据的特殊性，在 Java 中读取 InputStream 数据时，内部是通过一个指针的移动来读取一个一个的字节数据的，当读完一遍后，这个指针并不会 reset，因此第二遍读的时候就会出现问题了。而之前讲了，HTTP 请求的参数也是封装在 Request 对象中的 InputStream 里，所以当第二次调用 `getInputStream()` 时会抛出上述异常。具体的问题可以细分成多种情况：

1. **请求方式为 `multipart/form-data`，在拦截器中手动调用 `request.getInputStream()`**

```java
// 上文讲了在 doDispatch() 时就会进行处理，因此这里会取不到值
log.info("input stream content: {}", new String(StreamUtils.copyToByteArray(request.getInputStream())));
```

2. **请求方式为 `application/x-www-form-urlencoded`，在拦截器中手动调用 `request.getInputStream()`**

```java
// 第 1 次可以取到值
log.info("input stream content: {}", new String(StreamUtils.copyToByteArray(request.getInputStream())));
// 第一次执行 getParameter() 会调用 parseParameters()，parseParameters 进一步调用 getInputStream()
// 这里就取不到值了
log.info("form-data param: {}", request.getParameter("a"));
log.info("form-data param: {}", request.getParameter("b"));
```

3. **请求方式为 `application/json`，在拦截器中手动调用 `request.getInputStream()`**

```java
// 第 1 次可以取到值
log.info("input stream content: {}", new String(StreamUtils.copyToByteArray(request.getInputStream())));
// 之后再任何地方再调用 getInputStream() 都取不到值，会抛出异常
```

为了能够多次获取到 HTTP 请求的参数，我们需要将 InputStream 流中的数据缓存起来。

## 最佳解决方案

通过查阅资料，实际上 springframework 自己就有相应的 wrapper 来解决这个问题，在 `org.springframework.web.util` 包下有一个 `ContentCachingRequestWrapper` 的类。这个类的作用就是将 InputStream 缓存到 `ByteArrayOutputStream` 中，通过调用 ``getContentAsByteArray()` 实现流数据的可重复读取。

> ```
> /**
>  * {@link javax.servlet.http.HttpServletRequest} wrapper that caches all content read from
>  * the {@linkplain #getInputStream() input stream} and {@linkplain #getReader() reader},
>  * and allows this content to be retrieved via a {@link #getContentAsByteArray() byte array}.
> 
>  * @see ContentCachingResponseWrapper
>  */
> ```

在使用上，只需要添加一个 Filter，将 `HttpServletRequest` 包装成 `ContentCachingResponseWrapper` 返回给拦截器和 Controller 就可以了。

```java
@Slf4j
@WebFilter(urlPatterns = "/*")
public class CachingContentFilter implements Filter {
    private static final String FORM_CONTENT_TYPE = "multipart/form-data";

    @Override
    public void init(FilterConfig filterConfig) {
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {
        String contentType = request.getContentType();
        if (request instanceof HttpServletRequest) {
            HttpServletRequest requestWrapper = new ContentCachingRequestWrapper((HttpServletRequest) request);
            // #1
            if (contentType != null && contentType.contains(FORM_CONTENT_TYPE)) {
                chain.doFilter(request, response);
            } else {
                chain.doFilter(requestWrapper, response);
            }
            return;
        }
        chain.doFilter(request, response);
    }

    @Override
    public void destroy() {
    }
}

// 添加扫描 filter 注解
@ServletComponentScan
@SpringBootApplication
public class SeedApplication {
    public static void main(String[] args) {
        SpringApplication.run(SeedApplication.class, args);
    }
}
```

在拦截器中，获取请求参数：

```java
// 流数据获取，比如 json
// #2
String jsonBody = IOUtils.toString(wrapper.getContentAsByteArray(), "utf-8");
// form-data 和 urlencoded 数据
String paramA = request.getParameter("paramA");
Map<String,String[]> params = request.getParameterMap();
```

**tips：**

1. 这里需要根据 contentType 做一下区分，遇到 `multipart/form-data` 数据时，不需要 wrapper，会直接通过 MultipartResolver 将参数封装成 Map，当然这也可以灵活的在拦截器中判断。
2. wrapper 在具体使用中，我们可以使用 `getContentAsByteArray()` 来获取数据，并通过 `IOUtils` 转换成 String。尽量不使用 `request.getInputStream()`。因为虽然经过了包装，但是 InputStream 仍然只能读一次，而参数进入 Controller 的方法前 HttpMessageConverter 的参数转换需要调用这个方法，所以把它保留就可以了。

## 总结

遇到这个问题的时候也参考了很多博客，有的使用了 ContentCachingRequestWrapper，也有的自己实现了一个 Wrapper。但是自己实现 Wrapper 的方案，多半是直接在 Wrapper 的构造函数中读取流数据到 byte[] 数据中去，这样在遇到 `multipart/form-data` 这种数据类型的时候就会出现问题了，因为包装在调用 MultipartResolver 之前执行，再次调用的时候就读不到数据了。

所以博主又自己研究了一下 Spring 的源码，实现了这种方案，基本上可以处理多种通用的数据类型了。

## 参考文献

- [怎么重复使用 inputStream?](https://juejin.im/post/5d18c144518825559f46eeb3)
- [Read stream twice](https://stackoverflow.com/questions/9501237/read-stream-twice)
- [解决 request.getInputStream() 只能读取一次的问题](https://www.jianshu.com/p/ec1782e3ba04)
- [Http Servlet request lose params from POST body after read it once](https://stackoverflow.com/questions/10210645/http-servlet-request-lose-params-from-post-body-after-read-it-once)
- [Spring Boot系列之二：一张图看懂请求处理流程](https://juejin.im/post/5b4ac9b3e51d4519277b881b)
## 附录代码

```java
package com.example.seed.common.config;

import lombok.extern.slf4j.Slf4j;
import org.springframework.web.util.ContentCachingRequestWrapper;

import javax.servlet.*;
import javax.servlet.annotation.WebFilter;
import javax.servlet.http.HttpServletRequest;
import java.io.IOException;

/**
 * @author Fururur
 * @date 2020/5/6-14:26
 */
@Slf4j
@WebFilter(urlPatterns = "/*")
public class CachingContentFilter implements Filter {
    private static final String FORM_CONTENT_TYPE = "multipart/form-data";

    @Override
    public void init(FilterConfig filterConfig) {
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        String contentType = request.getContentType();
        if (request instanceof HttpServletRequest) {
            HttpServletRequest requestWrapper = new ContentCachingRequestWrapper((HttpServletRequest) request);
            if (contentType != null && contentType.contains(FORM_CONTENT_TYPE)) {
                chain.doFilter(request, response);
            } else {
                chain.doFilter(requestWrapper, response);
            }
            return;
        }
        chain.doFilter(request, response);
    }

    @Override
    public void destroy() {
    }
}
```
```java
package com.example.seed;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.web.servlet.ServletComponentScan;

@ServletComponentScan
@SpringBootApplication
public class SeedApplication {
    public static void main(String[] args) {
        SpringApplication.run(SeedApplication.class, args);
    }
}
```
```java
@RequestMapping("/query")
public void query(HttpServletRequest request) {
    ContentCachingRequestWrapper wrapper = new ContentCachingRequestWrapper(request);
    log.info("{}", new String(wrapper.getContentAsByteArray()));
}
```