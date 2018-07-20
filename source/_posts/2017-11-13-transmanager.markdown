---
layout: post
title: "Spring | 事务管理"
date: 2017-11-13 17:08
author: "Gubaidan"
header-img: "/Header/2nojJdJoADk.png"
cdn: 'header-on'
tags:
	- Spring
---

# 什么是事务


事务(Transaction) 通俗的理解就是一件事情。 做事情要有始有终，不能半途而费。事务也是这样，要么做完，要么不做， 不要做一半留半，也就是说事务必须是一个不可分割的整体，就像我们在化学课里学到的原子，是构成物质的最小单位， 人们就归纳出事务的第一 个特性:**原子性(Atomicit)**。

特别是在数据库领域，事务是一个非常重要的概念，除了原子性以外，它还有一个极其重要的特性，那就是**一致性(Cosisteney)**。也就是说，执行完数据库操作后，数据不会被破坏，打个比方，如果从A账户转账到B账户，不可能因为A账户扣了钱，而B账户没有加钱。如果出现了这类事情，后果很严重。

当我们对数据库的数据进行操作后，数据需要永久的保存不能丢失，这就是**持久性(Duability)**。

当我们编写了一条update语句，提交到数据库的一刹那， 有可能别人也提交了条delete语句到数据库中。也许我们都是对同一条记录进行操作， 可以想象，如果不稍加控制，就会出大麻烦。我们必须保证数据库操作之间是“隔离”的(线程之间有时也要做到隔离)，彼此之向没有任何干扰，这就是**隔离性(Isolation)**。

要想真正做到操作之间完全没有任何干扰是很难的，于是，数据库权威专家制定一个规范，这个规范就是**事务隔离级别(Trasaction Iolation Level):**

- **READ UNCOMMITTED**


- **READ COMMITTED**


- **REPEATABLE READ**


-  **SERIALIZABLE**


从上往下，级别越来越高，并发性越来越差，安全性越来越高。

这四个特性称为**ACID**  ，其中：原子性是基础，隔离性是手段，持久性是目的，这三个真正服务于一致性。

# 事务面临的问题


ACID 四个特性中，隔离性是保证一致性的手段，所以四个隔离级别用来解决高并发情况下产生的问题：

- Dirty Read（脏读）
- Unrepeatable（不可重读读）
- Phantom（幻读）

**脏读** ：脏读就是指当A事务正在访问数据，并且对数据进行了修改，而这种修改还没有提交到数据库中，这时，B事务也访问这个数据，然后使用了这个数据。

例子：会计小王正在清算账单，清算到一半时结果被上报给老板，显然这个被上报的结果是不正确的。

**不可重复读** ：是指在一个事务内，多次读同一数据。A事务读了两次数据，在A第一次读完数据后，B事务修改了数据。这样就发生了在一个事务内两次读到的数据是不一样的，因此称为是不可重复读。（即不能读到相同的数据内容）

例子 ：会计小王正在清算账单，老板十分钟前和十分钟后看到的结果不一样。

**幻读** ：是指当事务不是独立执行时发生的一种现象。A事务对一个表中的数据行数进行了统计（这种修改涉及到表中的全部数据行）。 同时，B事务向表中插入一行新数据。如果A事务再次进行统计后就会发现数据多了一行，就好象发生了幻觉一样。

例子：会计小王正在清算账单后，老板又入了一笔账，如果会计小王再一次统计就会发现清算结果变了，像幻觉一样。



<p style="text-align:center;">√: 可能出现    ×: 不会出现</p>

| 隔离级别         | 脏读 | 不可重复读 | 幻读 |
| :--------------- | ---- | ---------- | ---- |
| Read uncommitted | √    | √          | √    |
| Read committed   | ×    | √          | √    |
| Repeatable read  | ×    | ×          | √    |
| Serializable     | ×    | ×          | ×    |

 

# Spring事务传播行为



Spring在TransactionDefinition接口中定义了七个事务传播行为。



- **PROPAGATION_REQUIRED** 如果存在一个事务，则支持当前事务。如果没有事务则开启一个新的事务。

- **PROPAGATION_SUPPORTS** 如果存在一个事务，支持当前事务。如果没有事务，则非事务的执行。但是对于事务同步的事务管理器，PROPAGATION_SUPPORTS与不使用事务有少许不同。

