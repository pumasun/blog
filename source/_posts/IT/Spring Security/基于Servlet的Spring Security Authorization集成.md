---
title: 基于Servlet的Spring Security Authorization集成
date: 2023-06-29 16:29:25
tags:
---
[参考](https://docs.spring.io/spring-security/reference/servlet/authorization/index.html)

# Http Request权限认证
Spring Security允许开发者在request级别对授权进行建模。
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
3. 如果拒绝授权，则生成`AuthorizationDeniedEvent`，并抛出`AccessDeniedException`。在这种情况下，`ExceptionTranslationFilter`处理`AccessDeniedException`。
4. 如果授予访问权限，则生成`AuthorizationGrantedEvent`, `AuthorizationFilter`继续执行`FilterChain`，允许应用程序正常处理。
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
出于这个原因，开发者可能希望**允许所有FORWARD dispatch**。

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
出于这个原因，开发者可能希望**允许所有的ERROR分派**。

### Authencation查找是延迟的
记住，`AuthorizationManager` API使用了一个`Supplier<Authentication>`。
当请求 *总是被允许* 或 *总是被拒绝* 时，这对`authorizehttprequest`很重要。在这些情况下，**不会**查询Authentication，从而让请求更快。

## 授权端点
开发者可以通过按优先顺序添加更多规则来配置Spring Security，使其具有不同的规则。

如果想要求`/endpoint`只能由具有`USER`权限的终端用户访问，那么可以这样做:
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
Spring Security支持多种模式和规则;开发者还可以通过编程方式创建开发者自己模式和规则。

一旦获得授权，开发者可以使用*Security的测试支持*以以下方式进行测试:
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
    假设开发者不希望匹配`/endpoint`端点，而是希望匹配`/resource`**目录下的所有端点**。在这种情况下，开发者可以执行以下操作:
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
Spring Security支持根据正则表达式匹配请求。如果开发者想在子目录上应用比`**`更严格的匹配条件，这将很有用。

例如，考虑一个包含用户名和所有用户名必须是字母数字的规则的路径。可以使用`RegexRequestMatcher`来实现规则，像这样:
``` Java
http
    .authorizeHttpRequests((authorize) -> authorize
        .requestMatchers(RegexRequestMatcher.regexMatcher("/resource/[A-Za-z0-9]+")).hasAuthority("USER")
        .anyRequest().denyAll()
    )
```
### 使用HTTP Method进行匹配
开发者也可以通过*HTTP方法*匹配规则。在通过授予的权限进行授权时，比如授予*读*或*写*权限时，这很有用。

如果要求所有`GET`都有*读*权限，所有`POST`都有*写*权限，可以这样做:
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

为了解决这个问题，可以配置Spring Security Java配置来允许分派器(Dispatcher)类型，比如`FORWARD`和`ERROR`，如下所示:
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
在Java配置中，可以创建自己的`RequestMatcher`，并像这样提供给DSL:
```java
RequestMatcher printview = (request) -> request.getParameter("print") != null;
http
    .authorizeHttpRequests((authorize) -> authorize
        .requestMatchers(printview).hasAuthority("print")
        .anyRequest().authenticated()
    )
```
> 因为`RequestMatcher`是一个功能接口，所以开发者可以在DSL中将其作为lambda提供。
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
如果开发者希望配置Spring Security以使用单独的服务进行授权，开发者可以创建自己的AuthorizationManager并将其与anyRequest匹配。

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
当开发者拥有**静态资源**时，很容易将SecurityFilterChain配置为忽略这些值,比如使用下文的`securityMatcher`。
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
> 默认情况下，Spring Boot Starter Security不启用方法级授权。
> 需要程序中通过用`@EnableMethodSecurity`注解来启用它，任何@Configuration类都可以，如下所示:
> ``` Java
> @EnableMethodSecurity
> ```
> 之后， 可以对Spring管理的类和方法使用`@PreAuthorize`, `@PostAuthorize`, `@PreFilter`, 和`@PostFilter`注解，以授权方法调用，包括输入参数和返回值。

## 方法级别授权工作原理
Spring Security的方法授权支持很方便:
- 抽取细粒度的授权逻辑;例如，当授权决策需要使用方法参数和返回值时。
- 在服务层加强安全性
- 在风格上倾向于基于注解的配置，而不是基于`HttpSecurity`的配置

而且由于Method Security是使用`Spring AOP`构建的，因此开发者可以根据需要使用它的所有特性来覆盖Spring Security的默认值。

方法授权是*方法前*和*方法后*授权的组合。考虑一个以以下方式进行注解的Service bean:
``` Java
@Service
public class MyCustomerService {
    @PreAuthorize("hasAuthority('permission:read')")
    @PostAuthorize("returnObject.owner == authentication.name")
    public Customer readCustomer(String id) { ... }
}
```
当方法安全被启用时，对`MyCustomerService#readCustomer`的调用可能看起来像这样:
![Http Request权限认证](/img/IT/SpringSecurity/基于Servlet的Spring-Security-Authorization集成-2.png)
1. Spring AOP调用`readCustomer`的代理方法。在代理的其他顾问中，它调用与`@PreAuthorize`切入点匹配的`AuthorizationManagerBeforeMethodInterceptor`
2. 拦截器调用`PreAuthorizeAuthorizationManager#check`
3. 授权管理器使用`MethodSecurityExpressionHandler`来解析注解的SpEL表达式，并从包含`Supplier<Authentication>`和`MethodInvocation`的`MethodSecurityExpressionRoot`中构造相应的`EvaluationContext`。
4. 拦截器使用这个上下文对表达式求值;具体来说，它从`Supplier`读取`Authentication`，并检查它是否在其权限集合中具有读取权限
5. 如果评估通过，那么Spring AOP将继续调用该方法。
6. 如果没有，拦截器生成一个`AuthorizationDeniedEvent`并抛出一个`AccessDeniedException`，由`ExceptionTranslationFilter`捕获并向响应返回一个403状态码
7. 方法返回后，Spring AOP调用一个与`@PostAuthorize`切入点匹配的`AuthorizationManagerAfterMethodInterceptor`，操作与上面相同，但是使用的是`PostAuthorizeAuthorizationManager`
8. 如果计算通过(在本例中，返回值属于登录的用户)，处理将继续正常进行
9. 如果没有，则拦截器生成`AuthorizationDeniedEvent`并抛出`AccessDeniedException`，由`ExceptionTranslationFilter`捕获并向响应返回一个403状态码

> 如果不是在HTTP请求的上下文中调用该方法，需要自己处理`AccessDeniedException`

### 多个注解是顺序执行的
如上所示，如果方法调用涉及多个Method Security注解，这些注解将被依次执行。它们可以被认为是"与逻辑"关系。换句话说，要授权调用，所有注解检查都需要通过授权。

### 不支持重复注解
也就是说，不支持在同一方法上定义重复相同的注解。例如，不能在同一个方法上使用两次@PreAuthorize。

相反，使用SpEL的布尔(boolean)支持或使用它对委托的Bean的支持，在独立的Bean中实现需要重复注解实现的逻辑。

### 每个注解都有自己的切入点
每个注解都有自己的切入点实例，该实例会查找该注解或对应的元注解(meta-annotation)，从该方法及其封装类开始遍历其全部对象继承树。

### 每个注解都有自己的方法拦截器
每个注解都有自己专用的方法拦截器。这样做的原因是为了使事情更加可组合。例如，如果需要，可以禁用Spring Security默认设置，只生成`@PostAuthorize`方法拦截器。

方法拦截器如下:
- 对于`@PreAuthorize`, Spring Security使用`AuthenticationManagerBeforeMethodInterceptor#preAuthorization`，在其中使用`PreAuthorizeAuthorizationManager`
- 对于`@PostAuthorize`, Spring Security使用`AuthenticationManagerAfterMethodInterceptor#postAuthorize`，在其中使用`PostAuthorizeAuthorizationManager`
- 对于`@PreFilter`, Spring Security使用`PreFilterAuthorizationMethodInterceptor`
- 对于`@PostFilter`, Spring Security使用`PostFilterAuthorizationMethodInterceptor`
- 对于`@Secured`, Spring Security使用`AuthenticationManagerBeforeMethodInterceptor#secured`，在其中使用`SecuredAuthorizationManager`
- 对于`JSR-250注解`，Spring Security使用`AuthenticationManagerBeforeMethodInterceptor#jsr250`，在其中使用`Jsr250AuthorizationManager`

一般来说，可以认为当添加@EnableMethodSecurity时，Spring Security生成了一下拦截器:
``` Java
@Bean
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
static Advisor preAuthorizeMethodInterceptor() {
    return AuthorizationManagerBeforeMethodInterceptor.preAuthorize();
}

@Bean
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
static Advisor postAuthorizeMethodInterceptor() {
    return AuthorizationManagerAfterMethodInterceptor.postAuthorize();
}

@Bean
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
static Advisor preFilterMethodInterceptor() {
    return AuthorizationManagerBeforeMethodInterceptor.preFilter();
}

@Bean
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
static Advisor postFilterMethodInterceptor() {
    return AuthorizationManagerAfterMethodInterceptor.postFilter();
}
```

### 建议使用授予权限而非使用复杂的SpEL表达式
开发者可能经常不由自主的引入下面这样的表达式：
``` Java
@PreAuthorize("hasAuthority('permission:read') || hasRole('ADMIN')")
```
然而，也可以采用将权限*permission:read*授予角色*ROLE_ADMIN*的方法。比如采用`RoleHierarchy`的方式:
``` Java
@Bean
static RoleHierarchy roleHierarchy() {
    return new RoleHierarchyImpl("ROLE_ADMIN > permission:read");
}
```
然后在`MethodSecurityExpressionHandler`实例中设置它。这样就有了一个更简单的`@PreAuthorize`表达式，如下所示:
``` Java
@PreAuthorize("hasAuthority('permission:read')")
```
或者，在可能的情况下，将特定于应用程序的授权逻辑调整为在登录时授予的用户权限。

## 比较请求级和方法级授权
什么时候应该使用方法级授权而不是请求级授权?其中一些可以归结为习惯;但是，请参考一下优势列表来做出决定：
<table>
    <tr>
        <th></th>
        <th>请求级 request-level</th>
        <th>方法级 method-level</th>
    </tr>
    <tr>
        <th>授权类型</th>
        <td>粗粒度的</td>
        <td>细粒度的</td>
    </tr>
    <tr>
        <th>配置位置</th>
        <td>在Config类中声明</td>
        <td>限定在Method声明中</td>
    </tr>
    <tr>
        <th>配置风格</th>
        <td>DSL</td>
        <td>注解</td>
    </tr>
    <tr>
        <th>授权定义</th>
        <td>程序化的</td>
        <td>SpEL</td>
    </tr>
</table>

## 使用注解进行授权
Spring Security启用方法级授权支持的主要方式是通过可以添加到方法、类和接口中的注解。

### 使用@PreAuthorize授权方法调用
当方法安全启用时，可以在方法上使用`@PreAuthorize`注解，如下所示:
``` Java
@Component
public class BankService {
	@PreAuthorize("hasRole('ADMIN')")
	public Account readAccount(Long id) {
        // ... is only invoked if the `Authentication` has the `ROLE_ADMIN` authority
	}
}
```
这意味着只有当提供的表达式`hasRole('ADMIN')`通过时才能调用该方法。
可以使用以下代码测试这个类:
``` Java
@Autowired
BankService bankService;

@WithMockUser(roles="ADMIN")
@Test
void readAccountWithAdminRoleThenInvokes() {
    Account account = this.bankService.readAccount("12345678");
    // ... assertions
}

@WithMockUser(roles="WRONG")
@Test
void readAccountWithWrongRoleThenAccessDenied() {
    assertThatExceptionOfType(AccessDeniedException.class).isThrownBy(
        () -> this.bankService.readAccount("12345678"));
}
```
> `@PreAuthorize`也可以是元注解，在类或接口级别定义，并使用SpEL授权表达式。

虽然`@PreAuthorize`对于声明所需的权限非常有帮助，但它也可以用于计算涉及方法参数的更复杂的表达式。
``` Java
@Component
public class BankService {
	@PreAuthorize("#usename == authentication.name")
	public Account readAccount(String username) {
        // ... is only invoked if the `Authentication#getName` equals username parameter.
	}
}
```
上面的代码段通过比较`username`参数与`Authentication#getName`来确保用户只能请求属于他们的账户。
结果是，只有当请求参数中的username与登录的用户名匹配时，才会调用上述方法。如果没有，Spring Security将抛出一个`AccessDeniedException`并返回一个403状态码。

