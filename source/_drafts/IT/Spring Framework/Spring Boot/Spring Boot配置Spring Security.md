---
title: Spring Boot配置Spring Security
tags:
- Spring Boot
- Spring Security
categary:
- IT
- Spring Boot
---
# 默认配置
如果在classpath中存在Spring Security，那么web应用程序在默认情况下是开启安全防护的。
Spring Boot依赖Spring Security的内容协商策略来确定是使用HttpBasic还是FormLogin。
要在web应用程序中添加方法级安全性，您还可以在所需的设置中添加`@EnableGlobalMethodSecurity`。

默认的UserDetailsService(InMemoryUserDetailsManager)只有一个用户。用户名为user，密码是随机的，在应用程序启动时以WARN级别打印，示例如下:
```
Using generated security password: 78fa095d-3f4c-48b1-ad50-e24c31d5cf35
This generated password is for development use only. Your security configuration must be updated before running your application in production.
```

默认情况下，你在web应用程序中获得的基本特性是:
- 一个`UserDetailsService`bean，使用内存存储和一个拥有随即密码的用户。
- 整个应用程序的基于表单的登录或HTTP基本安全性(取决于请求中的Accept标头)。
- `DefaultAuthenticationEventPublisher`，用于发布身份验证事件。可以通过自定义`AuthenticationEventPublisher`改变默认配置。

# MVC Security

默认的安全配置在`SecurityAutoConfiguration`和`UserDetailsServiceAutoConfiguration中`实现。
- `SecurityAutoConfiguration`导入`SpringBootWebSecurityConfiguration`用于web安全，
- `UserDetailsServiceAutoConfiguration`配置认证，这在非web应用中也很重要。

要完全关闭默认的web应用程序安全配置，或者将多个Spring安全组件组合在一起，如OAuth2 Client和Resource Server，可以添加`SecurityFilterChain`类型的bean(这样做 **`不会禁用`** UserDetailsService配置)。

要关闭UserDetailsService配置，您可以添加`UserDetailsService`、`AuthenticationProvider`或`AuthenticationManager`类型的bean。

可以通过添加自定义`SecurityFilterChain` bean来覆盖访问规则。
