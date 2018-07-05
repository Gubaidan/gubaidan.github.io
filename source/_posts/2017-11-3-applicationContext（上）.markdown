---
layout: post
title: "Spring-BeanFactory与ApplicationContext基础"
date: 2017-11-03 16:43
author: "Gubaidan"
header-img: "/Header/hero@2x.jpg"
cdn: 'header-on'
tags:
	- Spring
---

# BeanFactory介绍

BeanFactory 是Spring中两大核心模块（AOP、IOC）之一IOC的基础，与传统的BeanFactory 不同，BeanFactory时一个通用型的Bean工厂，它可以创建和管理这些Bean，所有可以被Spring 容器实例化并管理的Java类都可以称作Bean。

Spring在BeanFactory这个接口的基础上实现了很多类，最常用的是XmlBeanFactory，但在Spring3.2	之后已经被弃用，建议使用XmlBeanDefinitionReader和，DefaultListableBeanFactory代替，Spring对这一继承机构设计的相当精巧。

![BeanFactory](http://p9n2j0ewi.bkt.clouddn.com/PostImg/2017-11-03-applicationContext/BeanFactoryDependence.png)

BeanFactory 接口位于类结构树的顶端，通过其他类和接口的装饰，最后的DefaultBeanFactory就是由其他类和接口不断扩充。

- ListableBeanFactory：该接口定义了访问容器中Bean基本信息的若干方法，如查看Bean的个数、获取某一类型Bean的配置名、查看容器中是否包含某一Bean等。

- HierarchicalBeanFactory：父子及联IOC容器的接口、子接口可以根据接口方法访问父容器。

- ConfiguratableBeanFactory：这是一个重要的接口，增强了IOC容器的可制定性。它定义了设置类装载器、属性编辑器、容器初始化后置处理器等方法。

- AutowireCapableBeanFactory：定义了将容器中的Bean按照某种规则（如名字匹配、类型匹配）进行自动装配的方法。

- SingletonBeanRegister：允许在运行期向容器注册单实例Bean的方法。

- BeanDefinitionRegister：接口提供了向容器注册BeanDefinition的方法。

  

### BeanFactory使用示例

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    <beans id = "pension" class = "com.smart.pension">
        p:name = "zhang"
        p:sex = "nan"
        p:location = "fujian"
    </beans>
</beans
```

```java
public class BeanFactoryT {
    @Test
    public void getBean() throws IOException {
        ResourcePatternResolver resourcePatternResolver = new PathMatchingResourcePatternResolver();
        Resource[] resource = resourcePatternResolver.getResources("com.smart.pension");
        System.out.println(Arrays.toString(resource));

        /**
         * 废弃、不建议使用
         */
//        BeanFactory beanFactory = new XmlBeanFactory(res);
        DefaultListableBeanFactory defaultListableBeanFactory = new DefaultListableBeanFactory();
        XmlBeanDefinitionReader xmlBeanDefinitionReader = new XmlBeanDefinitionReader(defaultListableBeanFactory);
        xmlBeanDefinitionReader.loadBeanDefinitions(resource);
        
        Pension zhang = defaultListableBeanFactory.getBean(Pension.class);
        car.move();
    }
}
```

XmlBeanDefinitionReader 通过Resource装载配置信息并启动IOC容器，这样就可以从beanFactory中的getBean（）方法通过全限定名或者“类名.class” 为参数获取Bean。对于单例的Bean，在第二次getBean时会直接从缓存中读取Bean然后返回。

在Spring容器启动的时候必须为其设置一个日志框架，用来记录启动信息，不然会启动报错。

# ApplicationContext

如果说beanFactory是“心脏”的话，那么ApplicationbContext就是一个完整的“躯体”了，Application由BeanFactory一步步扩充而来，提供了很多实用功能，很多在BeanFactory需要配置实现的功能在ApplicationContext中只需要简单的配置即可以实现。

### ApplicationbContext类结构体系

![application](http://p9n2j0ewi.bkt.clouddn.com/PostImg/2017-11-03-applicationContext/applicationContext.png)



ApplicationContext的主要实现类是FileSystemxmlApplicationContext和ClassPathXmlApplicationContext，前者主要加载存储在文件系统上的配置文件，后者主要加载类结构下的Xml文件。ApplicationContext继承了HierarchicalBeanFactory、即永远父子及联、通过接口让子容器访问父容器的方法。而且通过其他接口扩充了BeanFactory的功能。

- ApplicationEventPublisher：让容器拥有发布上下文事件的功能，包括启动关闭等事件。实现了ApplicationListener事件监听端口的Bean可以接收到容器事件，并进行事件响应。在ApplicationContext的抽象实现类AbstractApplicationContext中存在一个ApplicationEventMulticaster，它负责所有的监听器、以便在容器产生上下文事件时通知事件监听者。
- MessageResource：为应用提i18n国际化消息访问的功能。
- ResourcePatternResolver：所有ApplicationContext实现类都实现了类似于PathMatchingResourcePatternResolver的功能，可以通过带前缀的Ant风格的资源文件装载Spring配置文件。
- Lifecycle：该接口提供了start（）和stop（）两个方法，主要用于异步处理过程。在具体实用时，该接口同时被ApplicationContext实现及具体Bean的实现，ApplicationContext会将start/stop的信息传递给容器中所有实现了该接口的Bean，以达到管理和控制JMX、任务调度的目的。

ConfigurableApplicationContext扩展于ApplicationContext，它新增了两个方法：

```java
public interface ConfigurableApplicationContext extends ApplicationContext, Lifecycle, Closeable {
    String CONFIG_LOCATION_DELIMITERS = ",; \t\n";
    String CONVERSION_SERVICE_BEAN_NAME = "conversionService";
    String LOAD_TIME_WEAVER_BEAN_NAME = "loadTimeWeaver";
    String ENVIRONMENT_BEAN_NAME = "environment";
    String SYSTEM_PROPERTIES_BEAN_NAME = "systemProperties";
    String SYSTEM_ENVIRONMENT_BEAN_NAME = "systemEnvironment";

    void setId(String var1);

    void setParent(ApplicationContext var1);

    ConfigurableEnvironment getEnvironment();

    void setEnvironment(ConfigurableEnvironment var1);

    void addBeanFactoryPostProcessor(BeanFactoryPostProcessor var1);

    void addApplicationListener(ApplicationListener<?> var1);

    //整个spring启动的核心方法
    void refresh() throws BeansException, IllegalStateException;

    void registerShutdownHook();
	//关闭
    void close();

    boolean isActive();

    ConfigurableListableBeanFactory getBeanFactory() throws IllegalStateException;
}
```



和BeanFactory初始化相似（本质上也是因为继承并且扩展了不同类型的BeanFactory接口），如果配置文件在类路径下，则可以优先考虑使用ClassPathXmlApplicationContext

```java
ApplicationContext act = new ClassPathXmlApplicationContext("com.xxxx.config.xml"）;
```

如果配置文件放在文件系统中，则使用如下：

```java
ApplicationContext act = new FileSystemXmlApplication("/Users/xxx/config.xml");
```

在获取到ApplicationContext实例后，同样可以通过getBean（）获取bean。ApplicationContext和BeanFactory初始化时有一个重大的不同是BeanFactory并未初始化Bean，而ApplicationContext在应用初始化阶段就会实例化所有**单实例**的Bean。

### 注解式配置方式

Spring支持注解式的配置方式，主要功能来自JavaConfig子项目，现已升级为Spring核心项目。当一个pojo类标注了@Configuration注解，既可以提供Spring所需的Bean信息：

```java
//表示这是一个配置类
@Configuration
public class Person {

    //定义一个bean
    @Bean(name = "Female")
    public Female born(){
        Female Jerry = new Female();
        Jerry.setSkin("Blue");
        Jerry.setAge(22);
        return Jerry;
    }
}
```

Spring为注解实现了专门的ApplicationContext实现类：ApplicationConfigApplicationContext。加载上述配置示例如下：

```java
public class AnnotationConfig {
    
    @Test
    public void getBean(){
        //通过一个带有Configuration注解的POJO获得Bean
        ApplicationContext act = new AnnotationConfigApplicationContext(Person.class);
        Femail Jerry = act.getBean("Female");
        assertNotNull(Jerry);
    }
}
```

### Groovy DSL配置Bean

Spring 4.x支持使用Groovy DSL来进行Bean定义配置。其与基于XML文件的配置类似，只不过基于Groovy脚本语言可以实现更复杂的Bean配置逻辑：

```groovy
beans{
    Jerry(Person){//名字（类型）
        name = "jerry"
        sex = "female"
        age = 22
    }
}
```

基于Groovy的配置方式很容易让开发者配置复杂的Bean初始化过程，比@Configuration和XML更加灵活，读取Groovy也有专门的ApplicationContext - > GenericGorrvyApplicationContet:

```java
public class GroovyConfig {
    
    @Test
    public void getBean(){
        ApplicationContext act = 
        new GenericGorrvyApplicationContet("com.xxx.beans.groovy");
        Femail Jerry = act.getBean("Female");
        assertNotNull(Jerry);
    }
}
```

# 总结



对于BeanFactory和ApplicationContext，ApplicationContext面向的是代码开发人员，而BeanFactory则是底层支持，由ApplicationContext实现和扩展，几乎所有的场景都可以使用ApplicationContext。



> 参考：《Spring技术内幕》、《精通Spring 4.x 企业应用开发实战》