### 使用`@PostAuthorize`的授权方法结果
当方法安全启用时，可以在方法上使用`@PostAuthorize`注解，如下所示:
``` Java
@Component
public class BankService {
	@PostAuthorize("returnObject.owner == authentication.name")
	public Account readAccount(Long id) {
        // ... is only returned if the `Account` belongs to the logged in user
	}
}
```
这意味着该方法只能在提供的表达式`returnObject.owner == authentication.name`通过时返回值。returnbject表示要返回的Account对象。

可以使用以下代码测试这个类:
``` Java
@Autowired
BankService bankService;

@WithMockUser(username="owner")
@Test
void readAccountWhenOwnedThenReturns() {
    Account account = this.bankService.readAccount("12345678");
    // ... assertions
}

@WithMockUser(username="wrong")
@Test
void readAccountWhenNotOwnedThenAccessDenied() {
    assertThatExceptionOfType(AccessDeniedException.class).isThrownBy(
        () -> this.bankService.readAccount("12345678"));
}
```
> `@PostAuthorize`也可以是元注解，在类或接口级别定义，并使用SpEL授权表达式。
`@PostAuthorize`在防御不安全的直接对象引用时特别有用。实际上，它可以被定义为元注解，如下所示:
``` Java
@Target({ ElementType.METHOD, ElementType.TYPE })
@Retention(RetentionPolicy.RUNTIME)
@PostAuthorize("returnObject.owner == authentication.name")
public @interface RequireOwnership {}
```
可以将Service的注解如下修改：
``` Java
@Component
public class BankService {
	@RequireOwnership
	public Account readAccount(Long id) {
        // ... is only returned if the `Account` belongs to the logged in user
	}
}
```
结果是，只有当Account的所有者属性与登录的用户名匹配时，上述方法才会返回Account。如果没有，Spring Security将抛出一个`AccessDeniedException`并返回一个403状态码。

