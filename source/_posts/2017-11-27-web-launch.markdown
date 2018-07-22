---
layout: post
title: "Spring | 容器启动梳理"
date: 2017-11-27 22:42
author: "Gubaidan"
header-img: "/Header/l8Qo5NqYG_k.png"
cdn: 'header-on'
tags:
	- Spring
---

# 说明

我们有时候是用下面这种手动的方式启动Spring 容器：

```java
ApplicationContext applicationContext = new ClassPathXmlApplicationContext("application-Context.xml")；
```

除了这种方式，当然还有常见的配置文件启动，在web应用中，配置文件都是自动加载的，上述代码中的方式就不能满足需求了。在web应用中使用Spring，需要在web.xml中添加如下配置。

```xml
<listener>
       <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
<context-param>
     <param-name>contextConfigLocation</param-name>
     <param-value>classpath:application-Context.xml;</param-value>
</context-param>
```

##  ServletContext和ServletContextListene

ServletContext定义了一些方法方便Servlet和Servlet容器进行通讯，在一个web应用中所有的Servlet都公用一个ServletContext，Spring在和web应用结合使用的时候，是将Spring的容器存到ServletContext中的，通俗的说就是将一个ApplicationContext存储到ServletContext的一个Map属性中；而ServletContextListener用于监听ServletContext一些事件。分析就从ContextLoaderListener开始。在web应用启动读取web.xml时，发现配置了ContextLoaderListener，而ContextLoaderListener实现了ServletContextListener接口，因此会执行ContextLoaderListener类中的contextInitialized方法，方法的具体代码如下。

```java
/**
* 代码位置：ServletContextListener
*/
public interface ServletContextListener extends EventListener {

    /**
     * 接收Web应用程序初始化过程中的通知。
     * 在初始化Web应用程序中的任何过滤器或servlet之前，
     * 将通知所有ServletContextListener上下文初始化。
     */
    public void contextInitialized(ServletContextEvent sce);

    /**
     * 接收 ServletContext 关闭的事件信息。
     */
    public void contextDestroyed(ServletContextEvent sce);
}
```



```java
/**
* 代码位置：ContextLoaderListener#contextInitialized()
*/
public void contextInitialized(ServletContextEvent event) {
        initWebApplicationContext(event.getServletContext());
}
```



## initWebApplicationContext

<span id="initWebApplicationContext"></span>

继续进入initWebApplicationContext方法，这个方法在其父类ContextLoader实现，根据方法名可以看出这个方法是用于初始化一个WebApplicationContext，简单理解就是初始化一个Web应用下的Spring容器。方法的具体代码如下。

```java
/**
* 代码位置：ContextLoader#initWebApplicationContext()
*/
public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
	if (servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null) {
		throw new IllegalStateException(
					"Cannot initialize context because there is already a root application context present - " +
					"check whether you have multiple ContextLoader* definitions in your web.xml!");
	}

	Log logger = LogFactory.getLog(ContextLoader.class);
	servletContext.log("Initializing Spring root WebApplicationContext");
	if (logger.isInfoEnabled()) {
		logger.info("Root WebApplicationContext: initialization started");
	}
	long startTime = System.currentTimeMillis();

	try {
		// 将context存储在本地实例变量中，以保证它在ServletContext关闭时可用。
		if (this.context == null) {
			this.context = createWebApplicationContext(servletContext);
		}
		if (this.context instanceof ConfigurableWebApplicationContext) {
			ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
			if (!cwac.isActive()) {
				// 如果容器还未刷新 -> 则设置父容器
				if (cwac.getParent() == null) {
					// 如果context没有显示的设置父容器 ->
                    //如果存在root-webapplication就把它设置为父容器
					ApplicationContext parent = loadParentContext(servletContext);
					cwac.setParent(parent);
				}
				configureAndRefreshWebApplicationContext(cwac, servletContext);
			}
		}
			servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);
	
		ClassLoader ccl = Thread.currentThread().getContextClassLoader();
		if (ccl == ContextLoader.class.getClassLoader()) {
			currentContext = this.context;
		}
		else if (ccl != null) {
			currentContextPerThread.put(ccl, this.context);
		}

		if (logger.isDebugEnabled()) {
			logger.debug("Published root WebApplicationContext as ServletContext attribute with name [" +
					WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE + "]");
		}
		if (logger.isInfoEnabled()) {
			long elapsedTime = System.currentTimeMillis() - startTime;
			logger.info("Root WebApplicationContext: initialization completed in " + elapsedTime + " ms");
		}

		return this.context;
	}
	catch (RuntimeException ex) {
		logger.error("Context initialization failed", ex);
			servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, ex);
		throw ex;
	}
	catch (Error err) {
		logger.error("Context initialization failed", err);
			servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, err);
		throw err;
	}
}
```

