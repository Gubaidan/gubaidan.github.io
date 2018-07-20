---
layout: post
title: "Navicat | show variables like'char%' 字符集无法更改导致乱码"
date: 2018-03-03 14:02
author: "Gubaidan"
header-img: "/Header/hero@2x.jpg"
cdn: 'header-on'
tags:
        - mysql
---

## BUG FIX

今天用python把一个txt文件中的数据导入到Mysql，因为txt中包含中文信息，导致中文乱码。

- Mysql版本：5.7.22
- 远程系统 ：CentOS Linux release 7.4.1708 (Core) 
- 本机系统 : MacOS 10.13.5
- Navicat : 11.8
- python 3.6

导入中文乱码之后，查了一下远程Mysql的数据库编码：

```sql
mysql> SHOW VARIABLES LIKE 'character%';
+--------------------------+----------------------------------+
| Variable_name            | Value                            |
+--------------------------+----------------------------------+
| character_set_client     | latin1                           |
| character_set_connection | latin1                           |
| character_set_database   | utf8                             |
| character_set_filesystem | binary                           |
| character_set_results    | latin1                           |
| character_set_server     | utf8                             |
| character_set_system     | utf8                             |
| character_sets_dir       | /usr/local/mysql/share/charsets/ |
+--------------------------+----------------------------------+
8 rows in set (0.00 sec)
```

发现有三个编码是Latin1，于是更改Mysql配置文件my.inf

```ini
[mysqld] 
character_set_server=utf8   #这里不能写成 default-character-set=utf8
init-connect='set names utf8'
collation-server = utf8_general_ci

[mysql]
default-character-set = utf8

[client]
default-character-set=utf8
```

因Mysql5.7 之后 [mysqld] 中配置 default-character-set=utf8 将会报错。

修改完成之后重启mysql：

```sql
mysql> SHOW VARIABLES LIKE 'character%';
+--------------------------+----------------------------------+
| Variable_name            | Value                            |
+--------------------------+----------------------------------+
| character_set_client     | utf8                             |
| character_set_connection | utf8                             |
| character_set_database   | utf8                             |
| character_set_filesystem | binary                           |
| character_set_results    | utf8                             |
| character_set_server     | utf8                             |
| character_set_system     | utf8                             |
| character_sets_dir       | /usr/local/mysql/share/charsets/ |
+--------------------------+----------------------------------+
8 rows in set (0.00 sec)
```

但是更改之后，远程mysql数据库所有编码字符集正常，但是在navicat仍然显示：

```sql
mysql> SHOW VARIABLES LIKE 'character%';
+--------------------------+----------------------------------+
| Variable_name            | Value                            |
+--------------------------+----------------------------------+
| character_set_client     | latin1                           |
| character_set_connection | latin1                           |
| character_set_database   | utf8                             |
| character_set_filesystem | binary                           |
| character_set_results    | latin1                           |
| character_set_server     | utf8                             |
| character_set_system     | utf8                             |
| character_sets_dir       | /usr/local/mysql/share/charsets/ |
+--------------------------+----------------------------------+
8 rows in set (0.00 sec)
```

当然，我是用navicat图形界面查的，但是结果和这个是一样的。

测试了一下，导入数据后中文依旧乱码，而且在navicat 右键打开的数据库连接属性中 字符集也是utf8，既然字符集都是一样的，为什么还会出现乱码，百思不解。

因为怀疑是python处理文件时出错，特地排查了一下python

python文件读取：

```python
    with open('/Users/xxxx.txt', 'r', encoding="utf8") as txt:
        for line in txt:
```

存入数据库：

```python
try:
	engine_route = create_engine("mysql+pymysql://用户名:密码@host:3306/数据库?charset=utf8",
                                     max_overflow=5)
	pd.io.sql.to_sql(result, name='hostel', con=engine_route, if_exists='append',
                         index=False, index_label=False, schema='DoubleG', chunksize=10000)
finally:
	del result
```

因为都设置的是utf8，之间测试过，把上述编码改为 latin1 ，依然乱码。

本着试试又不会死的原则，我把navicat数据库连接中的 编码集由**utf-8 改为 自动**，再次查询Navicat：

```sql
mysql> SHOW VARIABLES LIKE 'character%';
+--------------------------+----------------------------------+
| Variable_name            | Value                            |
+--------------------------+----------------------------------+
| character_set_client     | utf8                             |
| character_set_connection | utf8                             |
| character_set_database   | utf8                             |
| character_set_filesystem | binary                           |
| character_set_results    | utf8                             |
| character_set_server     | utf8                             |
| character_set_system     | utf8                             |
| character_sets_dir       | /usr/local/mysql/share/charsets/ |
+--------------------------+----------------------------------+
8 rows in set (0.00 sec)
```

结果正常，数据测试正常。

## 总结

Mac 版Navicat 貌似和windows不同，windows中数据库连接中将字符编码改为utf8就能解决问题（当然前提是配置好mysql数据库编码），而mac版编码必须自动。