### 用@PreFilter过滤方法参数
> `@PreFilter`还不支持kotlin特定的数据类型
当方法安全启用时，可以在方法上使用`@PreFilter`注解，如下所示:
``` Java
@Component
public class BankService {
	@PreFilter("filterObject.owner == authentication.name")
	public Collection<Account> updateAccounts(Account... accounts) {
        // ... `accounts` will only contain the accounts owned by the logged-in user
        return updated;
	}
}
```
这意味着从Accounts中过滤出任何不满足`filterObject.owner == authentication.name`的值。filterObject表示帐户中的每个帐户，用于测试每个帐户。
可以使用以下代码测试这个类:
``` Java
@Autowired
BankService bankService;

@WithMockUser(username="owner")
@Test
void updateAccountsWhenOwnedThenReturns() {
    Account ownedBy = ...
    Account notOwnedBy = ...
    Collection<Account> updated = this.bankService.updateAccounts(ownedBy, notOwnedBy);
    assertThat(updated).containsOnly(ownedBy);
}
```
> @PreFilter也可以是元注解，在类或接口级别定义，并使用SpEL授权表达式。
@PreFilter支持数组、集合、映射和流(只要流是打开的)。
例如，上面的updateAccounts声明将以与下面四个声明相同的方式起作用:
``` Java
@PreFilter("filterObject.owner == authentication.name")
public Collection<Account> updateAccounts(Account[] accounts)

@PreFilter("filterObject.owner == authentication.name")
public Collection<Account> updateAccounts(Collection<Account> accounts)

@PreFilter("filterObject.value.owner == authentication.name")
public Collection<Account> updateAccounts(Map<String, Account> accounts)

@PreFilter("filterObject.owner == authentication.name")
public Collection<Account> updateAccounts(Stream<Account> accounts)
```
结果是，上述方法将只具有其所有者属性与登录用户名匹配的Account实例。

