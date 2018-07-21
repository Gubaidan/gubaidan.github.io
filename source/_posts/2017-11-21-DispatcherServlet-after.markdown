---
layout: post
title: "Spring | DispatcherServlet源码分析(下)"
date: 2017-11-21 22:45
author: "Gubaidan"
header-img: "/Header/l8Qo5NqYG_k.png"
cdn: 'header-on'
tags:
	- Spring
---

# 说明

DispatcherServlet是SpringMVC的核心分发器，它实现了请求分发，是处理请求的入口，本篇将深入源码分析它的请求分发过程。

 回顾DispatcherServlet 继承关系图：

<img style="width:90%;" src="http://p9n2j0ewi.bkt.clouddn.com/PostImg/2017-11-20-DispatcherServlet/servlet.png"/>


Servlet在service方法中进行请求接收与分发，DispatcherServlet的service方法继承自HttpServlet，具体代码如下图所示。

```java
/**
*代码位置：HttpServlet#service（）
**/
protected void service(HttpServletRequest req, HttpServletResponse resp)
        throws ServletException, IOException
    {
    //获取请求方式
    String method = req.getMethod();
	//如果是get ，METHOD_GET 不同的请求方法会执行不同的代码，包括put,delete,post,trace.....
    if (method.equals(METHOD_GET)) {
        long lastModified = getLastModified(req);
        if (lastModified == -1) {
            // servlet doesn't support if-modified-since, no reason
            // to go through further expensive logic
            doGet(req, resp);
        } else {
            long ifModifiedSince = req.getDateHeader(HEADER_IFMODSINCE);
            if (ifModifiedSince < lastModified) {
                // If the servlet mod time is later, call doGet()
                // Round down to the nearest second for a proper compare
                // A ifModifiedSince of -1 will always be less
                maybeSetLastModified(resp, lastModified);
                doGet(req, resp);
            } else {
                resp.setStatus(HttpServletResponse.SC_NOT_MODIFIED);
            }
        }

    } else if (method.equals(METHOD_HEAD)) {
        long lastModified = getLastModified(req);
        maybeSetLastModified(resp, lastModified);
        doHead(req, resp);

   } else if (method.equals(METHOD_POST)) {
        doPost(req, resp);
            
    } else if (method.equals(METHOD_PUT)) {
        doPut(req, resp);
            
    } else if (method.equals(METHOD_DELETE)) {
        doDelete(req, resp);
            
    } else if (method.equals(METHOD_OPTIONS)) {
        doOptions(req,resp);
            
    } else if (method.equals(METHOD_TRACE)) {
        doTrace(req,resp);
           
    } else {
        //
        // Note that this means NO servlet supports whatever
        // method was requested, anywhere on this server.
        //

        String errMsg = lStrings.getString("http.method_not_implemented");
        Object[] errArgs = new Object[1];
        errArgs[0] = method;
        errMsg = MessageFormat.format(errMsg, errArgs);
            
        resp.sendError(HttpServletResponse.SC_NOT_IMPLEMENTED, errMsg);
    }
 }
```

在FrameworkServlet中对这个protected修饰的service方法进行了重写，重写的目的是支持PATCH方式请求，具体代码如下图所示。 

```java
/**
代码位置：FrameworkServlet#service（）
**/
@Override
protected void service(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {

	HttpMethod httpMethod = HttpMethod.resolve(request.getMethod());
    //支持PATCH方法
	if (HttpMethod.PATCH == httpMethod || httpMethod == null) {
		processRequest(request, response);
	}
	else {
        //调用父类service方法
		super.service(request, response);
	}
}
```

上述分析中的doGet、doPost等方法在HttpServlet中没有实际可用的实现，如果要使用这些方法，子类需要重写这些方法，DispatcherServlet没有重写这些方法，在DispatcherServlet的父类FrameworkServlet中进行了重写，看几个重写后的方法代码。

```java
/**
代码位置：FrameworkServlet#doGet（）
**/
@Override
protected final void doGet(HttpServletRequest request, HttpServletResponse response)
		throws ServletException, IOException {
	processRequest(request, response);
}
```

