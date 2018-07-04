---
layout: post
title: "Spring-资源加载的方式"
date: 2017-11-02 13:20
author: "Gubaidan"
header-img: "/Header/RbM63-XsLUY.png"
cdn: 'header-on'
tags:
	- Spring
---

# Spring资源加载

![Resource接口依赖](http://p9n2j0ewi.bkt.clouddn.com/PostImg/2017-11-02-spring-resource/ResourceClassDependence.png)

为了访问访问不同的资源类型，必须使用不同的Resource实现类，这个是比较麻烦的，Spring提供了Resource这个强大的资源加载机制，不但能识别ftp、http、classpath等前缀的资源，还支持ant风格带通配符的资源地址。

### 资源地址表达式

以下是Spring支持的资源通配符前缀：

|  地址前缀  |             示例              |                        对应的资源类型                        |
| :--------: | :---------------------------: | :----------------------------------------------------------: |
| classpath: |      classpath:log4j.xml      | 从类路径中夹在资源，classpath：和classpath：/是等价，都是相对于类的跟路径，资源问见可以在标准的文件系统中，也可以在jar或者zip的类包中。 |
|   file:    |    file:/config/log4j.xml     | 使用urlResource从文件系统目录中装载资源，也可以采用绝对路金和地址 |
|  http://   | http://www.xxxx.com/log4j.xml |              使用urlResource从web服务器装载资源              |
|   ftp://   | ftp://www.xxxx/com/log4j.xml  |             使用urlResource从从ftp服务器装载资源             |
|  没有前缀  |   com.xxxxx.xxxx.log4j.xml    |    根据ApplicationContext的具体实现采用对应类型的resource    |

其中，和classpath： 对应的是classpath:* 前缀，假设有多个jar包或文件系统的路径都拥有一个相同的报名如（com.xxx），classpath： 只会在第一个包中查找，而classpath*则会在所有包前缀中查找资源文件。

这对于分模块打包的应用非常有用，假设一个名为smart的应用共分为3个模块，一个模块对应一个配置文件，分别是m1.xml , m2.xml , m3.xml ,都放到com.xxx目录下，每个模块单独打成jar包，使用classpath：只会加载第一个模块的配置文件，而使用classpath*则会加载所有配置文件。

Ant风格的资源地址支持三种匹配符。

- ？： 匹配文件名中的一个字符

- *：匹配文件名中任意字符。

- ** ：匹配多层路径。

  

下面是几个Ant风格的资源路径的示例：

- classpath:com.xx?x.log4j.xxml：匹配com.xxex.log4j.xml等

- file:/users/xxxxx/*.xml：匹配文件系统中以xml为后缀的所有资源文件。

- classpath:com.**.xml 匹配com路径下所有xml文件

- classpath:com/**/aaa/*.xml 匹配com不同路径下所有xml文件

  

### 资源加载器

Spring定义了一套资源加载的接口，并提供了实现：

![loader](http://p9n2j0ewi.bkt.clouddn.com/PostImg/2017-11-02-spring-resource/loader.png)

```java
import org.junit.jupiter.api.Test;
import org.springframework.core.io.Resource;
import org.springframework.core.io.support.PathMatchingResourcePatternResolver;
import org.springframework.core.io.support.ResourcePatternResolver;

import java.io.IOException;

import static org.hibernate.validator.internal.util.Contracts.assertNotNull;

public class ResourceLoaderText {
    @Test
    private void getResource() throws IOException {
        ResourcePatternResolver resourcePatternResolver = new PathMatchingResourcePatternResolver();

        //加载所有com.smart 下。xml结尾的文件
        Resource resource[] = resourcePatternResolver.getResources("classpath*: com.smart.**/*.properties");

        assertNotNull(resource);
        for(Resource item: resource){
            System.out.println(item.getDescription());
        }
    }

}
```



### 注意

用Resource操作文件时，如果文件在项目发布时会打包到jar包，那么不能使用Resource.getFile()方法，否则会抛出FileNotFound异常，但可以使用Resource.getInputStream()读取。