方法的第一行就是检查servletContext是否已经存储了一个默认名称的WebApplicationContext，因为在一个应用中Spring容器只能有一个，所以需要校验，至于这个默认的名称为什么是**WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE**，看到后面就会慢慢明白。直接关注重点代码，代码如下。

```java
/**
* 代码位置：ContextLoader#initWebApplicationContext()
*/
if (this.context == null) {
    //这里的context是ContextLoader的一个变量，声明代码如下。
	this.context = createWebApplicationContext(servletContext);
}
```

```java
/**
* 代码位置：ContextLoader私有属性
*/
private WebApplicationContext context; 
```



## createWebApplicationContext

继续进入createWebApplicationContext方法，具体代码如下。

```java
/**
* 代码位置：ContextLoader#createWebApplicationContext()
*/
protected WebApplicationContext createWebApplicationContext(ServletContext sc) {
   Class<?> contextClass = determineContextClass(sc);
   if (!ConfigurableWebApplicationContext.class.isAssignableFrom(contextClass)) {
      throw new ApplicationContextException("Custom context class [" + contextClass.getName() +
            "] is not of type [" + ConfigurableWebApplicationContext.class.getName() + "]");
   }
   return (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);
}
```

这个方法主要用于创建一个WebApplicationContext对象。因为WebApplicationContext只是一个接口，不能创建对象，所以需要找到一个WebApplicationContext接口的实现类，determineContextClass方法就是用于寻找实现类，如果开发人员在web.xml中配置了一个参数名为contextClass，值为WebApplicationContext接口实现类，那就会返回这个配置的实现类Class；如果没有配置，则会返回Spring默认的实现类XmlWebApplicationContext。直接进入determineContextClass方法体，代码如下。

## determineContextClass

```java
/**
* 代码位置：ContextLoader#determineContextClass()
*/
protected Class<?> determineContextClass(ServletContext servletContext) {
	String contextClassName = servletContext.getInitParameter(CONTEXT_CLASS_PARAM);
	if (contextClassName != null) {
		try {
				return ClassUtils.forName(contextClassName, ClassUtils.getDefaultClassLoader());
			}
		catch (ClassNotFoundException ex) {
				throw new ApplicationContextException(
						"Failed to load custom context class [" + contextClassName + "]", ex);
		}
	}
	else {
		contextClassName = defaultStrategies.getProperty(WebApplicationContext.class.getName());
		try {
				return ClassUtils.forName(contextClassName, ContextLoader.class.getClassLoader());
			}
		catch (ClassNotFoundException ex) {
				throw new ApplicationContextException(
						"Failed to load default context class [" + contextClassName + "]", ex);
		}
	}
}
```

上面的代码中使用了defaultStrategies，用于获取Spring默认的WebApplicationContext接口实现类XmlWebApplicationContext.java，它的声明如下。

```java
/**
* 代码位置：ContextLoader#私有属性和静态代码块
*/
private static final String DEFAULT_STRATEGIES_PATH = "ContextLoader.properties";
private static final Properties defaultStrategies;
static {
       try {
            ClassPathResource resource = new ClassPathResource(DEFAULT_STRATEGIES_PATH, ContextLoader.class);
            defaultStrategies = PropertiesLoaderUtils.loadProperties(resource);
        }catch (IOException ex) {
            throw new IllegalStateException("Could not load 'ContextLoader.properties': " + ex.getMessage());
        }
}
```

也就是说在ContextLoader.java的路径下，有一个ContextLoader.properties文件，查找并打开这个文件，文件内容如下。

```java
org.springframework.web.context.WebApplicationContext=org.springframework.web.context.support.XmlWebApplicationContext
```