可以看到这些请求都会进入当前FrameworkServlet类的processRequest方法进行处理，具体代码如下图所示。


```java
/**
代码位置：FrameworkServlet#processRequest（）
**/
protected final void processRequest(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {

	long startTime = System.currentTimeMillis();
	Throwable failureCause = null;

    //构造LocaleContext和RequestAttributes 绑定到当前线程
	LocaleContext previousLocaleContext = LocaleContextHolder.getLocaleContext();
	LocaleContext localeContext = buildLocaleContext(request);

	RequestAttributes previousAttributes = RequestContextHolder.getRequestAttributes();
	ServletRequestAttributes requestAttributes = buildRequestAttributes(request, response, previousAttributes);

	WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
	asyncManager.registerCallableInterceptor(FrameworkServlet.class.getName(), new RequestBindingInterceptor());

	initContextHolders(request, localeContext, requestAttributes);

	try {
        //抽象方法 子类实现
		doService(request, response);
	}
	catch (ServletException ex) {
		failureCause = ex;
		throw ex;
	}
	catch (IOException ex) {
		failureCause = ex;
		throw ex;
	}
	catch (Throwable ex) {
		failureCause = ex;
		throw new NestedServletException("Request processing failed", ex);
	}

	finally {
		resetContextHolders(request, previousLocaleContext, previousAttributes);
		if (requestAttributes != null) {
			requestAttributes.requestCompleted();
		}

		if (logger.isDebugEnabled()) {
			if (failureCause != null) {
				this.logger.debug("Could not complete request", failureCause);
			}
			else {
				if (asyncManager.isConcurrentHandlingStarted()) {
					logger.debug("Leaving response open for concurrent processing");
				}
				else {
					this.logger.debug("Successfully completed request");
				}
			}
		}

		publishRequestHandledEvent(request, response, startTime, failureCause);
	}
}
```

FrameworkServlet中的doService是一个抽象方法，DispatcherServlet重写了这个方法，具体代码如下图。 

```java
/**
代码位置：DispatcherServlet#doService（）
**/
@Override
protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
	if (logger.isDebugEnabled()) {
		String resumed = WebAsyncUtils.getAsyncManager(request).hasConcurrentResult() ? " resumed" : "";
		logger.debug("DispatcherServlet with name '" + getServletName() + "'" + resumed +
					" processing " + request.getMethod() + " request for [" + getRequestUri(request) + "]");
	}

	// Keep a snapshot of the request attributes in case of an include,
	// to be able to restore the original attributes after the include.
    //对include请求保存一份快照，等会保存到request 属性中
	Map<String, Object> attributesSnapshot = null;
	if (WebUtils.isIncludeRequest(request)) {
		attributesSnapshot = new HashMap<String, Object>();
		Enumeration<?> attrNames = request.getAttributeNames();
		while (attrNames.hasMoreElements()) {
			String attrName = (String) attrNames.nextElement();
			if (this.cleanupAfterInclude || attrName.startsWith("org.springframework.web.servlet")) {
				attributesSnapshot.put(attrName, request.getAttribute(attrName));
			}
		}
	}

	// Make framework objects available to handlers and view objects.
	request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, getWebApplicationContext());
	request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, this.localeResolver);
	request.setAttribute(THEME_RESOLVER_ATTRIBUTE, this.themeResolver);
	request.setAttribute(THEME_SOURCE_ATTRIBUTE, getThemeSource());

	FlashMap inputFlashMap = this.flashMapManager.retrieveAndUpdate(request, response);
	if (inputFlashMap != null) {
		request.setAttribute(INPUT_FLASH_MAP_ATTRIBUTE, Collections.unmodifiableMap(inputFlashMap));
	}
	request.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, new FlashMap());
	request.setAttribute(FLASH_MAP_MANAGER_ATTRIBUTE, this.flashMapManager);

	try {
        // 分发请求到具体的handle
		doDispatch(request, response);
	}
	finally {
		if (!WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
			// Restore the original attribute snapshot, in case of an include.
			if (attributesSnapshot != null) {
				restoreAttributesAfterInclude(request, attributesSnapshot);
			}
		}
	}
}
```

