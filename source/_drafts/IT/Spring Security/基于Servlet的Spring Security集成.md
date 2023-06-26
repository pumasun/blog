---
title: 基于Servlet的Spring Security集成
tags:
---

# 什么情况下使用Spring Security
1. 构建REST API
   1. 使用JWT认证
   2. 或者其他bear token认证。
2. 构建Web应用、API Gateway或者BFF(Backend-for-Frontend)
   1. 使用OAuth 2.0 或 OIDC
   2. 使用SAML 2.0
   3. 使用CAS
3. 需要管理用户/密码
   1. 基于LDAP
   2. 基于Active Directory
   3. 基于Spring Data
   4. 基于JDBC
4. 其他情况，需要考虑的事项
   1. 协议，支持基于Servlet的HTTPS， HTTP与Web-Socket
   2. 认证，用户如何进行认证，是由状态的还是无状态的。
   3. 授权，如何决定授权用户可以执行哪些操作。
   4. 防护，除了集成Spring Security的默认防护以外，还需要什么防护功能？

# 认证机制
- 用户名和密码-如何使用用户名/密码进行身份验证
- OAuth 2.0登录- OAuth 2.0登录与OpenID连接和非标准OAuth 2.0登录(即GitHub)
- SAML 2.0登录- SAML 2.0登录
- CAS (Central Authentication Server) -支持CAS (Central Authentication Server)
- 记住我-如何记住一个用户过去的会话到期
- JAAS身份验证—使用JAAS进行身份验证
- 预认证场景——使用外部机制(如SiteMinder或Java EE安全)进行身份验证，但仍然使用Spring security进行授权和防止常见漏洞利用。
- X509身份验证—X509身份验证

# 用户名和密码认证
验证用户名和密码是最通用的认证方法。

## 表单登录(Form)
用户是如何被重定向到登录表单的:
![重定向到登录表单](/img/IT/SpringSecurity/基于Servlet的Spring-Security集成-1.png)
上图建立在`SecurityFilterChain`图的基础上。
 1. 用户向未经授权的资源(/private)发出未经身份验证的请求。
 2. Spring Security的`AuthorizationFilter`通过抛出`AccessDeniedException`拒绝未经身份验证的请求。
 3. 由于用户没有经过身份验证，`ExceptionTranslationFilter`启动Start Authentication，并使用配置的`AuthenticationEntryPoint`发送重定向到登录页面。在大多数情况下，`AuthenticationEntryPoint`是一个`LoginUrlAuthenticationEntryPoint`的实例。
 4. 浏览器请求重定向到的登录页面。
 5. 应用程序中的某些内容，要求必须呈现登录页面。

当提交用户名和密码时，`UsernamePasswordAuthenticationFilter`对用户名和密码进行认证。`UsernamePasswordAuthenticationFilter`扩展了`AbstractAuthenticationProcessingFilter`，所以下面的图看起来应该非常相似:
![用户名和密码认证](img/IT/SpringSecurity/基于Servlet的Spring-Security集成-2.png)
该图建立在`SecurityFilterChain`图的基础上。
1. 当用户提交他们的用户名和密码时，`UsernamePasswordAuthenticationFilter`通过从`HttpServletRequest`实例中提取用户名和密码来创建`UsernamePasswordAuthenticationToken`，这是一种身份验证。
2. 接下来，`UsernamePasswordAuthenticationToken`被传递到要进行身份验证的`AuthenticationManager`实例中。`AuthenticationManager`的细节取决于用户信息的存储方式。
3. 如果身份验证失败，则返回Failure。
   1. 清除`SecurityContextHolder`。
   2. 调用`RememberMeServices.loginFail`。如果Remember-Me没有配置，无操作。
   3. 调用`AuthenticationFailureHandler`。
