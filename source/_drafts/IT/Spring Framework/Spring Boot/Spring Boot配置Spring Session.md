---
title: Spring Boot配置Spring Session
tags:
- Spring Boot
- Spring Session
categary:
- IT
- Spring Boot
---

# 默认配置
Spring Boot为提供Spring Session自动配置以支持广泛的数据存储。
在构建servlet web应用程序时，可以自动配置以下存储:
- Redis
- JDBC
- Hazelcast
- MongoDB

servlet自动配置取代了使用`@Enable*HttpSession`的需要。

如果在类路径中存在一个Spring Session模块，Spring Boot会自动使用该存储实现。如果您有多个实现，Spring Boot使用以下顺序来选择特定的实现:
- Redis
- JDBC
- Hazelcast
- MongoDB

如果没有Redis, JDBC, Hazelcast和MongoDB可用，`SessionRepository`将不被配置。

每个存储都有特定的附加设置。例如，可以为JDBC存储自定义表的名称，如下面的示例所示:
PropertiesYaml
``` YAML
spring:
  session:
    jdbc:
      table-name: "SESSIONS"
```
要设置会话的超时时间，可以使用`spring.session.timeout`属性。如果servlet web应用程序没有设置该属性，则自动配置将使用`server.servlet.session.timeout`作为后备值。

你可以使用@Enable*HttpSession (servlet)来控制Spring Session的配置。这将导致自动配置退出。然后可以使用注释的属性而不是前面描述的配置属性来配置Spring Session。