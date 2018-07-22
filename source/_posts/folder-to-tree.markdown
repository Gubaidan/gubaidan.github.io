---
layout: post
title: "工具 | CSDN博客转 -> markdown"
date: 2018-05-25 08:52
author: "Gubaidan"
header-img: "/Header/l8Qo5NqYG_k.png"
cdn: 'header-on'
tags: 
	- 工具 

---

# csdn2md

> 转换html至markdown

### 前言 Before:
如何转换html至md，简直纠结。         

**因而，用来迁移博客的工具当然有呀。**[github here](https://github.com/Gubaidan/csdn2md.git).
<!-- more -->

### 使用 Usage:

```java
@param {author} csdn用户名               
@param {dirPath} 文件保存路径

public class Main {

    private static String host = "http://blog.csdn.net";

    public static void main(String args[]) throws IOException {

        String author = "xxxx";                           //csdn用户名

        String dirPath = "/Users/xxxx/";   //文件保存路径（绝对路径）

        new CorePaser().parse(host, author, dirPath, true);  //是否爬取图片 默认false
    }
}
```

### 展示 Show:

展示某个人博客的转换效果

<img style="width:80%;" src="http://p9n2j0ewi.bkt.clouddn.com/PostImg/csdn/D2338CFF-0768-4E86-AAAA-9E8305FE381F.png"/>

### 用到的工具

> 工具-[html2markdown](https://github.com/pnikosis/jHTML2Md)