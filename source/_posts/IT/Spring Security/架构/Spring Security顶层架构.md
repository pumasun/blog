---
title: Spring Security顶层架构
tags:
- Spring Security
- Architecture
category:
- IT
- Spring Security
---

本文讨论基于Servlet的应用程序中的Spring Security高层级架构。
# 过滤器(Filter)回顾 
Spring Security的Servlet支持基于Servlet过滤器，因此首先了解过滤器的作用是有帮助的。下图显示了单个HTTP请求的处理程序的典型分层结构。
![Filter Chain](/img/IT/SpringSecurity/Spring-Security顶层架构-1.png)

客户端向应用程序发送请求，容器创建一个FilterChain，其中包含Filter实例和Servlet，它们应该根据请求URI的路径来处理HttpServletRequest。在Spring MVC应用程序中，Servlet是DispatcherServlet的一个实例。最多，一个Servlet可以处理一个HttpServletRequest和HttpServletResponse。但是，可以使用多个Filter:
- 防止下游Filter实例或Servlet被调用。在这种情况下，Filter通常会编写HttpServletResponse。
- 修改下游过滤器实例和Servlet使用的HttpServletRequest或HttpServletResponse。

Filter的功能来自传入它的FilterChain：
``` java
public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) {
	// do something before the rest of the application
    chain.doFilter(request, response); // invoke the rest of the application
    // do something after the rest of the application
}
```
由于Filter只影响下游的Filter实例和Servlet，因此调用每个Filter的顺序非常重要。

# DelegatingFilterProxy
Spring提供了一个名为DelegatingFilterProxy的Filter实现，来桥接Servlet容器的生命周期和Spring的ApplicationContext。Servlet容器允许使用它自己的标准来注册Filter实例，但是它不知道spring定义的bean。您可以通过标准的Servlet容器机制注册DelegatingFilterProxy，但将所有工作委托给实现Filter的Spring Bean。
下面是如何将DelegatingFilterProxy适配到Filter实例和FilterChain的示意图。
![DelegatingFilterProxy](/img/IT/SpringSecurity/Spring-Security顶层架构-2.png)

DelegatingFilterProxy从ApplicationContext中查找Bean Filter0，然后调用Bean Filter0。下面的清单显示了DelegatingFilterProxy的伪代码:
``` java
public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) {
	// Lazily get Filter that was registered as a Spring Bean
	// For the example in DelegatingFilterProxy delegate is an instance of Bean Filter0
	Filter delegate = getFilterBean(someBeanName);
	// delegate work to the Spring Bean
	delegate.doFilter(request, response);
}
```
DelegatingFilterProxy的另一个好处是它允许延迟查找Filter bean实例。这很重要，因为容器需要在容器启动之前注册Filter实例。然而，Spring通常使用ContextLoaderListener来加载Spring Bean，这在需要注册Filter实例之后才会完成。

# FilterChainProxy
Spring Security的Servlet支持包含在FilterChainProxy中。FilterChainProxy是Spring Security提供的一个特殊的过滤器，它允许通过SecurityFilterChain委托给许多过滤器实例。因为FilterChainProxy是一个Bean，所以它通常被包装在一个DelegatingFilterProxy中。
下图显示了FilterChainProxy的角色:
![FilterChainProxy](/img/IT/SpringSecurity/Spring-Security顶层架构-3.png)

