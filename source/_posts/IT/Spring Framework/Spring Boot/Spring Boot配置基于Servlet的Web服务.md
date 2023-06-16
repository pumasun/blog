---
title: Spring Boot配置基于Servlet的Web服务
tags:
- Spring Boot
- Spring MVC
categary:
- IT
- Spring Boot
---

# Spring MVC 自动配置(Auto-configuration)
Spring Boot为Spring MVC提供了自动配置，可以很好地与大多数应用程序配合使用。

自动配置在Spring默认设置的基础上增加了以下特性:
- 包含`ContentNegotiatingViewResolver`和`BeanNameViewResolver` bean。
- 支持提供静态资源，包括对`webjar`的支持。
- 自动注册`Converter`、`GenericConverter`和`Formatter` bean。
- 对`HttpMessageConverters`的支持。
- `MessageCodesResolver`的自动注册。
- 静态`index.html`支持。
- 自动使用`ConfigurableWebBindingInitializer` bean。

如果你想在保留那些Spring Boot MVC配置的基础上，做更多的MVC配置(拦截器、格式化器、视图控制器和其他功能)，你可以添加你自己的`@Configuration`类，类型为`WebMvcConfigurer`，但不添加`@EnableWebMvc`。

如果你想提供`RequestMappingHandlerMapping`、`RequestMappingHandlerAdapter`或`ExceptionHandlerExceptionResolver`的自定义实例，并且仍然保持Spring Boot MVC的配置，你可以声明一个`WebMvcRegistrations`类型的bean，并使用它来提供这些组件的自定义实例。

如果你想完全控制Spring MVC，你可以添加你自己的带有`@EnableWebMvc`注释的`@Configuration`，或者像在`@EnableWebMvc`的Javadoc中描述的那样添加你自己的带有`@Configuration`注释的`DelegatingWebMvcConfiguration`。

# HttpMessageConverters
Spring MVC使用`HttpMessageConverter`接口来转换HTTP请求和响应。
Spring Boot提供了一些开箱即用默认值。例如，对象可以自动转换为JSON(通过使用Jackson库)或XML(通过使用Jackson XML扩展，如果可用，或者通过使用JAXB，如果Jackson XML扩展不可用)。
缺省情况下，字符串采用`UTF-8`编码。

任何出现在上下文中的`HttpMessageConverter` bean都会被添加到转换器列表中。
如果你需要添加或自定义转换器，或者需要覆盖默认转换器，你可以使用Spring Boot的`HttpMessageConverters`类，如下面的代码所示:
``` Java
@Configuration(proxyBeanMethods = false)
public class MyHttpMessageConvertersConfiguration {

    @Bean
    public HttpMessageConverters customConverters() {
        HttpMessageConverter<?> additional = new AdditionalHttpMessageConverter();
        HttpMessageConverter<?> another = new AnotherHttpMessageConverter();
        return new HttpMessageConverters(additional, another);
    }

}
```

# MessageCodesResolver
一个生成错误代码的策略，用于拼装绑定错误(Binding errors)中错误消息。
如果你设置`spring.mvc.message-codes-resolver-format`为PREFIX_ERROR_CODE或POSTFIX_ERROR_CODE, 则Spring Boot会为你创建一个默认`MessageCodesResolver`(参见`DefaultMessageCodesResolver.Format`中的枚举)。

# 静态内容

## 静态内容目录位置
默认情况下，Spring Boot从指定目录提供静态内容。
- classpath中的`/static`
- classpath中的`/public`
- classpath中的`/resources`
- classpath中的`/META-INF/resources`
- ServletContext的根(/)目录
它使用来自Spring MVC的`ResourceHttpRequestHandler`，这样你就可以通过添加你自己的`WebMvcConfigiler`和重写`addResourceHandlers`方法来修改这个行为。
也可以使用`spring.web.resources.static-locations`属性(用目录位置列表替换默认值)自定义静态资源位置。servlet上下文的根路径(/)也会被自动添加为资源位置。

