---
layout: post
title: "Spring | AOP理解及实现"
date: 2017-11-09 13:20
author: "Gubaidan"
header-img: "/Header/RbM63-XsLUY.png"
cdn: 'header-on'
tags:
	- Spring
---

# AOP是什么

在简单的了解过AOP之后，被“面向方面编程”、“面向切面编程”、“切点”、“切面”这些术语搞的糊里糊涂，简单来说AOP是Aspect Origented Programing的简称，根据字面意思确实是面向切面编程，那么这个词到底又是什么意思？

按照软件重构的思想，如果一个类在多个方法或者类中出现相同的代码，我们就认为可以将这些代码提取到父类，但是有些代码例如在代码执行前得到执行开始时间，执行后获得结束时间，这样用来统计执行时间的代码可能隐藏在一堆杂乱的代码之中，这样的话通过父类继承的方式是无法实现的，这也就是AOP将要解决的问题。

> 我的通俗理解就是：**在某个方法执行前后动态的为其插入想要执行的方法或者代码。**

# AOP术语

**连接点（Joinpoint）:**特定点是程序执行的某个特定位置，如类开始初始化前，类初始化后，类的某个方法调用前/后，方法抛出异常后。一个类或一段代码拥有一些具有边界性质的特定点，这些代码中的特定点就被称为“连接点”。 

Spring仅支持方法的连接点，即仅能在方法调用前，方法调用后，方法抛出异常时及方法调用前后这些程序执行点织入增强。 

连接点由两个信息确定：一是用方法表示的程序执行点，二是用相对位置表示的方位。如在Test.foo()方法执行前的连接点，执行点为Test.foo(),方法为该方法执行前的位置。Spring使用切点对执行点进行定位，而方位则在增强类型中增强

**切点(Pointcut）:**每个程序类都有多个连接点，如一个拥有两个方法的类，这两个方法就是连接点，即连接点是程序类中客观存在的事物。AOP通过“切点”定位特定的连接点。连接点相当于数据库中的记录，而切点相当于查询条件，一个切点可以匹配多个连接点。

在Spring中，切点通过org.springframework.aop.Pointcut接口进行描述，它使用类和方法作为连接点的查询条件。Spring AOP的规则解析引擎负责解析切点所设定的查询条件，找到对应的连接点。确切地说，应该是执行点而非连接点，因为连接点是方法执行前/后等包括方位信息的具体程序执行点，而切点只定位到某个方法上，所以如果希望定位到具体的连接点上，需要提供方位信息

**增强（Advice）:**增强是织入目标类连接点上的一段程序代码。在Spring中，增强除用于描述一段程序代码外，还拥有另一个和连接点相关的信息，就是执行点的方位。结合执行点的方位信息和切点信息，就可以找到特定的连接。 

正因为增强即包含由于添加到目标连接点上的一段执行逻辑，又包含用于定位连接点的方位信息，所以Spring所提供的增强接口都是带方位名的，如BeforeAdivce，AfterReturningAdvice和ThrowsAdvice等。BeforeAdvice表示方法调用前的位置，而AfterReturningAdvice表示访问后的位置。 所以只有结合切点和增强，才能确定特定的连接点并实施逻辑增强。

**目标对象(Target)  :**增强逻辑的织入目标类。如果没有AOP，那么目标业务类需要自己实现所有的逻辑.

在AOP的帮助下，ForumService只实现那些非横切逻辑的程序逻辑，而性能监视和事务管理等这些横切逻辑就可以使用AOP动态织入特定的连接点上

**引介（Introduction）:** 引介是一种特殊的增强，为类添加一些属性和方法。这样，即使一个业务类原本没有实现某个接口，通过AOP的引介功能，也可以动态地为该业务类添加接口的实现逻辑，使业务类成为这个接口的实现类

**织入(Weaving)  :** 织入是将增强添加到目标类的具体连接点上的过程.AOP就像一台织布机，将目标类，增强或引介编织到一切。 

**AOP有三种织入方式：** 

- 编译期织入，这要求使用特殊的Java编译器 
- 类装载期织入，这要求使用特殊的类装载器 
- 动态代理织入，在运行期为目标类添加增强生成子类的方式，Spring采用动态代理织入

**代理（Proxy)  :** 一个类被AOP织入增强后，就产生一个结果类，它是融合了原类和增强逻辑的代理类。根据不同的代理方式，代理类既可能是和原类具有相同接口的类，也可能就是原类的子类，所以可以采用与调用原类相同的方式调用代理类。