4. 如果身份验证成功，则Success。
   1. `SessionAuthenticationStrategy`收到新登录的通知。
   2. 在`SecurityContexHolder`上设置`Authentication`。
   3. 调用`RememberMeServices.loginSuccess`。如果Remember-Me没有配置，无操作。
   4. `ApplicationEventPublisher`发布一个`InteractiveAuthenticationSuccessEvent`事件。
   5. 调用`AuthenticationSuccessHandler`。通常，这是一个`SimpleUrlAuthenticationSuccessHandler`，当我们重定向到登录页面时，它会重定向到由`ExceptionTranslationFilter`保存的请求。

默认情况下，Spring Security表单登录是启用的。然而，一旦提供了任何基于servlet的配置，就必须显式地提供基于表单的登录。下面的例子展示了一个最小的、显式的Java配置:
``` Java
public SecurityFilterChain filterChain(HttpSecurity http) {
	http
		.formLogin(withDefaults());
	// ...
}
```
当在Spring Security配置中指定登录页面时，开发人员负责呈现该页面。
``` HTML
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml" xmlns:th="https://www.thymeleaf.org">
	<head>
		<title>Please Log In</title>
	</head>
	<body>
		<h1>Please Log In</h1>
		<div th:if="${param.error}">
			Invalid username and password.</div>
		<div th:if="${param.logout}">
			You have been logged out.</div>
		<form th:action="@{/login}" method="post">
			<div>
			<input type="text" name="username" placeholder="Username"/>
			</div>
			<div>
			<input type="password" name="password" placeholder="Password"/>
			</div>
			<input type="submit" value="Log in" />
		</form>
	</body>
</html>
```
其中默认HTML表单有几个要求:
- 表单应该执行`post`到`/login`。
- 表单需要包含一个`CSRF`令牌，它由Thymeleaf自动包含。
- 表单应该在名为`username`的参数中指定用户名。
- 表单应该在名为`password`的参数中指定密码。
- 如果找到名为`error`的HTTP参数，则表明用户未能提供有效的用户名或密码。
- 如果找到名为`logout`的HTTP参数，则表明用户退出成功。
- 许多用户只需要定制登录页面。但是，如果需要，可以使用其他配置自定义前面显示的所有内容。
- 如果使用Spring MVC，则需要一个将`GET /login`映射到我们创建的登录模板的控制器。下面的例子展示了一个最小的LoginController:
``` JAVA
@Controller
class LoginController {
	@GetMapping("/login")
	String login() {
		return "login";
	}
}
```

## Basic认证
`WWW-Authenticate`报头被发送回未认证的客户端:
![发送WWW-Authenticate报头](/img/IT/SpringSecurity/基于Servlet的Spring-Security集成-3.png)
上图建立在`SecurityFilterChain`图的基础上。
1. 用户向未经授权的资源/私有发出未经身份验证的请求。
2. Spring Security的`AuthorizationFilter`表示通过抛出`AccessDeniedException`拒绝未经身份验证的请求。
3. 由于用户没有经过身份验证，`ExceptionTranslationFilter`启动Start Authentication。配置的`AuthenticationEntryPoint`是`BasicAuthenticationEntryPoint`的一个实例，它发送一个`WWW-Authenticate`头。`RequestCache`通常是一个不保存请求的`NullRequestCache`，因为客户端能够重播它最初请求的请求。
3. 当客户端接收到`WWW-Authenticate`报头时，它知道应该使用用户名和密码重试。用户名和密码的处理流程如下图所示:
![用户名和密码认证](/img/IT/SpringSecurity/基于Servlet的Spring-Security集成-4.png)
上图建立在SecurityFilterChain图的基础上。
1. 当用户提交他们的用户名和密码时，BasicAuthenticationFilter创建一个UsernamePasswordAuthenticationToken，这是一种通过从HttpServletRequest提取用户名和密码的身份验证类型。
2. 接下来，UsernamePasswordAuthenticationToken被传递到AuthenticationManager中进行身份验证。AuthenticationManager的细节取决于用户信息的存储方式。
3. 如果身份验证失败，则返回Failure。
   1. 清除SecurityContextHolder。
   2. 调用`RememberMeServices.loginFail`。如果记得我没有配置，无操作。
   3. 调用AuthenticationEntryPoint触发WWW-Authenticate再次发送。