> 如果你的应用被打包成jar包，不要使用`src/main/webapp`目录。虽然这个目录是一个通用的标准，但它**只适用于war**打包，如果生成一个jar，大多数构建工具都会忽略它。

在独立(standalone)的web应用程序中，默认没有启用来自容器的默认servlet。可以使用`server.servlet.register-default-servlet`来启用。
如果Spring决定不处理它，则使用默认servlet作为后备处理，从`ServletContext`的根目录提供内容。大多数情况下，这种情况不会发生(除非您修改了默认的MVC配置)，因为Spring总是可以通过`DispatcherServlet`处理请求。

## 静态内容的映射
默认情况下，资源将被映射到`/**`上，但您可以使用`spring.mvc.static-path-pattern`对其进行调整。例如，将所有资源重定位到`/resources/**`可以这样实现:
``` YAML
spring:
  mvc:
    static-path-pattern: "/resources/**"
```

## Webjars
除了前面提到的“标准”静态资源位置之外，还有webjar内容的特殊情况。默认情况下，任何路径在`/webjars/\*\*`中的资源都是从jar文件中提供的，如果它们被打包成webjar格式的话。路径可以使用`spring.mvc.webjars-path-pattern`进行设置。

## 禁止缓存(Cache-busting)
Spring Boot还支持Spring MVC提供的高级资源处理特性，允许使用缓存破坏静态资源或为webjar使用版本无关的url等用例。

要为webjar使用与版本无关的url，请添加`webjars-locator-core`依赖项。然后声明你的Webjar。
以jQuery为例，添加`/webjars/jquery/jquery.min.js`会得到`/webjars/jquery/x.y.z/jquery.min.js`，其中`x.y.z`是Webjar的版本号。

要使用缓存破坏，以下配置为所有静态资源配置缓存破坏解决方案，有效地在url中添加内容哈希，例如`<link href="/css/spring-2a2d595e6ed9a0b24f027f2b63b134d6.css"/>`:
``` YAML
spring:
  web:
    resources:
      chain:
        strategy:
          content:
            enabled: true
            paths: "/**"
```
> 到资源的链接在运行时在模板中被重写，因为`thyymeleaf`和`FreeMarker`自动配置了`ResourceUrlEncodingFilter`。
> 在使用JSP时，应该手动声明这个过滤器。
> 其他模板引擎目前不自动支持，但可以通过自定义模板宏/助手和使用`ResourceUrlProvider`来支持。

当使用加载器动态加载资源时，例如JavaScript模块，不能重命名文件。因此Spring Boot支持了其他策略并可以联合使用。
`fixed`策略在URL中添加一个静态版本字符串，而不改变文件名，如下例所示:
``` YAML
spring:
  web:
    resources:
      chain:
        strategy:
          content:
            enabled: true
            paths: "/**"
          fixed:
            enabled: true
            paths: "/js/lib/"
            version: "v12"
```
通过这种配置，位于`/js/lib/`下的JavaScript模块使用固定的版本控制策略(`/v12/js/lib/mymodule.js`)，而其他资源仍然使用内容控制策略(`<link href="/css/spring-2a2d595e6ed9a0b24f027f2b63b134d6.css"/>`)。
更多支持选项参考`WebProperties.Resources`。


## 欢迎页面
Spring Boot支持静态和模板化的欢迎页面。
它首先在配置的静态内容位置中查找`index.html`文件。如果没有找到，则查找`index`模板。如果找到任何一个，它将自动用作应用程序的欢迎页面。

## 自定义Favicon
与其他静态资源一样，Spring Boot在配置的静态内容位置中检查`favicon.ico`。如果存在这样的文件，它将自动用作应用程序的图标。

# 路径匹配与路径协商

## 路径匹配
Spring MVC可以通过查看请求路径并将其与应用程序中定义的映射(例如，Controller方法上的`@GetMapping`注释)相匹配，将传入的HTTP请求映射到处理程序。