- **PROPAGATION_MANDATORY** 如果已经存在一个事务，支持当前事务。如果没有一个活动的事务，则抛出异常。

- **PROPAGATION_REQUIRES_NEW** 总是开启一个新的事务。如果一个事务已经存在，则将这个存在的事务挂起。

- **PROPAGATION_NOT_SUPPORTED** 总是非事务地执行，并挂起任何存在的事务。

- **PROPAGATION_NEVER** 总是非事务地执行，如果存在一个活动事务，则抛出异常

- **PROPAGATION_NESTED** 如果一个活动的事务存在，则运行在一个嵌套的事务中. 如果没有活动事务, 则按**PROPAGATION_REQUIRED** 属性执行

事务是逻辑处理原子性的保证手段，通过使用事务控制，可以极大的避免出现逻辑处理失败导致的脏数据等问题。

事务最重要的两个特性，是事务的传播级别和数据隔离级别。传播级别定义的是事务的控制范围，事务隔离级别定义的是事务在数据库读写方面的控制范围。

## 事务的七种传播级别详解

1. PROPAGATION_REQUIRED ，默认的spring事务传播级别，使用该级别的特点是，如果上下文中已经存在事务，那么就加入到事务中执行，如果当前上下文中不存在事务，则新建事务执行。所以这个级别通常能满足处理大多数的业务场景。

2. PROPAGATION_SUPPORTS ，从字面意思就知道，supports，支持，该传播级别的特点是，如果上下文存在事务，则支持事务加入事务，如果没有事务，则使用非事务的方式执行。所以说，并非所有的包在transactionTemplate.execute中的代码都会有事务支持。这个通常是用来处理那些并非原子性的非核心业务逻辑操作。应用场景较少。

3. PROPAGATION_MANDATORY ， 该级别的事务要求上下文中必须要存在事务，否则就会抛出异常！配置该方式的传播级别是有效的控制上下文调用代码遗漏添加事务控制的保证手段。比如一段代码不能单独被调用执行，但是一旦被调用，就必须有事务包含的情况，就可以使用这个传播级别。

4. PROPAGATION_REQUIRES_NEW ，从字面即可知道，new，每次都要一个新事务，该传播级别的特点是，每次都会新建一个事务，并且同时将上下文中的事务挂起，执行当前新建事务完成以后，上下文事务恢复再执行。

   这是一个很有用的传播级别，举一个应用场景：现在有一个发送100个红包的操作，在发送之前，要做一些系统的初始化、验证、数据记录操作，然后发送100封红包，然后再记录发送日志，发送日志要求100%的准确，如果日志不准确，那么整个父事务逻辑需要回滚。
   怎么处理整个业务需求呢？就是通过这个PROPAGATION_REQUIRES_NEW 级别的事务传播控制就可以完成。发送红包的子事务不会直接影响到父事务的提交和回滚。

5. PROPAGATION_NOT_SUPPORTED ，这个也可以从字面得知，not supported ，不支持，当前级别的特点就是上下文中存在事务，则挂起事务，执行当前逻辑，结束后恢复上下文的事务。

   这个级别有什么好处？可以帮助你将事务极可能的缩小。我们知道一个事务越大，它存在的风险也就越多。所以在处理事务的过程中，要保证尽可能的缩小范围。比如一段代码，是每次逻辑操作都必须调用的，比如循环1000次的某个非核心业务逻辑操作。这样的代码如果包在事务中，势必造成事务太大，导致出现一些难以考虑周全的异常情况。所以这个事务这个级别的传播级别就派上用场了。用当前级别的事务模板抱起来就可以了。

6. PROPAGATION_NEVER ，该事务更严格，上面一个事务传播级别只是不支持而已，有事务就挂起，而PROPAGATION_NEVER传播级别要求上下文中不能存在事务，一旦有事务，就抛出runtime异常，强制停止执行！这个级别上辈子跟事务有仇。

7. PROPAGATION_NESTED ，字面也可知道，nested，嵌套级别事务。该传播级别特征是，如果上下文中存在事务，则嵌套事务执行，如果不存在事务，则新建事务。

那么什么是嵌套事务呢？很多人都不理解，我看过一些博客，都是有些理解偏差。