4. 如果身份验证成功，则Success。
   1. 身份验证设置在`SecuritycContexHolder`上。
   2. 调用`RememberMeServices.loginSuccess`。如果记得我没有配置，无操作。
   3. `BasicAuthenticationFilter`调用`FilterChain.doFilter(Request, Response)`来继续处理应用程序逻辑的其余部分。
   4. 默认情况下，Spring Security的HTTP基本身份验证支持是启用的。然而，一旦提供了任何基于servlet的配置，就必须显式地提供HTTP Basic。

下面的例子展示了一个最小的、显式的配置:
``` Java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) {
	http
		// ...
		.httpBasic(withDefaults());
	return http.build();
}
```

## Digest
> 在现代应用程序中不应该使用摘要身份验证，因为它被认为是不安全的。
> 最明显的问题是，您必须以明文或加密或MD5格式存储密码。所有这些存储格式都是不安全的。
> 相反，您应该使用单向自适应密码散列(bCrypt、PBKDF2、SCrypt等)来存储凭证，摘要身份验证不支持这种方式。

摘要身份验证试图解决基本身份验证的许多弱点，特别是通过确保凭证永远不会以明文形式通过网络发送。许多浏览器都支持摘要身份验证。
管理HTTP摘要身份验证的标准由RFC 2617定义，它更新了RFC 2069规定的摘要身份验证标准的早期版本。大多数用户代理实现RFC 2617。Spring Security的摘要身份验证支持与RFC 2617规定的“auth”保护质量(qop)兼容，它还提供了与RFC 2069的向后兼容性。如果您需要使用未加密的HTTP(没有TLS或HTTPS)并希望最大限度地提高身份验证过程的安全性，那么摘要身份验证被视为更有吸引力的选择。
但是，每个人都应该使用HTTPS。

摘要身份验证的核心是`nonce`。这是服务器生成的值。Spring Security的`nonce`采用以下格式:
``` TXT
base64(expirationTime + ":" + md5Hex(expirationTime + ":" + key))
expirationTime:   The date and time when the nonce expires, expressed in milliseconds
key:              A private key to prevent modification of the nonce token
```
请确保使用NoOpPasswordEncoder配置了不安全的明文密码存储。下面是使用Java Configuration配置摘要认证的示例:
``` Java
@Autowired
UserDetailsService userDetailsService;

DigestAuthenticationEntryPoint entryPoint() {
	DigestAuthenticationEntryPoint result = new DigestAuthenticationEntryPoint();
	result.setRealmName("My App Realm");
	result.setKey("3028472b-da34-4501-bfd8-a355c42bdf92");
}

DigestAuthenticationFilter digestAuthenticationFilter() {
	DigestAuthenticationFilter result = new DigestAuthenticationFilter();
	result.setUserDetailsService(userDetailsService);
	result.setAuthenticationEntryPoint(entryPoint());
}

@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
	http
		// ...
		.exceptionHandling(e -> e.authenticationEntryPoint(authenticationEntryPoint()))
		.addFilterBefore(digestFilter());
	return http.build();
}
```

# 密码存储机制
每个支持的读取用户名和密码的机制都可以使用任何支持的存储机制:
- 基于内存的身份验证的简单存储
- 使用JDBC身份验证的关系数据库
- 使用UserDetailsService自定义数据存储
- LDAP存储和LDAP认证

## 基于内存的认证
Spring Security的`InMemoryUserDetailsManager`实现了`UserDetailsService`，为存储在内存中的基于用户名/密码的身份验证提供支持。`InMemoryUserDetailsManager`通过实现`UserDetailsManager`接口提供对`UserDetails`的管理。当Spring Security配置为接受用户名和密码进行身份验证时，将使用基于`UserDetails`的身份验证。