# SecurityFilterChain
FilterChainProxy使用SecurityFilterChain来决定调用哪个Spring Security Filter实例来处理当前请求。
下图显示了SecurityFilterChain的角色。
![SecurityFilterChain](/img/IT/SpringSecurity/Spring-Security顶层架构-4.png)
SecurityFilterChain中的Security Filter是典型的Bean，但他们是被FilterChainProxy而不是DelegatingFilterProxy注册的。FilterChainProxy为直接向Servlet容器或DelegatingFilterProxy注册拥有许多优点：
- 首先，它为所有Spring Security的Servlet支持提供了一个起点。因此，如果您试图排除Spring Security的Servlet支持的故障，在FilterChainProxy中添加一个调试点是一个很好的开始。
- 其次，由于FilterChainProxy是Spring Security使用的核心，它可以执行被视为非可选的任务。例如，它清除SecurityContext以避免内存泄漏。它还应用了Spring Security的HttpFirewall来保护应用程序免受某些类型的攻击。
- 此外，它在确定何时应该调用SecurityFilterChain方面提供了更大的灵活性。在Servlet容器中，Filter实例仅基于URL调用。然而，FilterChainProxy可以通过使用RequestMatcher接口基于HttpServletRequest中的任何东西来确定调用。
下图显示了多个SecurityFilterChain实例：
![Multiple SecurityFilterChain](/img/IT/SpringSecurity/Spring-Security顶层架构-5.png)
在上图，FilterChainProxy决定应该使用哪个SecurityFilterChain。
- 只有第一个匹配的SecurityFilterChain被调用。如果请求了/api/messages/的URL，它首先匹配/api/**的SecurityFilterChain_0模式，因此只有SecurityFilterChain_0被调用，即使它也匹配SecurityFilterChainn模式。
- 如果请求的URL为/messages/，则它与/api/**的SecurityFilterChain_0模式不匹配，因此FilterChainProxy会继续尝试每个SecurityFilterChain。
- 假设没有其他SecurityFilterChain实例匹配，则调用SecurityFilterChain_n。
注意，SecurityFilterChain_0只配置了三个安全过滤器实例。然而，SecurityFilterChain_n配置了四个安全过滤器实例。需要注意的是，每个SecurityFilterChain可以是唯一的，并且可以单独配置。事实上，如果应用程序希望Spring security忽略某些请求，那么SecurityFilterChain可能没有任何安全过滤器实例。

# 安全过滤器
安全过滤器通过SecurityFilterChain API插入到FilterChainProxy中。Filter实例的顺序很重要。通常不需要知道Spring Security的Filter实例的顺序。然而，有时了解顺序是有益的。
以下是Spring安全过滤器排序的综合列表:
- ForceEagerSessionCreationFilter
- ChannelProcessingFilter
- WebAsyncManagerIntegrationFilter
- SecurityContextPersistenceFilter
- HeaderWriterFilter
- CorsFilter
- CsrfFilter
- LogoutFilter
- OAuth2AuthorizationRequestRedirectFilter
- Saml2WebSsoAuthenticationRequestFilter
- X509AuthenticationFilter
- AbstractPreAuthenticatedProcessingFilter
- CasAuthenticationFilter
- OAuth2LoginAuthenticationFilter
- Saml2WebSsoAuthenticationFilter
- UsernamePasswordAuthenticationFilter
- DefaultLoginPageGeneratingFilter
- DefaultLogoutPageGeneratingFilter
- ConcurrentSessionFilter
- DigestAuthenticationFilter
- BearerTokenAuthenticationFilter
- BasicAuthenticationFilter
- RequestCacheAwareFilter
- SecurityContextHolderAwareRequestFilter
- JaasApiIntegrationFilter
- RememberMeAuthenticationFilter
- AnonymousAuthenticationFilter
- OAuth2AuthorizationCodeGrantFilter
- SessionManagementFilter
- ExceptionTranslationFilter
- FilterSecurityInterceptor
- SwitchUserFilter

# 处理Security异常
ExceptionTranslationFilter允许将AccessDeniedException和AuthenticationException转换为HTTP响应。
ExceptionTranslationFilter作为安全过滤器之一插入到FilterChainProxy中。
下图显示了ExceptionTranslationFilter与其他组件的关系:
![Multiple SecurityFilterChain](/img/IT/SpringSecurity/Spring-Security顶层架构-6.png)
- 首先，ExceptionTranslationFilter调用FilterChain.doFilter(request, response)来调用应用程序的其余部分。
- 如果用户没有被认证或者是一个AuthenticationException，那么开始认证。
- SecurityContextHolder被清除。
- HttpServletRequest被保存，以便在身份验证成功后可以使用它重放原始请求。
- AuthenticationEntryPoint用于从客户端请求凭据。例如，它可能重定向到一个登录页面或发送一个WWW-Authenticate报头。
- 否则，如果是AccessDeniedException，则拒绝访问。调用AccessDeniedHandler来处理被拒绝的访问。
如果应用程序不抛出AccessDeniedException或AuthenticationException，则ExceptionTranslationFilter不做任何事情。

ExceptionTranslationFilter的伪代码是这样的:
``` java
try {
	filterChain.doFilter(request, response);
} catch (AccessDeniedException | AuthenticationException ex) {
	if (!authenticated || ex instanceof AuthenticationException) {
		startAuthentication();
	} else {
		accessDenied();
	}
}
```
- 正如过滤器回顾中所述，调用FilterChain.doFilter(request, response) 相当于调用应用程序的其余部分。这意味着如果应用程序的另一部分(FilterSecurityInterceptor或方法security)抛出AuthenticationException或AccessDeniedException，则在这里捕获并处理。
- 如果用户未经过身份验证或为AuthenticationException，则启动身份验证。
- 否则，拒绝访问

# Saving Requests Between Authentication
如处理安全异常中所述，当请求没有身份验证，并且请求的资源需要身份验证时，需要保存请求，以便经过身份验证的资源在身份验证成功后重新请求。在Spring Security中，这是通过使用RequestCache实现保存HttpServletRequest来完成的。
## RequestCache
HttpServletRequest保存在RequestCache中。当用户成功进行身份验证时，RequestCache用于重播原始请求。RequestCacheAwareFilter使用RequestCache来保存HttpServletRequest。
缺省情况下，使用HttpSessionRequestCache。下面的代码演示了如何定制RequestCache实现，该实现用于在命名为continue的参数存在时检查HttpSession中是否存在已保存的请求。
RequestCache只在continue参数存在时检查保存的请求。
``` java
@Bean
DefaultSecurityFilterChain springSecurity(HttpSecurity http) throws Exception {
	HttpSessionRequestCache requestCache = new HttpSessionRequestCache();
	requestCache.setMatchingRequestParameterName("continue");
	http
		// ...
		.requestCache((cache) -> cache
			.requestCache(requestCache)
		);
	return http.build();
}
```
## RequestCacheAwareFilter
RequestCacheAwareFilter使用RequestCache来保存HttpServletRequest。