嵌套是子事务套在父事务中执行，子事务是父事务的一部分，在进入子事务之前，父事务建立一个回滚点，叫save point，然后执行子事务，这个子事务的执行也算是父事务的一部分，然后子事务执行结束，父事务继续执行。重点就在于那个save point。看几个问题就明了了：

如果子事务回滚，会发生什么？

父事务会回滚到进入子事务前建立的save point，然后尝试其他的事务或者其他的业务逻辑，父事务之前的操作不会受到影响，更不会自动回滚。

如果父事务回滚，会发生什么？

父事务回滚，子事务也会跟着回滚！为什么呢，因为父事务结束之前，子事务是不会提交的，我们说子事务是父事务的一部分，正是这个道理。那么：

事务的提交，是什么情况？

是父事务先提交，然后子事务提交，还是子事务先提交，父事务再提交？答案是第二种情况，还是那句话，子事务是父事务的一部分，由父事务统一提交。

以上是事务的7个传播级别，在日常应用中，通常可以满足各种业务需求，但是除了传播级别，在读取数据库的过程中，如果两个事务并发执行，那么彼此之间的数据是如何影响的呢？

这就需要了解一下事务的另一个特性：数据隔离级别ACID

# Spring配置声明式事务

-  配置DataSource
- 配置事务管理器
- 事务的传播特性
- 那些类那些方法使用事务

Spring配置文件中关于事务配置总是由三个组成部分，分别是DataSource、TransactionManager和代理机制这三部分，无论哪种配置方式，一般变化的只是代理机制这部分。

DataSource、TransactionManager这两部分只是会根据数据访问方式有所变化，比如使用MyBatis进行数据访问 时，DataSource实际为SqlSessionFactoryBean，TransactionManager的实现为 DataSourceTransactionManager。

根据代理机制的不同，Spring事务的配置又有几种不同的方式：

第一种方式：每个Bean都有一个代理

 第二种方式：所有Bean共享一个代理基类

第三种方式：使用拦截器

第四种方式：使用tx标签配置的拦截器

第五种方式：全注解

## 使用tx标签配置的拦截器

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:aop="http://www.springframework.org/schema/aop"
	xmlns:tx="http://www.springframework.org/schema/tx"
	xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.0.xsd http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx-2.0.xsd http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-2.0.xsd">
	<!-- configure transaction -->
 
	<tx:advice id="defaultTxAdvice" transaction-manager="transactionManager">
		<tx:attributes>
			<tx:method name="get*" read-only="true" />
			<tx:method name="query*" read-only="true" />
			<tx:method name="find*" read-only="true" />
			<tx:method name="*" propagation="REQUIRED" rollback-for="java.lang.Exception" />
		</tx:attributes>
	</tx:advice>
 
	<tx:advice id="logTxAdvice" transaction-manager="transactionManager">
		<tx:attributes>
			<tx:method name="get*" read-only="true" />
			<tx:method name="query*" read-only="true" />
			<tx:method name="find*" read-only="true" />
			<tx:method name="*" propagation="REQUIRES_NEW"
				rollback-for="java.lang.Exception" />
		</tx:attributes>
	</tx:advice>
 
	<aop:config>
		<aop:pointcut id="defaultOperation"
			expression="@within(com.homent.util.DefaultTransaction)" />
		<aop:pointcut id="logServiceOperation"
			expression="execution(* com.homent.service.LogService.*(..))" />
			
		<aop:advisor advice-ref="defaultTxAdvice" pointcut-ref="defaultOperation" />
		<aop:advisor advice-ref="logTxAdvice" pointcut-ref="logServiceOperation" />
	</aop:config>