**切面（Aspect)  :**切面由切点和增强（引介）组成，既包括横切逻辑的定义，也包括连接点的定义。Spring AOP就是负责实施切面的框架，它将切面所定义的横切逻辑织入切面所指定的连接点中

# 基础知识

Spring AOP使用了两种代理机制：一种是基于JDK的动态代理；另一种是基于CGLib的动态代理。 
之所以需要两种代理机制，是因为JDK本身只提供接口的代理，而不支持类的代理。

CGLib所创建的动态代理对象的性能比JDK所创建的动态代理对象的性能高很多，但CGLib在创建代理对象时所花费的时间比JDK动态代理多。  对于singleton的代理对象或者具有实例池的代理，因为无需频繁地创建代理对象，所以比较适合采用CGLib动态代理技术；反之适合采用JDK动态代理技术。

# 实现者

按照增强在目标类方法中的连接点位置，可以分为5类：

- 前置增强：org.springframework.aop.BeforeAdvice代表前置增强，因为Spring只支持方法级的增强，所以MethodBeforeAdvice是目前可用的前置增强，表示在目标方法执行前实施增强，而BeforeAdvice是为了将来版本扩展需要而定义的。

- 后置增强：org.springframework.aop.AfterReturningAdvice代表后置增强，表示在目标方法执行后实施增强。

- 环绕增强：org.aopalliance.intercept.MethodInterceptor代表环绕增强，表示在目标方法执行前后实施增强。

- 异常抛出增强：org.springframework.aop.ThrowsAdvice代表抛出异常增强，表示在目标方法抛出异常后实施增强。

- 引介增强：org.springframework.aop.IntroductionInterceptor代表引介增强，表示在目标类中添加一些新的方法和属性

## BeforeAdvice

person接口：

```java
public interface Person {
    void speak(String name);
    void run(String time);
}
```

 具体实现类：

```java
public class Man implements Person{
    public void speak(String name) {
        System.out.println("speak to "+name);
    }

    public void run(String time) {
        System.out.println("runing "+time);
    }
}
```

增强类：执行Speak或者run方法之前执行

```java
public class PersonBeforeAdvice implements MethodBeforeAdvice {
    public void before(Method method, Object[] objects, Object obj) throws Throwable {
        String name=(String)objects[0];
        System.out.println("Hi! Mr "+name);
    }
}
```

PersonBeforeAdvice接口仅定义了唯一的方法：before(Method method,Object[] args,Object obj)throws Throwable,其中method为目标类的方法；objects为目标类方法的入参，而obj为目标类实例。当方法发生异常时，将阻止目标类方法的执行。

xml文件配置：

```xml
<bean id="manBefore" class="com.aop.advice.ManBeforeAdvice" />
<bean id="target" class="com.aop.advice.Man" />
<bean id="manman" class="org.springframework.aop.framework.ProxyFactoryBean"
	  p:interceptorNames="manBefore"
      p:proxyInterfaces="com.aop.advice.Person" 
	  p:target-ref="target"
	  p:proxyTargetClass="true"/>
```

测试类：

```java
public class BeforeAdviceTest {
    @Test
    public void before(){
        Person target=new Man();
        BeforeAdvice advice=new ManBeforeAdvice();

        //①Spring提供的代理工厂
        ProxyFactory pf=new ProxyFactory();
        //②设置代理目标
        pf.setTarget(target);
        //③为代理目标添加增强
        pf.addAdvice(advice);
        //④生成代理实例
        Person proxy=(Person) pf.getProxy();
        proxy.speak("gubaidan");
        proxy.run("2017-11-09");
    }
}
```

输出：

```shell
Hi Mr gubaidan        //增强效果
speak to gubaidan
Hi mr gubaidan        //增强效果
running 2017-11-09
```

## 解析ProxyFactory

在BeforeAdviceTest中，使用org.springframework.aop.framework.ProxyFactory代理工厂将GreetingBeforeAdvice的增强织入目标类NaiveWaiter中。ProxyFactory内部就是使用JDK或CGLib动态代理技术将增强应用到目标类中的。

Cglib2AopProxy使用CGLib动态代理技术创建代理，而JdkDynamicAopProxy使用JDK动态代理技术创建代理。 如果通过ProxyFactory的setInterfaces(Class[] interfaces)方法指定目标接口进行代理，则ProxyFactory使JdkDynamicAopProxy。

> 如果是针对类的代理，则使用Cglib2AopProxy。 

还可以通过ProxyFactory的setOptimize(true)方法让ProxyFactory启动优化代理方式，这样，针对接口的代理也会使用Cglib2AopProxy。

