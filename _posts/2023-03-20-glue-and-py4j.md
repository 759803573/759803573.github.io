---
layout: post
title: '3min阅读: 在AWS Glue中使用Py4j连接DB执行查询'
date: 2023-03-20
author: Calvin Wang
# cover: '/assets/img/202209/saml.drawio.png'
tags: aws glue py4j
---

> 本文目的就是说说Py4j， 连接DB查询就是个很常见的需求， 当个引子


## 遇到了什么问题？
受限于平台和AWS Glue本身的申请， 在仅使用Python时会遇到各种各的问题，尤其是依赖维护。
举个例子：
像pymssql这类需要编译的包，就够项目吃上一壶的了。

## 解决方式呢？
使用Py4j调用Java包来执行需要的操作。
```python
from py4j.java_gateway import java_import
jvm = spark._jvm
java_import(jvm , "org.postgresql.Driver")
java_import(jvm , "java.sql.DriverManager")

conn = jvm.DriverManager.getConnection(f'jdbc:postgresql://{host}:5432/{db}', usr, password)
sql = "select attnum, attname  from pg_catalog.pg_attribute pa limit 10"
stmt = conn.createStatement()
rs = stmt.executeQuery(sql)
while rs.next():
    print(f'attnum: {rs.getInt(1)} \t attname: {rs.getString(2)}')

# Output:
# attnum: 1        attname: oid
# attnum: 2        attname: proname
# attnum: 3        attname: pronamespace
# attnum: 4        attname: proowner
# attnum: 5        attname: prolang
# attnum: 6        attname: procost
# attnum: 7        attname: prorows
# attnum: 8        attname: provariadic
# attnum: 9        attname: prosupport
# attnum: 10       attname: prokind

```

## 这样做的好处呢？
1. Jar包迁移性好
2. Jar在一般性工具开发覆盖更全。
3. JDBC Jar 在功能上要比一些Python原生库强