在下面的示例中，我们使用Spring Boot CLI对密码值password进行编码，得到编码后的密码为`{bcrypt}$2a$10$GRLdNijSQMUvl/au9ofL.eDwmoohzzS7.rmNSJZ.0FxO/BTk76klW`:
```Java
@Bean
public UserDetailsService users() {
	UserDetails user = User.builder()
		.username("user")
		.password("{bcrypt}$2a$10$GRLdNijSQMUvl/au9ofL.eDwmoohzzS7.rmNSJZ.0FxO/BTk76klW")
		.roles("USER")
		.build();
	UserDetails admin = User.builder()
		.username("admin")
		.password("{bcrypt}$2a$10$GRLdNijSQMUvl/au9ofL.eDwmoohzzS7.rmNSJZ.0FxO/BTk76klW")
		.roles("USER", "ADMIN")
		.build();
	return new InMemoryUserDetailsManager(user, admin);
}
```
上面的示例以安全的格式存储密码，但在起始开发体验方面还有很多需要改进的地方。

在下面的示例中，我们使用`User.withDefaultPasswordEncoder`，以确保存储在内存中的密码受到保护。
> 不能防止通过反编译源代码获得密码。因此，`User.withDefaultPasswordEncoder`应该只用于“起始开发”，而不是用于生产。

``` Java
@Bean
public UserDetailsService users() {
	// The builder will ensure the passwords are encoded before saving in memory
	UserBuilder users = User.withDefaultPasswordEncoder();
	UserDetails user = users
		.username("user")
		.password("password")
		.roles("USER")
		.build();
	UserDetails admin = users
		.username("admin")
		.password("password")
		.roles("USER", "ADMIN")
		.build();
	return new InMemoryUserDetailsManager(user, admin);
}
```
基于xml的配置，没有简单的`User.withDefaultPasswordEncoder`使用方法。对于演示或刚开始，你可以选择在密码前加上`{noop}`来表示不应该使用编码:
``` XML
<user-service>
	<user name="user"
		password="{noop}password"
		authorities="ROLE_USER" />
	<user name="admin"
		password="{noop}password"
		authorities="ROLE_USER,ROLE_ADMIN" />
</user-service>
```

## 基于JDBC的身份验证
Spring Security的`JdbcDaoImpl`实现`UserDetailsService`，为使用JDBC检索的基于用户名和密码的身份验证提供支持。`JdbcUserDetailsManager`扩展`JdbcDaoImpl`，通过`UserDetailsManager`接口提供对`UserDetails`的管理。当Spring Security配置为接受用户名/密码进行身份验证时，将使用基于`UserDetails`的身份验证。

### 默认Schema
用于默认查询的相应默认Schema。您需要调整Schema以匹配查询和您使用的数据库方言的任何定制。
- User Schema
    JdbcDaoImpl需要表加载用户的密码、帐户状态(启用或禁用)和权限(角色)列表。
    > 默认的schema被暴露为以下名字的classpath资源 `org/springframework/security/core/userdetails/jdbc/users.ddl`.
    ``` SQL
    create table users(
        username varchar_ignorecase(50) not null primary key,
        password varchar_ignorecase(500) not null,
        enabled boolean not null
    );

    create table authorities (
        username varchar_ignorecase(50) not null,
        authority varchar_ignorecase(50) not null,
        constraint fk_authorities_users foreign key(username) references users(username)
    );
    create unique index ix_auth_username on authorities (username,authority);
    ```
    Oracle是一种流行的数据库选择，但需要稍微不同的Schema:
    ``` SQL
    CREATE TABLE USERS (
        USERNAME NVARCHAR2(128) PRIMARY KEY,
        PASSWORD NVARCHAR2(128) NOT NULL,
        ENABLED CHAR(1) CHECK (ENABLED IN ('Y','N') ) NOT NULL
    );


    CREATE TABLE AUTHORITIES (
        USERNAME NVARCHAR2(128) NOT NULL,
        AUTHORITY NVARCHAR2(128) NOT NULL
    );
    ALTER TABLE AUTHORITIES ADD CONSTRAINT AUTHORITIES_UNIQUE UNIQUE (USERNAME, AUTHORITY);
    ALTER TABLE AUTHORITIES ADD CONSTRAINT AUTHORITIES_FK1 FOREIGN KEY (USERNAME) REFERENCES USERS (USERNAME) ENABLE;
    ```