### 使用@PostFilter过滤方法结果
> @PostFilter还不支持kotlin特定的数据类型
当方法安全启用时，可以在方法上使用`@PostFilter`注解，如下所示:
```
@Component
public class BankService {
	@PostFilter("filterObject.owner == authentication.name")
	public Collection<Account> readAccounts(String... ids) {
        // ... the return value will be filtered to only contain the accounts owned by the logged-in user
        return accounts;
	}
}
```
这意味着从返回值中过滤出任何不满足`filterObject.owner == authentication.name`的值。filterObject表示accounts中的每个帐户，用于测试每个帐户。
可以使用以下代码测试这个类:
```
@Autowired
BankService bankService;

@WithMockUser(username="owner")
@Test
void readAccountsWhenOwnedThenReturns() {
    Collection<Account> accounts = this.bankService.updateAccounts("owner", "not-owner");
    assertThat(accounts).hasSize(1);
    assertThat(accounts.get(0).getOwner()).isEqualTo("owner");
}
```
> @PostFilter也可以是元注解，在类或接口级别定义，并使用SpEL授权表达式。
@PostFilter支持数组、集合、映射和流(只要流是打开的)。
例如，上面的readAccounts声明将以与下面三个声明相同的方式起作用:
``` Java
@PostFilter("filterObject.owner == authentication.name")
public Account[] readAccounts(String... ids)

@PostFilter("filterObject.value.owner == authentication.name")
public Map<String, Account> readAccounts(String... ids)

@PostFilter("filterObject.owner == authentication.name")
public Stream<Account> readAccounts(String... ids)
```
### 使用@Secured授权方法调用
> `@Secured`是授权调用的遗留功能，已被@PreAuthorize，建议改为@PreAuthorize。