从Spring Framework `5.3`起，Spring MVC支持多种将请求路径匹配到Controller处理程序的实现策略。之前仅支持`AntPathMatcher`策略，现在也提供`PathPatternParser`支持。
在Spring Boot中，可以使用以下配置属性选择、配置新的策略：
``` YAML
spring:
  mvc:
    pathmatch:
      matching-strategy: "path-pattern-parser"
```
> `PathPatternParser` 是一种优化的实现，但限制了某些路径模式变体的使用。
> 它与`后缀模式匹配`或使用`servlet前缀`(spring.mvc.servlet.path)映射DispatcherServlet不兼容。

默认情况下，如果没有为请求找到处理程序，Spring MVC将发送404 Not Found错误响应。
要抛出`NoHandlerFoundException`，需要设置configprop:`spring.mvc.throw-exception-if-no-handler-found=true`。
> 请注意，默认情况下，静态内容的服务映射到`/**`，因此将为所有请求提供处理程序。要抛出`NoHandlerFoundException`，
> - 还必须设置`spring.mvc.static-path-pattern`设置为更具体的值，如`/resources/**`
> - 或`spring.web.resources.add-mappings=true`以完全禁用静态内容的服务

## 内容协商
Spring Boot默认 **`禁用`** `后缀模式匹配`，这意味着像`GET /projects/spring-boot.json`这样的请求。不会匹配到`@GetMapping("/projects/spring-boot")`映射。这被认为是Spring MVC应用程序的最佳实践。这个特性在过去主要用于HTTP客户端没有发送正确的`Accept`请求头;我们需要确保向客户端发送正确的内容类型(Content Type)。如今，内容协商更加可靠。

还有其他方法来处理HTTP客户端不一致地发送正确的`Accept`请求头。相比使用后缀匹配，我们可以使用查询参数来确保像`GET /projects/spring-boot?format=json`将被映射到`@GetMapping("/projects/spring-boot")`:
``` YAML
spring:
  mvc:
    contentnegotiation:
      favor-parameter: true
```
也可以用一下配置自定义参数名:
``` YAML
spring:
  mvc:
    contentnegotiation:
      favor-parameter: true
      parameter-name: "myparam"
```
大多数标准的媒体类型都支持开箱即用，但你仍然可以这样定义新的媒体类型:
``` YAML
spring:
  mvc:
    contentnegotiation:
      media-types:
        markdown: "text/markdown"
```

# ConfigurableWebBindingInitializer
Spring MVC使用一个`WebBindingInitializer`来为一个特定的请求初始化一个`WebDataBinder`。如果你用注解@Bean创建了自己的`ConfigurableWebBindingInitializer`, 则Spring Boot将自动配置Spring MVC使用它。


# 页面模板引擎
除了REST web服务，您还可以使用Spring MVC来提供动态HTML内容。Spring MVC支持各种模板技术，包括Thymeleaf、FreeMarker和JSP。此外，许多其他模板引擎也包含了它们自己的Spring MVC集成。

Spring Boot包括对以下模板引擎的自动配置支持:
- FreeMarker
- Groovy
- Thymeleaf
- Mustache
  
当你以默认配置使用这些模板引擎之一时，你的模板会自动从`src/main/resources/templates`中获取。

>当运行使用嵌入式servlet容器(并打包为可执行归档文件)的Spring Boot应用程序时，JSP支持存在一些限制，应尽量避免使用JSP:
> - 对于Jetty和Tomcat，如果使用WAR打包，它应该可以工作。当使用java -jar启动时，可执行WAR将工作，并且还可以部署到任何标准容器中。使用可执行jar时不支持JSP。
> - Undertow不支持JSP。
> - 创建自定义error.jsp页面不会覆盖错误处理的默认视图。应该使用自定义错误页面。

# 异常处理
默认情况下，Spring Boot提供了一个`/error`映射，以一种简单的方式处理所有错误，并在servlet容器中将其注册为“全局”错误页面。
- 对于机器客户机，它生成一个JSON响应，其中包含错误、HTTP状态和异常消息的详细信息。
- 对于浏览器客户端，有一个以HTML格式呈现相同数据的简易错误视图，可以通过添加一个解析error的视图(View)。

