---
title: 基于Servlet的Spring Security Authorization集成
date: 2023-06-29 16:29:25
tags:
---
[参考](https://docs.spring.io/spring-security/reference/servlet/authorization/index.html)

# Http Request权限认证
Spring Security允许您在request级别对授权进行建模。
例如，使用Spring Security，可以实现`/admin`下的所有页面都需要一个*权限*，而所有其他页面只需要*身份验证*。

默认情况下，Spring Security要求对每个请求进行身份验证。也就是说，任何时候使用HttpSecurity实例时，都有必要声明自定义的授权规则。

当使用HttpSecurity实例时，至少应该设定:
``` Java
http
    .authorizeHttpRequests((authorize) -> authorize
        .anyRequest().authenticated()
    )
```
这告诉Spring Security，应用程序中的任何端点都至少需要对安全上下文进行*身份验证*才能允许访问。

## 请求授权组件工作原理
![Http Request权限认证](/img/IT/SpringSecurity/基于Servlet的Spring-Security-Authorization集成-1.png)
1. 首先，`AuthorizationFilter`构造一个`Supplier`，该`Supplier`从`SecurityContexHolder`中检索一个`Authentication`。
2. 其次，它将`Supplier<Authentication>`和`HttpServletRequest`传递给`AuthorizationManager`。`AuthorizationManager`将请求与`authorizeHttpRequests`中的模式匹配，并执行相应的规则。
3. 如果拒绝授权，则发布`AuthorizationDeniedEvent`，并抛出`AccessDeniedException`。在这种情况下，`ExceptionTranslationFilter`处理`AccessDeniedException`。
4. 如果授予访问权限，则发布`AuthorizationGrantedEvent`, `AuthorizationFilter`继续执行`FilterChain`，允许应用程序正常处理。
### AuthorizationFilter默认是最后一个
默认情况下，`AuthorizationFilter`位于*Spring Security过滤器链*的最后一个。
这意味着Spring Security的*身份验证过滤器*、*漏洞利用保护*和集成的其他过滤器**不需要授权**。如果在`AuthorizationFilter`之前添加自己的过滤器，它们也不需要授权;否则，需要授权。

当添加Spring MVC端点时，这通常变得很重要。因为它们是由DispatcherServlet执行的，而这是在AuthorizationFilter之后，所以端点需要包含在authorizeHttpRequests中才能被允许。

### 所有Dispatche都是授权的
`AuthorizationFilter`不仅在每个请求上执行，而且在每个dispatch上执行。
这意味着REQUEST dispatch需要授权，`forward`、`ERROR`和`INCLUDE`也一样。

例如，Spring MVC可以将请求转发(FORWARD)给渲染thymleaf模板的视图解析器，如下所示:
``` Java
@Controller
public class MyController {
    @GetMapping("/endpoint")
    public String endpoint() {
        return "endpoint";
    }
}
```
在这种情况下，授权执行了两次:
- 一次用于授权`/endpoint`
- 一次用于转发给Thymeleaf以呈现“endpoint”模板。
出于这个原因，你可能希望**允许所有FORWARD dispatch**。

这个原则的另一个例子是Spring Boot如何处理错误。如果容器捕获到异常，举例如下:
``` Java
@Controller
public class MyController {
    @GetMapping("/endpoint")
    public String endpoint() {
        throw new UnsupportedOperationException("unsupported");
    }
}
```
然后Boot将它分派给ERROR dispatch。
在这种情况下，授权也会发生两次:
- 一次用于授权`/endpoint`，
- 一次用于dispatch错误。
出于这个原因，**你可能希望允许所有的ERROR分派**。

### Authencation查找是延迟的
记住，`AuthorizationManager` API使用了一个`Supplier<Authentication>`。
当请求 *总是被允许* 或 *总是被拒绝* 时，这对`authorizehttprequest`很重要。在这些情况下，**不会**查询Authentication，从而让请求更快。

## 授权端点
您可以通过按优先顺序添加更多规则来配置Spring Security，使其具有不同的规则。

如果你想要求`/endpoint`只能由具有`USER`权限的终端用户访问，那么你可以这样做:
``` Java
@Bean
SecurityFilterChain web(HttpSecurity http) throws Exception {
	http
		.authorizeHttpRequests((authorize) -> authorize
			.requestMatchers("/endpoint").hasAuthority('USER')
			.anyRequest().authenticated()
		)
        // ...

	return http.build();
}
```
如上所示，可以将声明分解为模式/规则对。
`AuthorizationFilter`按照列表的顺序处理这些配置对，**仅将第一个匹配应用于请求**。这意味着即使`/**`也会匹配`/endpoint`，上述规则也不是问题。上述规则可以理解为“如果请求是`/endpoint`，那么需要`USER`权限;否则，只需要身份验证”。
Spring Security支持多种模式和规则;您还可以通过编程方式创建您自己模式和规则。

一旦获得授权，您可以使用*Security的测试支持*以以下方式进行测试:
``` Java
@WithMockUser(authorities="USER")
@Test
void endpointWhenUserAuthorityThenAuthorized() {
    this.mvc.perform(get("/endpoint"))
        .andExpect(status().isOk());
}

@WithMockUser
@Test
void endpointWhenNotUserAuthorityThenForbidden() {
    this.mvc.perform(get("/endpoint"))
        .andExpect(status().isForbidden());
}

@Test
void anyWhenUnauthenticatedThenUnauthorized() {
    this.mvc.perform(get("/any"))
        .andExpect(status().isUnauthorized())
}
```
## 请求匹配
请求匹配的方法:
- 最简单的，即匹配任何请求。
- 过URI模式进行匹配。Spring Security支持两种用于URI模式匹配的语言:Ant(如上所示)和正则表达式。

### 使用Ant进行匹配
Ant是Spring Security用来匹配请求的**默认**语言。

1. 匹配单个端点或目录，如上例`/endpoint`所示
2. 匹配端点或目录下的所有端点
    假设您不希望匹配`/endpoint`端点，而是希望匹配`/resource`**目录下的所有端点**。在这种情况下，您可以执行以下操作:
    ``` java
    http
        .authorizeHttpRequests((authorize) -> authorize
            .requestMatchers("/resource/**").hasAuthority("USER")
            .anyRequest().authenticated()
        )
    ```
    上述规则可以理解为“如果请求是`/resource`或其某个子目录，需要`USER`权限;否则，只需要身份验证。”
3. 请求中提取**路径值**
   如下所示:
    ``` java
    http
        .authorizeHttpRequests((authorize) -> authorize
            .requestMatchers("/resource/{name}").access(new WebExpressionAuthorizationManager("#name == authentication.name"))
            .anyRequest().authenticated()
        )
    ```

配置好授权以后，可以使用*Security的测试支持*以以下方式进行测试:
``` java
@WithMockUser(authorities="USER")
@Test
void endpointWhenUserAuthorityThenAuthorized() {
    this.mvc.perform(get("/resource/jon"))
        .andExpect(status().isOk());
}

@WithMockUser
@Test
void endpointWhenNotUserAuthorityThenForbidden() {
    this.mvc.perform(get("/resource/jon"))
        .andExpect(status().isForbidden());
}

@Test
void anyWhenUnauthenticatedThenUnauthorized() {
    this.mvc.perform(get("/any"))
        .andExpect(status().isUnauthorized())
}
```

### 使用正则表达式进行匹配
Spring Security支持根据正则表达式匹配请求。如果您想在子目录上应用比`**`更严格的匹配条件，这将很有用。

例如，考虑一个包含用户名和所有用户名必须是字母数字的规则的路径。你可以使用`RegexRequestMatcher`来实现规则，像这样:
``` Java
http
    .authorizeHttpRequests((authorize) -> authorize
        .requestMatchers(RegexRequestMatcher.regexMatcher("/resource/[A-Za-z0-9]+")).hasAuthority("USER")
        .anyRequest().denyAll()
    )
```
### 使用HTTP Method进行匹配
您也可以通过*HTTP方法*匹配规则。在通过授予的权限进行授权时，比如授予*读*或*写*权限时，这很有用。

如果要求所有`GET`都有*读*权限，所有`POST`都有*写*权限，你可以这样做:
``` Java
http
    .authorizeHttpRequests((authorize) -> authorize
        .requestMatchers(HttpMethod.GET).hasAuthority("read")
        .requestMatchers(HttpMethod.POST).hasAuthority("write")
        .anyRequest().denyAll()
    )
```
上述规则可以理解为:“如果请求是`GET`，那么需要*读*权限;或者如果请求是`POST`，则需要*写*权限;否则，拒绝请求。”

> **默认情况下拒绝请求**是一种推荐的安全实践，因为它将规则集转换为允许列表。

配置好授权以后，可以使用*Security的测试支持*以以下方式进行测试:
``` Java
@WithMockUser(authorities="read")
@Test
void getWhenReadAuthorityThenAuthorized() {
    this.mvc.perform(get("/any"))
        .andExpect(status().isOk());
}

@WithMockUser
@Test
void getWhenNoReadAuthorityThenForbidden() {
    this.mvc.perform(get("/any"))
        .andExpect(status().isForbidden());
}

@WithMockUser(authorities="write")
@Test
void postWhenWriteAuthorityThenAuthorized() {
    this.mvc.perform(post("/any").with(csrf()))
        .andExpect(status().isOk())
}

@WithMockUser(authorities="read")
@Test
void postWhenNoWriteAuthorityThenForbidden() {
    this.mvc.perform(get("/any").with(csrf()))
        .andExpect(status().isForbidden());
}
```
### 根据分派器(Dispatcher)类型匹配
> !暂不支持XML!

如前所述，Spring Security默认授权所有调度程序类型。即使在REQUESTdispatch上建立的安全上下文延续到后续dispatch，细微的不匹配有时也会导致意外的`AccessDeniedException`。

为了解决这个问题，你可以配置Spring Security Java配置来允许分派器(Dispatcher)类型，比如`FORWARD`和`ERROR`，如下所示:
``` Java
http
    .authorizeHttpRequests((authorize) -> authorize
        .dispatcherTypeMatchers(DispatcherType.FORWARD, DispatcherType.ERROR).permitAll()
        .requestMatchers("/endpoint").permitAll()
        .anyRequest().denyAll()
    )
```

### 使用自定义匹配器
> !暂不支持XML!
在Java配置中，你可以创建自己的`RequestMatcher`，并像这样提供给DSL:
```java
RequestMatcher printview = (request) -> request.getParameter("print") != null;
http
    .authorizeHttpRequests((authorize) -> authorize
        .requestMatchers(printview).hasAuthority("print")
        .anyRequest().authenticated()
    )
```
> 因为`RequestMatcher`是一个功能接口，所以您可以在DSL中将其作为lambda提供。
> 但是，如果要从请求中提取值，则需要有一个具体的类，因为这需要重写默认方法。

配置好授权以后，可以使用*Security的测试支持*以以下方式进行测试:
``` Java
@WithMockUser(authorities="print")
@Test
void printWhenPrintAuthorityThenAuthorized() {
    this.mvc.perform(get("/any?print"))
        .andExpect(status().isOk());
}

@WithMockUser
@Test
void printWhenNoPrintAuthorityThenForbidden() {
    this.mvc.perform(get("/any?print"))
        .andExpect(status().isForbidden());
}
```

## 为Request授权
匹配请求后，用授权规则就可以对其进行授权。

下面是DSL中内置的授权规则:
- `permitAll`: 请求不需要授权，是一个公共端点;注意，在这种情况下，永远不会从Session中检索Authentication
- `denyAll`: 请求在任何情况下都不被允许;注意，在这种情况下，永远不会从Session中检索Authentication
- `hasAuthority`: 请求要求认证具有与给定值匹配的G`rantedAuthority`
- `hasRole`: `hasAuthority`的快捷方式，前缀为`ROLE_`或任何被配置为默认前缀的东西
- `hasAnyAuthority`: 请求要求认证具有与任何给定值匹配的`GrantedAuthority`
- `hasAnyRole`: `hasAnyAuthority`的快捷方式，它以`ROLE_`或任何被配置为默认前缀的前缀为前缀
- `access`: 请求使用这个自定义`AuthorizationManager`来确定访问权限

举例如下：
``` Java
@Bean
SecurityFilterChain web(HttpSecurity http) throws Exception {
	http
		// ...
		.authorizeHttpRequests(authorize -> authorize                                  
            .dispatcherTypeMatchers(FORWARD, ERROR).permitAll() 
			.requestMatchers("/static/**", "/signup", "/about").permitAll()         
			.requestMatchers("/admin/**").hasRole("ADMIN")                             
			.requestMatchers("/db/**").access(allOf(hasAuthority('db'), hasRole('ADMIN')))   
			.anyRequest().denyAll()                                                
		);

	return http.build();
}
```
说明如下：
- 授权分派`FORWARD`和`ERROR`以允许Spring MVC呈现视图、Spring Boot呈现error
- 指定任何用户都可以访问的多个URL模式。具体来说，如果URL以“/resources/”开头，等于“/signup”或等于“/about”，则任何用户都可以访问。
- 任何以“/admin/”开头的URL将被限制为具有“ROLE_ADMIN”角色的用户才能访问。**注意，由于调用的是hasRole方法，因此不需要指定“ROLE_”前缀**。
- 任何以“/db/”开头的URL都要求用户具有“db”权限以及“ROLE_ADMIN”权限。**注意，由于使用的是hasRole表达式，因此不需要指定“ROLE_”前缀**。
- 任何未匹配的URL都被拒绝访问。如果不想意外忘记更新授权规则，这是一个很好的策略。


## 使用授权数据库、策略代理或其他服务
如果您希望配置Spring Security以使用单独的服务进行授权，您可以创建自己的AuthorizationManager并将其与anyRequest匹配。

首先，自定义的`AuthorizationManager`举例如下:
``` Java
@Component
public final class OpenPolicyAgentAuthorizationManager implements AuthorizationManager<RequestAuthorizationContext> {
    @Override
    public AuthorizationDecision check(Supplier<Authentication> authentication, RequestAuthorizationContext context) {
        // make request to Open Policy Agent
    }
}
```
通过以下方式将其添加到Spring Security:
``` Java
@Bean
SecurityFilterChain web(HttpSecurity http, AuthorizationManager<RequestAuthorizationContext> authz) throws Exception {
	http
		// ...
		.authorizeHttpRequests((authorize) -> authorize
            .anyRequest().access(authz)
		);

	return http.build();
}
```
## 推荐permitAll而非的忽视
当您拥有**静态资源**时，很容易将SecurityFilterChain配置为忽略这些值,比如使用下文的`securityMatcher`。
更安全的方法是像这样使用`permitAll`来允许它们:
``` Java
http
    .authorizeHttpRequests((authorize) -> authorize
        .requestMatchers("/css/**").permitAll()
        .anyRequest().authenticated()
    )
```

## 安全(Security)匹配器
使用securityMatchers来确定是否应该将给定的`HttpSecurity`应用于给定的请求。
同样，我们可以使用requestMatchers来确定应该应用于给定请求的授权规则。
> 如果`securityMatcher`不匹配，则SecurityFilterChain将不被执行。
举例如下:
``` Java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

	@Bean
	public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
		http
			.securityMatcher("/api/**")                            
			.authorizeHttpRequests(authorize -> authorize
				.requestMatchers("/user/**").hasRole("USER")       
				.requestMatchers("/admin/**").hasRole("ADMIN")     
				.anyRequest().authenticated()                      
			)
			.formLogin(withDefaults());
		return http.build();
	}
}
```

# 方法级别授权
> 默认情况下，Spring Boot Starter Security不激活方法级授权。
> 需要程序中通过用`@EnableMethodSecurity`注解来激活它，任何@Configuration类都可以，如下所示:
> ``` Java
> @EnableMethodSecurity
> ```
> 之后， 可以对Spring管理的类和方法使用`@PreAuthorize`, `@PostAuthorize`, `@PreFilter`, 和`@PostFilter`注解，以授权方法调用，包括输入参数和返回值。

## 方法级别授权工作原理
Spring Security的方法授权支持很方便:
- 抽取细粒度的授权逻辑;例如，当授权决策需要使用方法参数和返回值时。
- 在服务层加强安全性
- 在风格上倾向于基于注解的配置，而不是基于`HttpSecurity`的配置

而且由于Method Security是使用`Spring AOP`构建的，因此您可以根据需要使用它的所有特性来覆盖Spring Security的默认值。

方法授权是*方法前*和*方法后*授权的组合。考虑一个以以下方式进行注释的Service bean:
``` Java
@Service
public class MyCustomerService {
    @PreAuthorize("hasAuthority('permission:read')")
    @PostAuthorize("returnObject.owner == authentication.name")
    public Customer readCustomer(String id) { ... }
}
```
当方法安全被激活时，对`MyCustomerService#readCustomer`的调用可能看起来像这样:
![Http Request权限认证](/img/IT/SpringSecurity/基于Servlet的Spring-Security-Authorization集成-2.png)

# 域对象安全访问控制(ACL)