### 使用JSR-250注解授权方法调用
如果开发者想使用`JSR-250注解`，Spring Security也支持。@PreAuthorize表达能力更强，推荐使用。

要使用JSR-250注解，应该首先修改Method Security声明，使其生效，如下所示:
``` Java
@EnableMethodSecurity(jsr250Enabled = true)
```
这将导致Spring Security发布相应的方法拦截器，该拦截器授权带有`@RolesAllowed`、`@PermitAll`和`@DenyAll`注解的方法、类和接口。

### 在类或接口级别声明注解
它还支持在类和接口级别使用方法安全性注解。

如果是在类级别，如下所示:
``` Java
@Controller
@PreAuthorize("hasAuthority('ROLE_USER')")
public class MyController {
    @GetMapping("/endpoint")
    public String endpoint() { ... }
}
```
然后，所有方法继承类级行为。
或者，如果它在类和方法级别都声明如下:
``` Java
@Controller
@PreAuthorize("hasAuthority('ROLE_USER')")
public class MyController {
    @GetMapping("/endpoint")
    @PreAuthorize("hasAuthority('ROLE_ADMIN')")
    public String endpoint() { ... }
}
```
然后，**声明注解的方法重写类级注解**。
接口也是如此，不同的是，**如果一个类从两个不同的接口继承了注解，那么启动将失败**。这是因为Spring Security无法识别开发者想使用哪一个。
在这种情况下，可以通过向具体方法添加注解来解决歧义。

### 使用元注解
方法安全性支持元注解。这意味着开发者可以根据特定于应用程序的用例使用任何注解并提高可读性。

例如，可以简化`@PreAuthorize("hasRole('ADMIN')")`为`@IsAdmin`，如下所示:
``` Java
@Target({ ElementType.METHOD, ElementType.TYPE })
@Retention(RetentionPolicy.RUNTIME)
@PreAuthorize("hasRole('ADMIN')")
public @interface IsAdmin {}
```
结果是，在开发者的安全方法上，开发者现在可以执行以下操作:
``` Java
@Component
public class BankService {
	@IsAdmin
	public Account readAccount(Long id) {
        // ... is only returned if the `Account` belongs to the logged in user
	}
}
```
这使得方法定义更具可读性。

### 启用特定注解
关闭`@EnableMethodSecurity`的预配置，并替换为自定义的。
- 如果想自定义`AuthorizationManager`或`Pointcut`，可以选择这样做。
- 或者可能只想启用特定的注解，比如`@PostAuthorize`。
举例如下:
``` Java
@Configuration
@EnableMethodSecurity(prePostEnabled = false)
class MethodSecurityConfig {
	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	Advisor postAuthorize() {
		return AuthorizationManagerBeforeMethodInterceptor.postAuthorize();
	}
}
```
上面的代码片段通过首先禁用Method Security的预配置，然后生成`@PostAuthorize`拦截器来实现这一点。

## 使用<intercept-methods>进行授权
> 推荐使用Spring Security的基于注解的支持来实现方法安全性

## 以编程方式授权方法
基于java而不是基于SpEL实现复杂的授权规则有很多方法。这使得用户可以使用完整Java语言特性，从而**提高可测试性和流程控制**。