进入doDispatch方法，这个方法实现了将请求分发到具体Handler、执行拦截器的preHandle方法、调用Handler（编写的Controller）处理具体逻辑、执行拦截器的postHandle方法、处理返回的ModelAndView或异常、执行拦截器的afterCompletion方法，具体代码如下。/** 代码位置：DispatcherServlet#doService（） **/

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
	HttpServletRequest processedRequest = request;
	HandlerExecutionChain mappedHandler = null;
	boolean multipartRequestParsed = false;

	WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

	try {
		ModelAndView mv = null;
		Exception dispatchException = null;

		try {
			processedRequest = checkMultipart(request);
			multipartRequestParsed = (processedRequest != request);

			// 根据实际请求的url查找具体的handle，这些handle被封装在HandlerMethod中，
            //这个HandlerMethod又封装在HandlerExecutionChin中
			mappedHandler = getHandler(processedRequest);
			if (mappedHandler == null || mappedHandler.getHandler() == null) {
				noHandlerFound(processedRequest, response);
				return;
			}

			// 便利HandlerAdapter， 查找支持这个请求的HandlerAdapter
			HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

			// Process last-modified header, if supported by the handler.
			String method = request.getMethod();
			boolean isGet = "GET".equals(method);
			if (isGet || "HEAD".equals(method)) {
				long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
				if (logger.isDebugEnabled()) {
					logger.debug("Last-Modified value for [" + getRequestUri(request) + "] is: " + lastModified);
				}
				if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
					return;
				}
			}
            
			//拦截器的mappedHandler
			if (!mappedHandler.applyPreHandle(processedRequest, response)) {
				return;
			}

			// 调用实际编写的handler处理请求，也就是controller
			mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
            
            .....

	}
```

上图描述中的HandlerMethod和HandlerExecutionChain代码如下所示

```java
/**
*代码位置：HandlerMethod
**/
public class HandlerMethod {
    protected final Log logger = LogFactory.getLog(this.getClass());
    private final Object bean;    //controller实例
    private final BeanFactory beanFactory;
    private final Class<?> beanType;
    private final Method method;   //实际方法
    private final Method bridgedMethod;
    private final MethodParameter[] parameters;  //参数
    private final HandlerMethod resolvedFromHandlerMethod;
    ...
```



```java
/**
*代码位置：HandlerExecutionChain
**/
public class HandlerExecutionChain {

	private static final Log logger = LogFactory.getLog(HandlerExecutionChain.class);

	private final Object handler;   //handlerMethod实例

	private HandlerInterceptor[] interceptors;

	private List<HandlerInterceptor> interceptorList;//拦截器实例

	private int interceptorIndex = -1;
    ...
```




# 总结

首先，Spring框架在启动的时候IOC容器会遍历IOC容器中的所有bean，对标注了@Controller或@RequestMapping注解的类中方法进行遍历，将类和方法上的@RequestMapping注解中的value值进行合并，使用@RequestMapping注解的相关参数值(如value、method等)封装一个RequestMappingInfo，将这个Controller实例、方法及方法参数信息(类型、注解等)封装到HandlerMethod中，然后以RequestMappingInfo为key，HandlerMethod为value存到一个以Map为结构的handlerMethods中。

接着，将@RequestMapping注解中的value(即请求路径)值取出，即url，然后以url为key，以RequestMappingInfo为value，存到一个以Map为结构的urlMap属性中。

s客户端发起请求的时候，根据请求的URL到urlMap中查找，找到RequestMappingInfo，然后根据RequestMappingInfo到handlerMethods中查找，找到对应的HandlerMethod，接着将HandlerMethod封装到HandlerExecutionChain；接着遍历容器中所有HandlerAdapter实现类，找到支持这次请求的HandlerAdapter，如RequestMappingHandlerAdapter，然后执行SpringMVC拦截器的前置方法(preHandle方法)，然后对请求参数解析及转换，然后(使用反射)调用具体Controller的对应方法返回一个ModelAndView对象，执行拦截器的后置方法(postHandle方法)，然后对返回的结果进行处理，最后执行afterCompletion方法。

> 参考：《Spring技术内幕》、《精通Spring 4.x 企业应用开发实战》 

 

 

 

 