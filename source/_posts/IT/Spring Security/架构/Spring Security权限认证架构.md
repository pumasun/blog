---
title: Spring Security权限认证架构
date: 2023-06-29 16:55:03
tags:
- Spring Security
- Architecture
category:
- IT
- Spring Security
---

[参考](https://docs.spring.io/spring-security/reference/servlet/authorization/architecture.html)

# 授权
身份验证讨论了所有`Authentication`的实现如何存储`GrantedAuthority`对象列表。这代表被授予主体的权限。`GrantedAuthority`对象由`AuthenticationManager`插入到`Authentication`对象中，然后在做出授权决策时由AccessDecisionManager实例读取。
`GrantedAuthority`接口只有一个方法:
``` Java
String getAuthority();
```
`AuthorizationManager`实例使用此方法来获取`GrantedAuthority`的精确字符串表示形式。通过将表示形式作为字符串返回，大多数`AuthorizationManager`实现可以很容易地“读取”`GrantedAuthority`。如果`GrantedAuthority`不能精确地表示为字符串，则认为该`GrantedAuthority`是“复杂的”，并且getAuthority()必须返回null。

复杂`GrantedAuthority`的一个示例是存储应用于不同客户账号的操作和权限阈值列表的实现。将这个复杂的`GrantedAuthority`表示为字符串是非常困难的。因此，`getAuthority()`方法应该返回null。这向任何`AuthorizationManager`表明，它需要支持特定的`GrantedAuthority`实现才能理解其内容。

Spring Security包括一个具体的`GrantedAuthority`实现:`SimpleGrantedAuthority`。该实现允许将任何用户指定的字符串转换为`GrantedAuthority`。Spring Security架构中包含的所有`AuthenticationProvider`实例都使用`SimpleGrantedAuthority`填充身份验证对象。

默认情况下，基于角色的授权规则的前缀为`ROLE_`。这意味着，如果有一个授权规则要求安全上下文具有`USER`角色，Spring security将在默认情况下查找返回`ROLE_USER`的`GrantedAuthority#getAuthority`。

你可以用`GrantedAuthorityDefaults`自定义它。`GrantedAuthorityDefaults`的存在是为了允许自定义前缀，用于基于角色的授权规则。

你可以通过暴露`GrantedAuthorityDefaults` bean来配置授权规则，以使用不同的前缀，如下所示:

Custom MethodSecurityExpressionHandler
``` Java
@Bean
static GrantedAuthorityDefaults grantedAuthorityDefaults() {
	return new GrantedAuthorityDefaults("MYPREFIX_");
}
```
> 您可以使用**静态方法**公开`GrantedAuthorityDefaults`，以确保Spring在初始化Spring Security的方法安全认证(Method-Security)的@Configuration类之前发布它。

# 调用处理
Spring Security提供了拦截器来控制对安全对象(如方法调用或web请求)的访问。
- 调用前决策由`AuthorizationManager`实例做出，决定是否允许调用继续执行。
- 调用后决策由`AuthorizationManager`实例做出，决定是否返回给定值。

# AuthorizationManager
> `AuthorizationManager`取代了`AccessDecisionManager`和`AccessDecisionVoter`。
> 建议自定义`AccessDecisionManager`或`AccessDecisionVoter`的应用程序改为使用`AuthorizationManager`。

`AuthorizationManagers`由Spring Security的基于请求、基于方法和基于消息的授权组件调用，并负责做出最终的访问控制决策。AuthorizationManager接口包含两个方法:
``` Java
AuthorizationDecision check(Supplier<Authentication> authentication, Object secureObject);

default AuthorizationDecision verify(Supplier<Authentication> authentication, Object secureObject)
        throws AccessDeniedException {
    // ...
}
```
`AuthorizationManager`的`check`方法被传入了做出授权决策所需的所有相关信息。特别是，传递安全对象(secure Object)可以检查实际安全对象调用中包含的那些参数。
例如，我们假设安全对象是一个`MethodInvocation`。可以很容易地查询`MethodInvocation`中的任何用户参数，然后在`AuthorizationManager`中实现某种安全逻辑，以确保允许主体操作该参数。
- 如果授予访问权限，实现将返回 *是* (positive)的AuthorizationDecision
- 如果拒绝访问，则返 *否* (negative)的AuthorizationDecision
- 如果放弃做出决策，则返回 *空* (null)的AuthorizationDecision。

`verify`调用将执行检查，并在*否*AuthorizationDecision的情况下抛出`AccessDeniedException`。

# 基于委托的AuthorizationManager实现
虽然用户可以实现他们自己的`AuthorizationManager`来控制授权的各个方面，Spring Security还提供了一个可以与各个`AuthorizationManager`协作的`委托的AuthorizationManager`。

`RequestMatcherDelegatingAuthorizationManager`将为请求匹配最合适的`委托的AuthorizationManager`。对于方法安全控制，你可以使用`AuthorizationManagerBeforeMethodInterceptor`和`AuthorizationManagerAfterMethodInterceptor`。

授权管理器实现说明了相关的类:

![授权管理器实现](img/IT/SpringSecurity/Spring-Security权限认证架构-1.png)

这样就可以轮询AuthorizationManager实现的组合来进行授权决策。

# AuthorityAuthorizationManager
Spring Security提供的最常见的`AuthorizationManager`是`AuthorityAuthorizationManager`。它配置了一组给定的授权，用于在当前`Authentication`中对比。
- 如果身份验证包含任何配置的授权，它将返回*是*AuthorizationDecision。
- 否则它将返回一个*否*AuthorizationDecision。

# AuthenticatedAuthorizationManager
`AuthenticatedAuthorizationManager`可用于区分匿名(anonymous)、完全身份验证(fully-authenticated)和记住我(remember-me)身份验证的用户。
许多站点允许在`remember-me`身份验证下进行某些**有限**的访问，但要求用户通过**登录来确认其身份以获得完全访问权限**。

#  AuthorizationManagers
AuthenticationManagers中还有一些有用的静态工厂，可以将单个AuthenticationManagers组合成更复杂的表达式。

# 自定义授权管理器
开发者可以自定义`AuthorizationManager`，并且可以在其中放入您想要的任何访问控制逻辑。它可能特定于某个应用程序(与业务逻辑相关)，实现某些安全管理逻辑。例如，您可以创建一个实现，来查询*Open Policy Agent*或这实现自己的授权数据库。

# 适配`AccessDecisionManager`和`AccessDecisionVoter`
在`AuthorizationManager`之前，Spring Security发布了`AccessDecisionManager`和`AccessDecisionVoter`。

在某些情况下，比如迁移旧的应用程序，可能需要引入调用`AccessDecisionManager`或`AccessDecisionVoter`的`AuthorizationManager`。
要调用现有的`AccessDecisionManager`，你可以这样做:
``` Java
@Component
public class AccessDecisionManagerAuthorizationManagerAdapter implements AuthorizationManager {
    private final AccessDecisionManager accessDecisionManager;
    private final SecurityMetadataSource securityMetadataSource;

    @Override
    public AuthorizationDecision check(Supplier<Authentication> authentication, Object object) {
        try {
            Collection<ConfigAttribute> attributes = this.securityMetadataSource.getAttributes(object);
            this.accessDecisionManager.decide(authentication.get(), object, attributes);
            return new AuthorizationDecision(true);
        } catch (AccessDeniedException ex) {
            return new AuthorizationDecision(false);
        }
    }

    @Override
    public void verify(Supplier<Authentication> authentication, Object object) {
        Collection<ConfigAttribute> attributes = this.securityMetadataSource.getAttributes(object);
        this.accessDecisionManager.decide(authentication.get(), object, attributes);
    }
}
``` 
然后将它添加到`SecurityFilterChain`中。

或者只调用`AccessDecisionVoter`，你可以这样做:
``` Java
@Component
public class AccessDecisionVoterAuthorizationManagerAdapter implements AuthorizationManager {
    private final AccessDecisionVoter accessDecisionVoter;
    private final SecurityMetadataSource securityMetadataSource;

    @Override
    public AuthorizationDecision check(Supplier<Authentication> authentication, Object object) {
        Collection<ConfigAttribute> attributes = this.securityMetadataSource.getAttributes(object);
        int decision = this.accessDecisionVoter.vote(authentication.get(), object, attributes);
        switch (decision) {
        case ACCESS_GRANTED:
            return new AuthorizationDecision(true);
        case ACCESS_DENIED:
            return new AuthorizationDecision(false);
        }
        return null;
    }
}
```
然后将它添加到`SecurityFilterChain`中。

# 可继承的多层角色
一个常见的需求是，应用程序中的特定角色应该自动“包含”其他角色。
例如，在具有`admin`和`user`角色概念的应用程序中，可能希望*管理员*能够执行*普通用户*可以执行的所有操作。要实现这一点:
- 可以确保所有*管理员用户*也被分配了`user`角色
- 或者，可以修改每个访问控制定义，要求包括`user`角色和`admin`角色
如果应用程序中有很多不同的角色，这可能会变得相当复杂。

使用角色层次结构可以配置哪些角色(或权限)应该包含其他角色(或权限)。Spring Security的`RoleVoter`的扩展版本`RoleHierarchyVoter`配置了一个`RoleHierarchy`，从中可以获得分配给用户的所有“可访问权限”。典型的配置可能是这样的:

``` Java
@Bean
static RoleHierarchy roleHierarchy() {
    var hierarchy = new RoleHierarchyImpl();
    hierarchy.setHierarchy("ROLE_ADMIN > ROLE_STAFF\n" +
            "ROLE_STAFF > ROLE_USER\n" +
            "ROLE_USER > ROLE_GUEST");
}

// and, if using method security also add
@Bean
static MethodSecurityExpressionHandler methodSecurityExpressionHandler(RoleHierarchy roleHierarchy) {
	DefaultMethodSecurityExpressionHandler expressionHandler = new DefaultMethodSecurityExpressionHandler();
	expressionHandler.setRoleHierarchy(roleHierarchy);
	return expressionHandler;
}
```
> `RoleHierarchy` bean配置还没有移植到`@EnableMethodSecurity`。因此，本例使用的是`AccessDecisionVoter`。
> 如果您需要`RoleHierarchy`来支持*方法安全性*，请继续使用`@EnableGlobalMethodSecurity`。

这里我们有四个层次结构的角色:`ROLE_ADMIN`->`ROLE_STAFF`->`ROLE_USER`->`ROLE_GUEST`。有用`ROLE_ADMIN`的用户，在通过适配了RoleHierarchyVoter的AuthorizationManager进行授权验证时，将表现得好像他们拥有所有四个角色。符号`>`可以理解为`包括`。

角色层次结构为简化应用程序的访问控制配置数据，或减少需要分配给用户的权限数量，提供了一种方便的方法。

对于更加复杂的业务场景，在特定访问权限与角色之间，开发者可能希望定义一个逻辑映射，在加载用户信息时进行两者的转换。

# 遗留的授权验证组件
参考[官方文档](https://docs.spring.io/spring-security/reference/servlet/authorization/architecture.html#authz-legacy-note)