BeforeAdviceTest使用的是CGLib动态代理技术，当我们指定接口进行代理时，将使用JDK动态代理技术

```java
//①Spring提供的代理工厂
ProxyFactory pf=new ProxyFactory();
//②设置代理目标
pf.setInterfaces(target.getClass().getInterfaces());//指定接口进行代理
pf.setTarget(target);
```

如果指定启动代理优化，则ProxyFactory还将使用Cglib2AopProxy代理

```java
//①Spring提供的代理工厂
ProxyFactory pf=new ProxyFactory();
//②设置代理目标
pf.setInterfaces(target.getClass().getInterfaces());//指定接口进行代理
pf.setTarget(target);
pf.setOpaque(true);   //启动优化
```

ProxyFactory通过addAdvice(Advice)方法添加一个增强，可以使用该方法添加多个增强。多个增强形成一个增强链，它们的调用顺序和添加顺序一致，可以通过addAdvice(int ,Advice)方法将增强添加到增强链的具体位置(初始位置为0)

ProxyFactoryBean是FactoryBean接口的实现类，FactoryBean负责实例化一个Bean。ProxyFactoryBean负责为其他Bean创建代理实例，它在内部使用ProxyFactory来完成这项工作。

ProxyFactoryBean的几个常用的可配置属性 :

- target：代理的目标对象 
- proxyInterfaces：代理所要实现的接口，可以是多个接口 
- interceptorNames：需要织入目标对象的Bean列表，采用Bean的名称指定。这些Bean必须是实现了org.aopalliance.intercept.MethodInterceptor或org.springframework.aop.Advisor的Bean，配置中的顺序对应调用的顺序。 
- singleton：返回的代理是否单实例，默认为单实例。 
- optimize：当设置为true时，强制使用CGLib动态代理。对于singleton的代理，推荐使用CGLib；对于其他作用域类型的代理，最好使用JDK动态代理 
- proxyTargetClass：是否对类进行代理（而不是对接口进行代理）。当设置为true时，使用CGLib动态代理

 

## 引介增强

引介增强为目标类创建新的方法和属性，所以引介增强的连接点是类级别的，而非方法级别的。通过引介增强，可以为目标类添加一个接口的实现，可以为目标类创建实现某接口的代理。

Spring定义了引介增强接口IntroductionInterceptor，该接口没有定义任何方法，Spring为其提供了DelagatingIntroductionInterceptor实现类。一般情况下，通过扩展该实现类定义自己的引介增强类。

前面性能监视的例子，我们对所有的业务类都织入了性能监视的增强。由于性能监视会影响业务系统的性能，所以是否启动性能监视应该是可控的。现在通过引介增强来实现。

首先定义一个用于标识目标类是否支持性能监视的接口：

```java
public interface Monitorable {
    void setMonitorActive(boolean active);
}
```

通过setMonitorActive(boolean active)接口方法控制业务类性能监视功能的激活和关闭状态

下面通过扩展DelegatingIntroductionInterceptor为目标类引入性能监视的可控功能

```java
public class ControllablePerformanceMonitor extends DelegatingIntroductionInterceptor implements Monitorable{
   private ThreadLocal<Boolean> MonitorStatusMap=new ThreadLocal<Boolean>(); //①

   public void setMonitorActive(boolean active){   //②
       MonitorStatusMap.set(active);
   }

   //③拦截方法
    public Object invoke(MethodInvocation mi) throws Throwable {
        Object obj=null;
        //④对于支持性能监视可控代理，通过判断其状态决定是否开启性能监视功能
        if(MonitorStatusMap.get()!=null&&MonitorStatusMap.get()){
            PerformanceMonitor.begin(mi.getClass().getName()+"."+mi.getMethod().getName());
            obj=super.invoke(mi);
            PerformanceMonitor.end();
        }else{
            obj=super.invoke(mi);
        }
        return obj;
    }
}
```



扩展DelegatingIntroductionInterceptor的同时还实现了Monitorable接口，提供接口方法的实现。 
在①定义了一个ThreadLocal类型的变量，用于保存性能监视开关状态，为了解决单实例singleton线程安全的问题，通过ThreadLocal让每个线程单独使用一个状态。

在③中覆盖了父类中的invoke()方法，该方法用于拦截目标类方法的调用，根据开关状态有条件地对目标实例方法进行性能监视。 

在④处，MonitorStatusMap.get()方法返回的Boolean被自动拆包为boolean类型的值