- Group Schema
    如果你的应用程序使用组，你需要提供组Schema:
    ``` SQL
    create table groups (
        id bigint generated by default as identity(start with 0) primary key,
        group_name varchar_ignorecase(50) not null
    );

    create table group_authorities (
        group_id bigint not null,
        authority varchar(50) not null,
        constraint fk_group_authorities_group foreign key(group_id) references groups(id)
    );

    create table group_members (
        id bigint generated by default as identity(start with 0) primary key,
        username varchar(50) not null,
        group_id bigint not null,
        constraint fk_group_members_group foreign key(group_id) references groups(id)
    );
    ```
### 设置数据源
在配置`JdbcUserDetailsManager`之前，我们必须创建一个数据源。在我们的示例中，我们设置了一个用默认用户模式初始化的嵌入式数据源。
``` Java
@Bean
DataSource dataSource() {
	return new EmbeddedDatabaseBuilder()
		.setType(H2)
		.addScript(JdbcDaoImpl.DEFAULT_USER_SCHEMA_DDL_LOCATION)
		.build();
}
在生产环境中，您希望确保设置了到外部数据库的连接。

```
### `JdbcUserDetailsManager`Bean
在这个示例中，我们使用Spring Boot CLI对password的密码值进行编码，并获得编码后的密码为`{bcrypt}$2a$10$GRLdNijSQMUvl/au9ofL.eDwmoohzzS7.rmNSJZ.0FxO/BTk76klW`。
``` Java
@Bean
UserDetailsManager users(DataSource dataSource) {
	UserDetails user = User.builder()
		.username("user")
		.password("{bcrypt}$2a$10$GRLdNijSQMUvl/au9ofL.eDwmoohzzS7.rmNSJZ.0FxO/BTk76klW")
		.roles("USER")
		.build();
	UserDetails admin = User.builder()
		.username("admin")
		.password("{bcrypt}$2a$10$GRLdNijSQMUvl/au9ofL.eDwmoohzzS7.rmNSJZ.0FxO/BTk76klW")
		.roles("USER", "ADMIN")
		.build();
	JdbcUserDetailsManager users = new JdbcUserDetailsManager(dataSource);
	users.createUser(user);
	users.createUser(admin);
	return users;
}
```

## UserDetails
`UserDetails`由`UserDetailsService`返回。`DaoAuthenticationProvider`验证`UserDetails`，然后返回一个`Authentication`，该`Authentication`的主体(principal)是配置的`UserDetailsService`返回的`UserDetails`。

## UserDetailsService
`DaoAuthenticationProvider`使用`UserDetailsService`来检索用户名、密码和其他属性，以便使用用户名和密码进行身份验证。Spring Security提供了`UserDetailsService`的内存和JDBC实现。

您可以通过将自定义`UserDetailsService`公开为bean来定义自定义身份验证。例如，下面的清单定制了身份验证，假设`CustomUserDetailsService`实现了`UserDetailsService`:

> 只有在没有填充`AuthenticationManagerBuilder`并且没有定义`AuthenticationProviderBean`的情况下才会使用此方法。

``` Java
@Bean
CustomUserDetailsService customUserDetailsService() {
	return new CustomUserDetailsService();
}
```

## PasswordEncoder
Spring Security的servlet支持包括通过与`PasswordEncoder`集成安全地存储密码。您可以通过公开`PasswordEncoder` Bean来定制Spring Security使用的`PasswordEncoder`实现。

## DaoAuthenticationProvider
`DaoAuthenticationProvider`是一个`AuthenticationProvider`实现，它使用`UserDetailsService`和`PasswordEncoder`来验证用户名和密码。