### 方法一: 在SpEL中使用自定义Bean
首先，像下面这样声明一个bean，它有一个接受`MethodSecurityExpressionOperations`实例的方法:
``` Java
@Component("authz")
public class AuthorizationLogic {
    public boolean decide(MethodSecurityExpressionOperations operations) {
        // ... authorization logic
    }
}
```
然后，在注解中如下引用自定义的Bean
``` Java
@Controller
public class MyController {
    @PreAuthorize("@authz.decide(#root)")
    @GetMapping("/endpoint")
    public String endpoint() {
        // ...
    }
}
```
Spring Security将为每个方法调用调用该bean上的给定方法。
这样做的好处是，所有授权逻辑都在一个单独的类中，可以独立地对其进行单元测试和正确性验证。它还可以使用完整Java语言特性。

# 方法二: 使用自定义授权管理器(Authorization Manager)
首先，声明一个授权管理器实例，比如:
``` Java
@Component
public class MyAuthorizationManager implements AuthorizationManager<MethodInvocation> {
    public AuthorizationDecision check(Supplier<Authentication> authentication, MethodInvocation invocation) {
        // ... authorization logic
    }
}
```
然后，生成带有切入点的方法拦截器，该切入点对应于开发者希望`AuthorizationManager`运行的时机。
例如，可以这样替换`@PreAuthorize`和`@PostAuthorize`的工作方式:
``` Java
@Configuration
@EnableMethodSecurity(prePostEnabled = false)
class MethodSecurityConfig {
    @Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	Advisor postAuthorize(MyAuthorizationManager manager) {
		return AuthorizationManagerBeforeMethodInterceptor.preAuthorize(manager);
	}

	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	Advisor postAuthorize(MyAuthorizationManager manager) {
		return AuthorizationManagerAfterMethodInterceptor.postAuthorize(manager);
	}
}
```
> 可以使用`AuthorizationInterceptorsOrder`中指定的Order常量将拦截器放在Spring Security方法拦截器之间。
### 方法三: 自定义表达式处理
自定义每个SpEL表达式的处理方式。
要做到这一点，可以公开一个自定义`MethodSecurityExpressionHandler`，如下所示:
``` Java
@Bean
static MethodSecurityExpressionHandler methodSecurityExpressionHandler(RoleHierarchy roleHierarchy) {
	DefaultMethodSecurityExpressionHandler handler = new DefaultMethodSecurityExpressionHandler();
	handler.setRoleHierarchy(roleHierarchy);
	return handler;
}
```
> 使用一个**静态方法**公开`MethodSecurityExpressionHandler`，以确保Spring在初始化Spring Security的Method Security的@Configuration类之前发布它

开发者还可以子类化`DefaultMessageSecurityExpressionHandler`，以在默认值之外添加自定义授权表达式。

## 使用AspectJ进行授权
### 用自定义切入点匹配方法
由于Method Security基于Spring AOP构建，因此可以声明与注解无关的模式，类似于请求级授权。这具有**集中方法级授权规则**的潜在优势。

例如，可以通过生成自己的Adviso来为服务层匹配AOP表达式和授权规则，如下所示:
``` Java
@Bean
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
static Advisor protectServicePointcut() {
    JdkRegexpMethodPointcut pattern = new JdkRegexpMethodPointcut();
    pattern.setPattern("execution(* com.mycompany.*Service.*(..))");
    return new AuthorizationManagerBeforeMethodInterceptor(pattern, hasRole("USER"));
}
```

### 与AspectJ Byte-weaving
**有时**可以通过使用AspectJ将Spring Security `advice`编织到bean的字节码中来**增强性能**。

在设置了AspectJ之后，在@EnableMethodSecurity注解中声明使用AspectJ:
``` Java
@EnableMethodSecurity(mode=AdviceMode.ASPECTJ)
```
结果是Spring Security将它的`advice`作为`AspectJ Advisor`发布，这样它们就可以被相应地编织进来。

## 指定Order
如前所述，每个注解都有一个Spring AOP方法拦截器，每个注解在Spring AOP Advisor链中都有固定的位置。
比如说，`@PreFilter`方法拦截器的Order是100，`@PreAuthorize`的Order是200，以此类推。

之所以重要，是因为还有其他基于AOP的注解，如`@EnableTransactionManagement`，其`order`为`Integer.MAX_VALUE`。换句话说，默认情况下，它们位于advisor链的末端。

有时，在Spring Security之前执行其他Advice可能很有价值。例如，如果开发者有一个带有`@Transactional`和`@PostAuthorize`注解的方法，开发者可能希望在`@PostAuthorize`运行时事务仍然是打开的，以便允许`AccessDeniedException`将触发回滚。

