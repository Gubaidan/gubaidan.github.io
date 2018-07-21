---
layout: post
title: "Spring | DispatcherServlet源码分析(上)"
date: 2017-11-20 19:13
author: "Gubaidan"
header-img: "/Header/l8Qo5NqYG_k.png"
cdn: 'header-on'
tags:
	- Spring
---

# 说明

DispatcherServlet是SpringMVC的核心分发器，它实现了请求分发，是处理请求的入口，本篇将深入源码分析它的初始化阶段。

 下图为DispatcherServlet 继承关系图：

<img style="width:90%;" src="http://p9n2j0ewi.bkt.clouddn.com/PostImg/2017-11-20-DispatcherServlet/servlet.png"/>

既然DispatcherServlet是一个Servlet，那么初始化的时候一定会执行init方法，查看源码发现DispatcherServlet的init方法继承自HttpServletBean，具体代码如下所示。

```java
/**
* 方法位置：HttpServletBean#init() 
*/
@Override
public final void init() throws ServletException {
	if (logger.isDebugEnabled()) {
		logger.debug("Initializing servlet '" + getServletName() + "'");
	}

	// Set bean properties from init parameters.
	try {
		//从web.xml中获取初始化参数并生成一个ServletConfigPropertyValues对象
		PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
		//将当前对象封装为一个BeanWrapper，此时bena属性还未设置
		BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
		ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
		bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
		//初始化BeanWrapper
		initBeanWrapper(bw);
		//将构造器的属性实例设置到BeanWrapper准备下一步包装
		bw.setPropertyValues(pvs, true);
	}
	catch (BeansException ex) {
		if (logger.isErrorEnabled()) {
			logger.error("Failed to set bean properties on servlet '" + getServletName() + "'", ex);
		}
		throw ex;
	}

	//空实现，可由子类实现
	initServletBean();

	if (logger.isDebugEnabled()) {
		logger.debug("Servlet '" + getServletName() + "' configured successfully");
	}
}
```

上述代码具体加载的web.xml初始化参数如下。

```xml
<!-- springmvc前端控制器 -->
<servlet>
    <servlet-name>springmvc</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <!-- contextConfigLocation配置springmvc加载的配置文件（配置处理器映射器、适配器等等） 如果不配置contextConfigLocation，默认加载的是/WEB-INF/servlet名称-serlvet.xml（springmvc-servlet.xml） -->
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath*:spring/springmvc.xml</param-value>
    </init-param>
</servlet>
<servlet-mapping>
    <servlet-name>springmvc</servlet-name>
    <url-pattern>/</url-pattern>
</servlet-mapping>
```

上面示例配置中，配置了一个 contextConfigLocation参数，该参数用于构造SpringMVC上下文。

对于initServletBean方法，FrameworkServlet中进行了重写，具体代码如下图所示（就是init()方法中空实现的那个）。

```java
/**
* 方法位置：FrameworkServlet#initServletBean() 
*/
@Override
protected final void initServletBean() throws ServletException {
	getServletContext().log("Initializing Spring FrameworkServlet '" + getServletName() + "'");
	if (this.logger.isInfoEnabled()) {
		this.logger.info("FrameworkServlet '" + getServletName() + "': initialization started");
	}
	long startTime = System.currentTimeMillis();

	try {
		//初始化WebApplicationContext，此处的WebApplicationContext最终会放入一个servletcontext
		this.webApplicationContext = initWebApplicationContext();
            //空实现，子类可重写
			initFrameworkServlet();
		}
	catch (ServletException ex) {
			this.logger.error("Context initialization failed", ex);
			throw ex;
		}
	catch (RuntimeException ex) {
			this.logger.error("Context initialization failed", ex);
			throw ex;
		}

	if (this.logger.isInfoEnabled()) {
		long elapsedTime = System.currentTimeMillis() - startTime;
		this.logger.info("FrameworkServlet '" + getServletName() + "': initialization completed in " +
					elapsedTime + " ms");
	}
}
```

initWebApplicationContext方法主要代码如下图所示。 

```java
/**
* 方法位置：FrameworkServlet#initServletBean() 
*/
protected WebApplicationContext initWebApplicationContext() {
    //在getServletContext 中查找WebApplicationContext，如果web.xml配置了contextLoaderListener
    //和contetConfigLocation则rootContext 是由xml配置了contextLoaderListener 监听的上下文
	WebApplicationContext rootContext =
				WebApplicationContextUtils.getWebApplicationContext(getServletContext());
	WebApplicationContext wac = null;
	//DispatcherServlet的构造方法有一个是由WebApplicationContext进行创建的构造方法，如果使用这种
    //方式，则由下面的代码生成实例
	if (this.webApplicationContext != null) {
		// A context instance was injected at construction time -> use it
		wac = this.webApplicationContext;
		if (wac instanceof ConfigurableWebApplicationContext) {
			ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
			if (!cwac.isActive()) {
				// The context has not yet been refreshed -> provide services such as
				// setting the parent context, setting the application context id, etc
				if (cwac.getParent() == null) {
					// The context instance was injected without an explicit parent -> set
					// the root application context (if any; may be null) as the parent
					cwac.setParent(rootContext);
				}
				configureAndRefreshWebApplicationContext(cwac);
			}
		}
	}
	if (wac == null) {
		// 如果执行到这里wac还是为空，再次到ServletContext中查找，没有报错
		wac = findWebApplicationContext();
	}
	if (wac == null) {
		// 如果还是为空，就创建一个局部变量
		wac = createWebApplicationContext(rootContext);
	}

	if (!this.refreshEventReceived) {
		//这里手动刷新，交给子类实现
		onRefresh(wac);
	}

	if (this.publishContext) {
		// 将这里的WebApplicationContext 放入 servletContext
		String attrName = getServletContextAttributeName();
		getServletContext().setAttribute(attrName, wac);
		if (this.logger.isDebugEnabled()) {
			this.logger.debug("Published WebApplicationContext of servlet '" + getServletName() +
						"' as ServletContext attribute with name [" + attrName + "]");
		}
	}

	return wac;
}
```