如果要自定义默认错误处理行为，可以设置`server.error`属性。
要完全替换默认行为，您可以实现`ErrorController`并注册该类型的Bean定义，或者添加`ErrorAttributes`类型的bean以使用现有机制但替换内容。

你也可以使用常规的Spring MVC特性，比如`@ExceptionHandler`方法和`@ControllerAdvice`。`ErrorController`会捕捉任何未被处理的异常。

> 可以继承BasicErrorController以实现自定义的ErrorController。
> 特别适用于为新内容类型(Content Type)添加处理程序(默认是专门处理`text/html`，并为其他所有内容提供后备处理)。
> 要实现自定义`ErrorController`，扩展`BasicErrorController`，添加一个带有`@RequestMapping`的公共方法(该方法有一个`produces`属性)，并创建一个此类型的Bean。

你也可以定义一个带`@ControllerAdvice`注解的类，让它为特定的控制器，或者异常类型返回自定义JSON内容，，如下例所示:
``` Java
@ControllerAdvice(basePackageClasses = SomeController.class)
public class MyControllerAdvice extends ResponseEntityExceptionHandler {

    @ResponseBody
    @ExceptionHandler(MyException.class)
    public ResponseEntity<?> handleControllerException(HttpServletRequest request, Throwable ex) {
        HttpStatus status = getStatus(request);
        return new ResponseEntity<>(new MyErrorBody(status.value(), ex.getMessage()), status);
    }

    private HttpStatus getStatus(HttpServletRequest request) {
        Integer code = (Integer) request.getAttribute(RequestDispatcher.ERROR_STATUS_CODE);
        HttpStatus status = HttpStatus.resolve(code);
        return (status != null) ? status : HttpStatus.INTERNAL_SERVER_ERROR;
    }

}
```


## 自定义异常页面
如果希望为给定状态码显示自定义HTML错误页面，可以将文件添加到`/error`目录。错误页面可以是静态HTML(即添加在任何静态资源目录下)，也可以是使用模板构建的。文件的名称应该是确切的状态码或状态码的掩码。
例如，要将404映射到静态HTML文件，你的目录结构将如下所示:
```
src/
 +- main/
     +- java/
     |   + <source code>
     +- resources/
         +- public/
             +- error/
             |   +- 404.html
             +- <other public assets>
```
使用FreeMarker模板映射所有5xx错误，你的目录结构如下:
```
src/
 +- main/
     +- java/
     |   + <source code>
     +- resources/
         +- templates/
             +- error/
             |   +- 5xx.ftlh
             +- <other templates>
```
对于更复杂的映射，您还可以添加实现`ErrorViewResolver`接口的bean，如下面的示例所示:
``` Java
public class MyErrorViewResolver implements ErrorViewResolver {

    @Override
    public ModelAndView resolveErrorView(HttpServletRequest request, HttpStatus status, Map<String, Object> model) {
        // Use the request or status to optionally return a ModelAndView
        if (status == HttpStatus.INSUFFICIENT_STORAGE) {
            // We could add custom model values here
            new ModelAndView("myview");
        }
        return null;
    }

}
```


# CORS支持
跨域资源共享(Cross-origin resource sharing, CORS)是由大多数浏览器实现的W3C规范，它允许您以灵活的方式指定授权哪种跨域请求，而不是使用一些不安全或者功能单一的方法，如`IFRAME`或`JSONP`。

从4.2版本开始，Spring MVC支持CORS。在Spring Boot应用程序中使用带有`@CrossOrigin`注释的控制器方法CORS配置不需要任何特定的配置。全局CORS配置可以通过使用自定义的`addCorsMappings(CorsRegistry)`方法注册`WebMvcConfigurer` bean来定义，如下例所示:
``` Java
@Configuration(proxyBeanMethods = false)
public class MyCorsConfiguration {

    @Bean
    public WebMvcConfigurer corsConfigurer() {
        return new WebMvcConfigurer() {

            @Override
            public void addCorsMappings(CorsRegistry registry) {
                registry.addMapping("/api/**");
            }

        };
    }

}
```