---
title: Spring Security身份认证架构
date: 2023-06-15 17:04:51
tags:
- Spring Security
- Architecture
category:
- IT
- Spring Security
---

[参考](https://docs.spring.io/spring-security/reference/servlet/authentication/architecture.html)

# Servlet身份认证架构
本文描述了用于Servlet身份验证的Spring安全性的主要体系结构组件。
- SecurityContextHolder --- Spring Security在SecurityContextHolder中存储身份验证者的详细信息。
- SecurityContext --- 从SecurityContextHolder中获得，包含当前已认证用户的身份验证。
- Authentication --- 可以是AuthenticationManager的输入，来提供用户已经向验证提供的凭据；或SecurityContext中的当前用户。
- GrantedAuthority --- 在身份验证上授予主体的权限(即角色、范围等)。
- AuthenticationManager --- 定义Spring Security的过滤器如何执行身份验证的API。
- ProviderManager --- AuthenticationManager最常见的实现。
- AuthenticationProvider --- 由ProviderManager用于执行特定类型的身份验证。
- 使用AuthenticationEntryPoint请求凭据 --- 用于从客户端请求凭据(即重定向到登录页面，发送WWW-Authenticate响应等)
- AbstractAuthenticationProcessingFilter --- 用于身份验证的基本过滤器。这也很好地说明了身份验证的高层级流程以及各个部分如何协同工作。

# SecurityContextHolder
在Spring Security的认证模型中心位置的是SecurityContextHolder。
它包含着SecurityContext。

![SecurityContextHolder](/img/IT/SpringSecurity/Spring-Security身份认证架构-1.png)

SecurityContextHolder是Spring Security存储身份验证者的详细信息的地方。
Spring Security并不关心如何填充SecurityContextHolder。
如果包含值，则用作当前已验证的用户。

表明用户已通过身份验证的最简单方法是直接设置SecurityContextHolder:
``` Java
SecurityContext context = SecurityContextHolder.createEmptyContext(); 
Authentication authentication =
    new TestingAuthenticationToken("username", "password", "ROLE_USER"); 
context.setAuthentication(authentication);

SecurityContextHolder.setContext(context);
```

- 我们首先创建一个空的SecurityContext。您应该创建一个新的SecurityContext实例，来避免多线程间的竞争情况，而不是使用SecurityContextHolder.getContext(). setauthentication (authentication)。
- 接下来，我们创建一个新的Authentication对象。Spring Security并不关心在SecurityContext上设置了什么类型的身份验证实现。这里，我们使用TestingAuthenticationToken，因为它非常简单。更常见的生产场景是UsernamePasswordAuthenticationToken(用户详细信息、密码、权限)。
- 最后，我们在SecurityContextHolder上设置了SecurityContext。Spring Security使用此信息进行授权。

要获得关于经过身份验证的主体的信息，访问SecurityContextHolder即可：
``` Java
SecurityContext context = SecurityContextHolder.getContext();
Authentication authentication = context.getAuthentication();
String username = authentication.getName();
Object principal = authentication.getPrincipal();
Collection<? extends GrantedAuthority> authorities = authentication.getAuthorities();
```
默认情况下，SecurityContextHolder使用ThreadLocal来存储这些细节，这意味着即使SecurityContext没有显式地作为参数传递给这些方法，SecurityContext也始终对同一线程中的方法可用。
需要注意在当前主体的请求处理后清除线程，以保证方式使用ThreadLocal是安全的。
Spring Security的FilterChainProxy确保SecurityContext总是被清除。

由于处理线程的特定方式，有些应用程序并不完全适合使用ThreadLocal。例如，Swing客户端可能希望Java虚拟机中的所有线程使用相同的安全上下文。您可以在启动时使用策略配置SecurityContextHolder，以指定您希望如何存储上下文。
- 对于一个独立运行的应用程序，您将使用SecurityContextHolder.MODE_GLOBAL策略。
- 其他应用程序可能希望由安全线程生成的线程也具有相同的安全标识。你可以通过使用SecurityContextHolder.MODE_INHERITABLETHREADLOCAL来实现这一点。

有两种方式替换默认的模式SecurityContextHolder.MODE_THREADLOCAL：
- 首先是设置一个系统属性。
- 第二种方法是调用SecurityContextHolder上的静态方法。

大多数应用程序不需要更改默认值。但是，如果您需要，请查看SecurityContextHolder的JavaDoc以了解更多信息。

# SecurityContext
SecurityContext是从SecurityContextHolder中获得的。
SecurityContext包含一个Authentication对象。

# Authentication
Authentication接口在Spring Security中有两个主要用途:
- AuthenticationManager的输入，用于提供用户为进行身份验证所提供的凭据。在此场景中使用时，isAuthenticated()返回false。
- 表示当前已验证的用户。您可以从SecurityContext中获取当前的Authentication。

认证包括:
- 主体：用户标识。当使用用户名/密码进行身份验证时，这通常是UserDetails的实例。
- 凭证：通常是密码。在许多情况下，在用户通过身份验证后，将清除该选项，以确保它不会泄漏。
- 授权：GrantedAuthority实例是授予用户的高层级权限。比如：角色和作用域。

# GrantedAuthority
GrantedAuthority实例是授予用户的高层级权限。比如：Role和Scope。
您可以从Authentication.getAuthorities()方法获取GrantedAuthority实例。该方法提供了一个GrantedAuthority对象的集合。
毫无疑问，授予的授权是授予主体的授权。这种权限通常是“角色”，例如ROLE_ADMINISTRATOR或ROLE_HR_SUPERVISOR。这些角色稍后将配置为web授权、方法授权和域对象授权。Spring Security的其他部分解释这些权限，并期望它们出现。
当使用基于用户名/密码的身份验证时，GrantedAuthority实例通常由UserDetailsService加载。
通常，GrantedAuthority对象是应用程序范围的权限。它们并不特定于给定的域对象。
因此，您不太可能有一个GrantedAuthority来表示对编号54的Employee对象的权限，因为如果有数千个这样的权限，您将很快耗尽内存(或者，至少会导致应用程序花费很长时间来验证用户)。当然，Spring Security是专门为处理这种常见需求而设计的，但是您应该使用项目的域对象安全功能来实现这一目的。

# AuthenticationManager
AuthenticationManager是定义Spring Security的过滤器如何执行身份验证的API。然后，由调用AuthenticationManager的控制器(即Spring Security的Filters实例)在SecurityContextHolder上设置返回的身份验证。如果你没有集成Spring Security的Filters实例，你可以直接设置SecurityContextHolder，不需要使用AuthenticationManager。
虽然AuthenticationManager的实现可以是任何东西，但最常见的实现是ProviderManager。

# ProviderManager
ProviderManager是AuthenticationManager最常用的实现。
ProviderManager委托给AuthenticationProvider实例的列表。
每个AuthenticationProvider都有机会指示身份验证应该成功、失败，或者指示它不能做出决定并让下游的AuthenticationProvider来决定。
如果配置的AuthenticationProvider实例都不能进行身份验证，则身份验证失败，并出现ProviderNotFoundException。这是一个特殊的AuthenticationException，指示ProviderManager未配置为支持传递给它的身份验证类型。

![ProviderManager](/img/IT/SpringSecurity/Spring-Security身份认证架构-2.png)

实际上，每个AuthenticationProvider都知道如何执行特定类型的身份验证。例如，一个AuthenticationProvider可能能够验证用户名/密码，而另一个AuthenticationProvider可能能够验证SAML断言。这允许每个AuthenticationProvider执行一种非常特定的身份验证类型，同时支持多种类型的身份验证，并且只公开一个AuthenticationManager Bean。
ProviderManager还允许配置可选的父AuthenticationManager，在没有AuthenticationProvider可以执行身份验证的情况下，将参考该父AuthenticationManager。父对象可以是任何类型的AuthenticationManager，但它通常是ProviderManager的一个实例。

![ProviderManager实例](/img/IT/SpringSecurity/Spring-Security身份认证架构-3.png)

事实上，多个ProviderManager实例可能共享同一个父AuthenticationManager。这在有多个SecurityFilterChain实例的场景中比较常见，这些实例有一些共同的身份验证(共享的父AuthenticationManager)，但也有不同的身份验证机制(不同的ProviderManager实例)。

![不同的ProviderManager实例](/img/IT/SpringSecurity/Spring-Security身份认证架构-4.png)

默认情况下，ProviderManager尝试从成功的身份验证请求返回的Authentication对象中清除任何敏感凭据信息。这可以防止信息(比如密码)在HttpSession中保留的时间超过所需的时间。
例如，当您使用用户对象的缓存来提高无状态应用程序中的性能时，这可能会导致问题。如果Authentication包含对缓存中的对象的引用(例如UserDetails实例)，并且删除了它的凭据，则不再可能根据缓存的值进行身份验证。如果使用缓存，则需要考虑到这一点。一个明显的解决方案是首先在缓存实现中或在创建返回的Authentication对象的AuthenticationProvider中复制该对象。或者，您也可以在ProviderManager上禁用eraseCredentialsAfterAuthentication属性。有关Javadoc类，请参阅Javadoc。

# AuthenticationProvider
您可以向ProviderManager中注入多个AuthenticationProviders实例。
每个AuthenticationProvider执行特定类型的身份验证。例如，DaoAuthenticationProvider支持基于用户名/密码的身份验证，而JwtAuthenticationProvider支持验证JWT令牌。

# 使用AuthenticationEntryPoint请求凭证
AuthenticationEntryPoint用于发送用以从客户端请求凭据的HTTP响应。
有时，客户端主动包含凭据(例如用户名和密码)来请求资源。在这些情况下，Spring Security不需要提供从客户端请求凭据的HTTP响应，因为已经包含了凭据。
在其他情况下，客户端向未被授权访问的资源发出未经身份验证的请求。在这种情况下，使用AuthenticationEntryPoint的某个实现从客户端请求凭据。AuthenticationEntryPoint的实现可以执行重定向到登录页面，使用WWW-Authenticate报头响应，或采取其他操作。

# AbstractAuthenticationProcessingFilter
AbstractAuthenticationProcessingFilter用作对用户凭证进行身份验证的基本过滤器。在对凭证进行身份验证之前，Spring Security通常使用AuthenticationEntryPoint请求凭证。
接下来，AbstractAuthenticationProcessingFilter可以对提交给它的任何身份验证请求进行身份验证。

![AbstractAuthenticationProcessingFilter](/img/IT/SpringSecurity/Spring-Security身份认证架构-5.png)

(1)当用户提交他们的凭证时，AbstractAuthenticationProcessingFilter从HttpServletRequest创建一个Authentication对象来进行身份验证。创建的Authentication对象类型取决于AbstractAuthenticationProcessingFilter的子类。例如，UsernamePasswordAuthenticationFilter创建一个UsernamePasswordAuthenticationToken对象，并从HttpServletRequest中获取提交的用户名和密码。
(2)接下来，Authentication对象被传递到AuthenticationManager进行身份验证。
(3)如果认证失败，则失败。
- SecurityContextHolder被清除。
- RememberMeServices.loginFail被调用。如果没有配置remember me，则不做任何操作。
- 调用AuthenticationFailureHandler。
(4)如果认证成功，则表示成功。
- SessionAuthenticationStrategy被通知有一个新的登录。
- 在SecurityContextHolder上设置Authentication对象。之后，SecurityContextPersistenceFilter将SecurityContext保存到HttpSession。
- RememberMeServices.loginSuccess被调用。如果没有配置remember me，则不做任何操作。
- ApplicationEventPublisher发布一个InteractiveAuthenticationSuccessEvent事件对象。
- 调用AuthenticationSuccessHandler。