`AuthenticationManager`的工作原理:
![DaoAuthenticationProvider用法](img/IT/SpringSecurity/基于Servlet的Spring-Security集成-5.png)
1. 身份验证过滤器将`UsernamePasswordAuthenticationToken`传递给`AuthenticationManager`, `AuthenticationManager`由`ProviderManager`实现。
2. `ProviderManager`被配置为使用`DaoAuthenticationProvider`类型的`AuthenticationProvider`。
3. `DaoAuthenticationProvider`从`UserDetailsService`中查找`UserDetails`。
4. `DaoAuthenticationProvider`使用`PasswordEncoder`在上一步返回的`UserDetails`上验证密码。
5. 当身份验证成功时，返回的`Authentication`类型为`UsernamePasswordAuthenticationToken`，其主体(pricipal)是配置的`UserDetailsService`返回的`UserDetails`。最终，返回的`UsernamePasswordAuthenticationToken`由身份验证过滤器设置到`SecurityContextHolder`中。

## 基于LDAP的认证
LDAP(轻量级目录访问协议)经常被组织用作用户信息的中央存储库和身份验证服务。它还可以用于存储应用程序用户的角色信息。

Spring Security在配置为接受用户名/密码进行身份验证时使用基于LDAP的身份验证。然而，尽管使用用户名和密码进行身份验证，但它不使用UserDetailsService，因为在绑定身份验证中，LDAP服务器不返回密码，因此应用程序无法执行密码验证。

对于如何配置LDAP服务器，有许多不同的场景，因此Spring Security的LDAP提供程序是完全可配置的。它使用单独的策略接口进行身份验证和角色检索，并提供默认实现，可以对其进行配置以处理各种情况。

### 先决条件
在尝试将LDAP与Spring Security一起使用之前，您应该熟悉它。下面的链接很好地介绍了所涉及的概念，并提供了使用免费LDAP服务器OpenLDAP: www.zytrax.com/books/ldap/设置目录的指南。熟悉用于从Java访问LDAP的JNDI API也很有用。我们没有在LDAP提供程序中使用任何第三方LDAP库(Mozilla、JLDAP或其他)，但是Spring LDAP得到了广泛的使用，因此如果您计划添加自己的定制，那么熟悉该项目可能会很有用。

> 在使用LDAP身份验证时，您应该确保正确地配置了LDAP连接池。

### 设置嵌入式LDAP服务器
您需要做的第一件事是确保您有一个LDAP服务器来指向您的配置。为了简单起见，通常最好从嵌入式LDAP服务器开始。Spring Security支持使用:
- 嵌入式UnboundID服务器
- 嵌入式ApacheDS服务器

在下面的示例中，我们公开用户。ldif作为类路径资源，用两个用户user和admin初始化嵌入式LDAP服务器，这两个用户的密码都是password:
users.ldif
``` LDIF
dn: ou=groups,dc=springframework,dc=org
objectclass: top
objectclass: organizationalUnit
ou: groups

dn: ou=people,dc=springframework,dc=org
objectclass: top
objectclass: organizationalUnit
ou: people

dn: uid=admin,ou=people,dc=springframework,dc=org
objectclass: top
objectclass: person
objectclass: organizationalPerson
objectclass: inetOrgPerson
cn: Rod Johnson
sn: Johnson
uid: admin
userPassword: password

dn: uid=user,ou=people,dc=springframework,dc=org
objectclass: top
objectclass: person
objectclass: organizationalPerson
objectclass: inetOrgPerson
cn: Dianne Emu
sn: Emu
uid: user
userPassword: password

dn: cn=user,ou=groups,dc=springframework,dc=org
objectclass: top
objectclass: groupOfNames
cn: user
uniqueMember: uid=admin,ou=people,dc=springframework,dc=org
uniqueMember: uid=user,ou=people,dc=springframework,dc=org

dn: cn=admin,ou=groups,dc=springframework,dc=org
objectclass: top
objectclass: groupOfNames
cn: admin
uniqueMember: uid=admin,ou=people,dc=springframework,dc=org
```