通过Spring配置，将这个引介增强织入业务类ForumService中 

```xml
<bean id="pmonitor" class="com.smart.introduce.ControllablePerformaceMonitor" />
<bean id="forumServiceTarget" class="com.smart.introduce.ForumService" />
<bean id="forumService" class="org.springframework.aop.framework.ProxyFactoryBean"
	p:interfaces="com.smart.introduce.Monitorable"
	p:target-ref="forumServiceTarget"
	p:interceptorNames="pmonitor" 
	p:proxyTargetClass="true" />
```

首先，需要指定引介增强所实现的接口，如①。其次，由于只能通过为目标创建子类的方法生成引介增强的代理，所以必须将proxyTargetClass设置为true

如果没对ControllablePerformanceMonitor进行线程安全的特殊处理，就必须将singleton属性设置为true，让ProxyFactoryBean产生prototype作用域类型的代码。当CGLib动态创建代理的性能很低，而每次通过getBean()方法中从容器中获取作用域类型为prototype的Bean时都将返回一个新的代理实例，这种性能影响是巨大的，这就是为什么通过ThreadLocal对ControllablePerformanceMonitor的开关状态进行线程安全化处理的原因。 

通过线程安全化处理后，就可以使用默认的singleton Bean作用域，这样创建代理的动作仅发生一次。

```java
public class IntroduceTest {
    @Test
    public void introduce(){
        String configPath="com/smart/introduce/beans.xml";
        ApplicationContext ctx=new ClassPathXmlApplicationContext(configPath);
        ForumService forumService=(ForumService)ctx.getBean("forumService");

        forumService.removeForum(10);    //默认情况下未开启性能监视功能
        forumService.removeTopic(1024);
        Monitorable monitorable=(Monitorable)forumService;  //开启性能监视，实现了Monitorable接口
        monitorable.setMonitorActive(true);
        forumService.removeForum(10);
        forumService.removeTopic(1024);
    }
}
```

 ②处，强制性地将forumService转换为Monitorable类型，代码的成功执行表示从Spring容器中返回的代理确实引入了Monitorable接口方法的实现。



# 以XML方式配置切面

## 简介

除了使用AspectJ注解声明切面，Spring也支持在bean配置文件中声明切面。这种声明是通过aop名称空间中的XML元素完成的。

正常情况下，基于注解的声明要优先于基于XML的声明。通过AspectJ注解，切面可以与AspectJ兼容，而基于XML的配置则是Spring专有的。由于AspectJ得到

越来越多的 AOP框架支持，所以以注解风格编写的切面将会有更多重用的机会。

## 配置细节

在bean配置文件中，所有的Spring AOP配置都必须定义在<aop:config>元素内部。对于每个切面而言，都要创建一个<aop:aspect>元素来为具体的切面实现

引用后端bean实例。切面bean必须有一个标识符，供<aop:aspect>元素引用。

```xml
&lt;bean id="logAspect" class="com.atguigu.aspect.LogAspect"&gt;&lt;/bean&gt;
&lt;bean id="validatorAspect" class="com.atguigu.aspect.ValidatorAspect"&gt;&lt;/bean&gt;
&lt;bean id="mathCalulator" class="com.atguigu.calulator.MathCalulator"&gt;&lt;/bean&gt;
&lt;aop:config&gt;
	&lt;aop:aspect ref="logAspect"&gt;&lt;/aop:aspect&gt;
	&lt;aop:aspect ref="validatorAspect"&gt;&lt;/aop:aspect&gt;
&lt;/aop:config&gt;

```

## 声明切入点

切入点使用<aop:pointcut>元素声明。 

切入点必须定义在<aop:aspect>元素下，或者直接定义在<aop:config>元素下。 

定义在<aop:aspect>元素下：只对当前切面有效 

定义在<aop:config>元素下：对所有切面都有效 

基于XML的AOP配置不允许在切入点表达式中用名称引用其他切入点。

```xml
&lt;aop:pointcut expression="execution(* *.*(..))" id="mypoint"/&gt;
```



## 声明通知

在aop名称空间中，每种通知类型都对应一个特定的XML元素。 通知元素需要使用<pointcut-ref>来引用切入点，或用<pointcut>直接嵌入切入点表达式。 method属性指定切面类中通知方法的名称

```xml
&lt;aop:before method="logStart" pointcut="execution(* *.*(..))"/
```


# 最后

编程的意义是解决和简化手工劳动，而AOP的意义则是简化代码开发。

>参考：《Spring技术内幕》、《精通Spring 4.x 企业应用开发实战》