这里配置了Spring默认的WebApplicationContext接口实现类XmlWebApplicationContext.java。回到createWebApplicationContext方法，WebApplicationContext接口实现类的Class已经找到，然后就是使用构造函数进行初始化完成WebApplicationContext对象创建。 
继续回到 [>> initWebApplicationContext](#initWebApplicationContext) 方法，此时这个context就指向了刚刚创建的WebApplicationContext对象。因为XmlWebApplicationContext间接实现了ConfigurableWebApplicationContext接口，所以将会执行如下代码。

```java
/**
* 代码位置：ContextLoader#initWebApplicationContext（）
*/
if (this.context instanceof ConfigurableWebApplicationContext) {
	ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
	if (!cwac.isActive()) {
		if (cwac.getParent() == null) {
			ApplicationContext parent = loadParentContext(servletContext);
				cwac.setParent(parent);
		}
        configureAndRefreshWebApplicationContext(cwac, servletContext);
	}
}
```

这里关注重点代码configureAndRefreshWebApplicationContext(cwac, servletContext)，configureAndRefreshWebApplicationContext方法具体代码如下。

## configureAndRefreshWebApplicationContext

```java
/**
* 代码位置：ContextLoader#configureAndRefreshWebApplicationContext（）
*/
protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc) {
	if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
        //如果应用程序上下文ID仍是其原始默认值则根据可用信息分配更有用的ID
		String idParam = sc.getInitParameter(CONTEXT_ID_PARAM);
		if (idParam != null) {
			 wac.setId(idParam);
		}else {
			wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
                        ObjectUtils.getDisplayString(sc.getContextPath()));
		}
	}
	wac.setServletContext(sc);
	String configLocationParam = sc.getInitParameter(CONFIG_LOCATION_PARAM);
	if (configLocationParam != null) {
        wac.setConfigLocation(configLocationParam);
    }
    ConfigurableEnvironment env = wac.getEnvironment();
    if (env instanceof ConfigurableWebEnvironment) {
        ((ConfigurableWebEnvironment) env).initPropertySources(sc, null);
    }
    customizeContext(sc, wac);
    wac.refresh();
}
```

还是一样，关注重点代码。看一下CONFIG_LOCATION_PARAM这个常量的值是”contextConfigLocation”，OK，这个就是web.xml中配置applicationContext.xml的。这个参数如果没有配置，在XmlWebApplicationContext中是有默认值的，具体的值如下。

```
public static final String DEFAULT_CONFIG_LOCATION = "/WEB-INF/applicationContext.xml";
```

也就是说，如果没有配置contextConfigLocation参数，将会使用/WEB-INF/applicationContext.xml。关注configureAndRefreshWebApplicationContext方法的最后一行代码wac.refresh()，是不是有点眼熟，继续跟踪代码，refresh方法的具体代码如下。

```java
/**
* 代码位置：AbstractApplicationContext#refresh（）
*/
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        prepareRefresh();
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
        prepareBeanFactory(beanFactory);
        try {
            postProcessBeanFactory(beanFactory);
            invokeBeanFactoryPostProcessors(beanFactory);
            registerBeanPostProcessors(beanFactory);
            initMessageSource();
            initApplicationEventMulticaster();
            onRefresh();
            registerListeners();
            finishBeanFactoryInitialization(beanFactory);
            finishRefresh();
        }catch (BeansException ex) {
            logger.warn("Exception encountered during context initialization - cancelling refresh attempt", ex);
            destroyBeans();
            cancelRefresh(ex);
            throw ex;
        }
    }
}
```

这个refresh [《Spring | AbstractApplicationContext refresh()方法源码解析》](http://gubaidan.top/2017/11/06/2017-11-06-refresh/) 中的refresh方法，用于完成Bean的解析、实例化及注册。  继续分析，回到[>> initWebApplicationContext](#initWebApplicationContext) 方法，将执行如下代码。

```java
servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);
```

可以看到，这里将初始化后的context存到了servletContext中，具体的就是存到了一个Map变量中，key值就是WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE这个常量。

使用Spring的WebApplicationContextUtils工具类获取这个WebApplicationContext方式如下。

```java
WebApplicationContext applicationContext = WebApplicationContextUtils.getWebApplicationContext(request.getServletContext()); 
```



>参考：《Spring技术内幕》、《精通Spring 4.x 企业应用开发实战》