如果web.xml中配置了如下参数，将会在应用启动的时候初始化一个WebApplicationContext实例，并将其保存在ServletContext中。 

```xml
<listener>
      <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>

```

上图中的代码很容易理解，接下来进入DispatcherServlet的onRefresh方法，具体代码如下图所示。

```java
/**
* 方法位置：DispatcherServlet#onRefresh() 
*/
@Override
protected void onRefresh(ApplicationContext context) {
	initStrategies(context);
}

/**
 * Initialize the strategy objects that this servlet uses.
 * <p>May be overridden in subclasses in order to initialize further strategy objects.
 */
protected void initStrategies(ApplicationContext context) {
	initMultipartResolver(context);
	initLocaleResolver(context);
	initThemeResolver(context);
	initHandlerMappings(context);
	initHandlerAdapters(context);
	initHandlerExceptionResolvers(context);
	initRequestToViewNameTranslator(context);
	initViewResolvers(context);
	initFlashMapManager(context);
}
```

initStrategies方法中的各个方法，从方法名上可以很容易理解其作用，大部分的执行逻辑都是先从WebApplicationContext中查找，找不到的情况下再加载和DispatcherServlet同目录下的DispatcherServlet.properties中的各个策略，如初始化HandlerMapping，具体如下图所示。 

```java
private void initHandlerMappings(ApplicationContext context) {
	this.handlerMappings = null;

	if (this.detectAllHandlerMappings) {
		// Find all HandlerMappings in the ApplicationContext, including ancestor contexts.
		Map<String, HandlerMapping> matchingBeans =
				BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerMapping.class, true, false);
		if (!matchingBeans.isEmpty()) {
			this.handlerMappings = new ArrayList<HandlerMapping>(matchingBeans.values());
			// We keep HandlerMappings in sorted order.
			AnnotationAwareOrderComparator.sort(this.handlerMappings);
		}
	}
	else {
		try {
			HandlerMapping hm = context.getBean(HANDLER_MAPPING_BEAN_NAME, HandlerMapping.class);
			this.handlerMappings = Collections.singletonList(hm);
		}
		catch (NoSuchBeanDefinitionException ex) {
			// Ignore, we'll add a default HandlerMapping later.
		}
	}

	// 加载和DispatcherServlet同目录下的DispatcherServlet.properties中的各个策略
	if (this.handlerMappings == null) {
		this.handlerMappings = getDefaultStrategies(context, HandlerMapping.class);
		if (logger.isDebugEnabled()) {
			logger.debug("No HandlerMappings found in servlet '" + getServletName() + "': using default");
		}
	}
}
```



# 总结

首先，Spring框架在启动的时候IOC容器会遍历IOC容器中的所有bean，对标注了@Controller或@RequestMapping注解的类中方法进行遍历，将类和方法上的@RequestMapping注解中的value值进行合并，使用@RequestMapping注解的相关参数值(如value、method等)封装一个RequestMappingInfo，将这个Controller实例、方法及方法参数信息(类型、注解等)封装到HandlerMethod中，然后以RequestMappingInfo为key，HandlerMethod为value存到一个以Map为结构的handlerMethods中。

接着，将@RequestMapping注解中的value(即请求路径)值取出，即url，然后以url为key，以RequestMappingInfo为value，存到一个以Map为结构的urlMap属性中。

客户端发起请求的时候，根据请求的URL到urlMap中查找，找到RequestMappingInfo，然后根据RequestMappingInfo到handlerMethods中查找，找到对应的HandlerMethod，接着将HandlerMethod封装到HandlerExecutionChain；接着遍历容器中所有HandlerAdapter实现类，找到支持这次请求的HandlerAdapter，如RequestMappingHandlerAdapter，然后执行SpringMVC拦截器的前置方法(preHandle方法)，然后对请求参数解析及转换，然后(使用反射)调用具体Controller的对应方法返回一个ModelAndView对象，执行拦截器的后置方法(postHandle方法)，然后对返回的结果进行处理，最后执行afterCompletion方法。

> 参考：《Spring技术内幕》、《精通Spring 4.x 企业应用开发实战》

 

 

 

 