为了让`@EnableTransactionManagemen`t在方法授权建议运行之前打开一个事务，可以像这样设置`@EnableTransactionManagement`的Order:
``` Java
@EnableTransactionManagement(order = 0)
```
由于最早的方法拦截器(@PreFilter)被设置为100的Order，因此设置为0意味着事务通知将在所有Spring Security advice之前运行。


## 使用SpEL表达授权
Spring Security将其所有授权字段(Field)和方法(Method)封装在一组根对象中。最通用的根对象叫做`SecurityExpressionRoot`，它构成了`MethodSecurityExpressionRoot`的基础。Spring Security在准备计算授权表达式的值时将这个根对象传递给`MethodSecurityEvaluationContext`。

### 使用授权表达式*字段*和*方法*
它提供的第一件事是为SpEL表达式提供一组增强的授权字段和方法。以下是对最常见方法的快速概述:
- `permitAll`: 该方法不需要调用授权;注意，在这种情况下，永远不会从会话中检索Authentication
- `denyAll`: 在任何情况下都不允许使用该方法;注意，在这种情况下，永远不会从会话中检索Authentication
- `hasAuthority`: 该方法要求认证具有与给定值匹配的GrantedAuthority
- `hasRole`: hasAuthority的快捷方式，前缀为ROLE_或任何被配置为默认前缀的东西
- `hasAnyAuthority`: 该方法要求认证具有与任何给定值匹配的GrantedAuthority
- `hasAnyRole`: `hasAnyAuthority`的快捷方式，它以ROLE_或任何被配置为默认前缀的前缀为前缀
- `hasPermission`: 关联到`PermissionEvaluator`实例进行对象级授权
下面简要介绍一下最常见的领域:
- authentication: 与此方法调用关联的Authentication实例
- principal: 与此方法调用关联的`Authentication#getPrincipal`

举例如下：
``` Java
Component
public class MyService {
    // 任何人不得以任何理由调用此方法
    @PreAuthorize("denyAll") 
    MyResource myDeprecatedMethod(...);

    // 此方法只能由授予ROLE_ADMIN权限的认证调用
    @PreAuthorize("hasRole('ADMIN')") 
    MyResource writeResource(...)

    // 此方法只能由授予db和ROLE_ADMIN权限的身份验证调用
    @PreAuthorize("hasAuthority('db') and hasRole('ADMIN')") 
    MyResource deleteResource(...)

    // 此方法只能由aud声明等于“my-audience”的principal调用。
    @PreAuthorize("principal.claims['aud'] == 'my-audience'") 
    MyResource readResource(...);

    // 只有当bean authz的检查方法返回true时，才能调用此方法
	@PreAuthorize("@authz.check(authentication, #root)")
    MyResource shareResource(...);
}
```

## 使用方法参数
Spring Security提供了一种发现方法参数的机制，以便也可以在SpEL表达式中访问它们。

为了获得完整的引用，Spring Security使用`DefaultSecurityParameterNameDiscoverer`来发现参数名。默认情况下，将为方法尝试以下选项。

如果Spring Security的`@P`注解出现在方法的**单个**参数上，则使用该值。下面的例子使用了@P注解:
``` Java
@PreAuthorize("hasPermission(#c, 'write')")
public void updateContact(@P("c") Contact contact);
```
此表达式的目的是要求当前Authentication具有专门针对此Contact实例的写权限。

在幕后，这是通过使用`AnnotationParameterNameDiscoverer`实现的，开发者可以自定义它以支持任何指定注解的value属性。

- 如果Spring Data的@Param注解出现在该方法的至少一个参数上，则使用该值。下面的例子使用了@Param注解:
    ``` Java
    @PreAuthorize("#n == authentication.name")
    Contact findContactByName(@Param("n") String name);
    ```
    这个表达式的目的是要求name等于authentication#getName，以便授权调用。
- 如果使用`-parameters`参数编译代码，则使用标准JDK反射API来发现参数名称。这对类和接口都有效。
- 最后，如果使用debug符号编译代码，则通过使用debug符号发现参数名。这**不适用于接口**，因为它们没有关于参数名称的调试信息。对于接口，必须使用*注解*或`-parameters`方法。

# 域对象安全访问控制(ACL)
[参考](https://docs.spring.io/spring-security/reference/servlet/authorization/acls.html)