</beans>
```

如上面的Spring配置文件所示，日志服务的事务策略配置为propagation="REQUIRES_NEW"，告诉Spring不管上下文是否有事务，Log Service被调用时都要求一个完全新的只属于Log Service自己的事务。通过该事务策略，Log Service可以独立的记录日志信息，不再受到业务逻辑事务的干扰。

## @Transactional注解

@Transactional属性 

| 属性                   | 类型                               | 描述                                   |
| ---------------------- | ---------------------------------- | -------------------------------------- |
| value                  | String                             | 可选的限定描述符，指定使用的事务管理器 |
| propagation            | enum: Propagation                  | 可选的事务传播行为设置                 |
| isolation              | enum: Isolation                    | 可选的事务隔离级别设置                 |
| readOnly               | boolean                            | 读写或只读事务，默认读写               |
| timeout                | int (in seconds granularity)       | 事务超时时间设置                       |
| rollbackFor            | Class对象数组，必须继承自Throwable | 导致事务回滚的异常类数组               |
| rollbackForClassName   | 类名数组，必须继承自Throwable      | 导致事务回滚的异常类名字数组           |
| noRollbackFor          | Class对象数组，必须继承自Throwable | 不会导致事务回滚的异常类数组           |
| noRollbackForClassName | 类名数组，必须继承自Throwable      | 不会导致事务回滚的异常类名字数组       |

 **用法**:

​       @Transactional 可以作用于接口、接口方法、类以及类方法上。当作用于类上时，该类的所有 public 方法将都具有该类型的事务属性，同时，我们也可以在方法级别使用该标注来覆盖类级别的定义。

​         虽然 @Transactional 注解可以作用于接口、接口方法、类以及类方法上，但是 Spring 建议不要在接口或者接口方法上使用该注解，因为这只有在使用基于接口的代理时它才会生效。另外， @Transactional 注解应该只被应用到 public 方法上，这是由 Spring AOP 的本质决定的。如果你在 protected、private 或者默认可见性的方法上使用 @Transactional 注解，这将被忽略，也不会抛出任何异常。

​        默认情况下，只有来自外部的方法调用才会被AOP代理捕获，也就是，类内部方法调用本类内部的其他方法并不会引起事务行为，即使被调用方法使用@Transactional注解进行修饰。

```java
	@Autowired
	private MyBatisDao dao;
	
	@Transactional
	@Override
	public void insert(Test test) {
		dao.insert(test);
		throw new RuntimeException("test");//抛出unchecked异常，触发事物，回滚
	}
```

noRollbackFor:

```java
@Transactional(noRollbackFor=RuntimeException.class)
	@Override
	public void insert(Test test) {
		dao.insert(test);
		//抛出unchecked异常，触发事物，noRollbackFor=RuntimeException.class,不回滚
		throw new RuntimeException("test");
	}
```

Class，当作用于类上时，该类的所有 public 方法将都具有该类型的事务属性:

```java
@Transactional
public class MyBatisServiceImpl implements MyBatisService {
 
	@Autowired
	private MyBatisDao dao;
	
	
	@Override
	public void insert(Test test) {
		dao.insert(test);
		//抛出unchecked异常，触发事物，回滚
		throw new RuntimeException("test");
	}
```

propagation=Propagation.NOT_SUPPORTED:

```java
	@Transactional(propagation=Propagation.NOT_SUPPORTED)
	@Override
	public void insert(Test test) {
		//事物传播行为是PROPAGATION_NOT_SUPPORTED，以非事务方式运行，不会存入数据库
		dao.insert(test);
	}
```



# Spring事务典型问题

1、spring事务控制放在service层，在service方法中一个方法调用service中的另一个方法，默认开启几个事务？

spring的事务传播方式默认是PROPAGATION_REQUIRED，判断当前是否已开启一个新事务，有则加入当前事务，否则新开一个事务（如果没有就开启一个新事务），所以答案是开启了一个事务。

2、spring 什么情况下进行事务回滚？

Spring、EJB的声明式事务默认情况下都是在抛出unchecked exception后才会触发事务的回滚

unchecked异常,即运行时异常runntimeException 回滚事务;

checked异常,即Exception可try{}捕获的不会回滚.当然也可配置spring参数让其回滚。

spring的事务边界是在调用业务方法之前开始的，业务方法执行完毕之后来执行commit or rollback(Spring默认取决于是否抛出runtime异常).

如果抛出runtime exception 并在你的业务方法中没有catch到的话，事务会回滚。

一般不需要在业务方法中catch异常，如果非要catch，在做完你想做的工作后（比如关闭文件等）一定要抛出runtime exception，否则spring会将你的操作commit,这样就会产生脏数据.所以你的catch代码是画蛇添足。

 

> 参考：《Spring 技术内幕》、《架构探险》
>
> [深入理解事务--Spring事务的传播机制](https://blog.csdn.net/yuanlaishini2010/article/details/45792069)