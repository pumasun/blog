---
title: Spring Boot简介

tags:
- Spring Boot
categary:
- IT
- Spring Boot

---
# 简介
Spring Boot帮助您创建可以运行的独立的、基于Spring的生产级应用程序。
我们对Spring平台和第三方库采用固定默认的配置，因此您可以轻松上阵。大多数Spring Boot应用程序只需要很少的Spring配置。

您可以使用Spring Boot创建Java应用程序，这些应用程序可以通过使用`java -jar`或更传统的战争部署来启动。

我们的主要目标是:
- 为所有Spring开发提供一个快速且广泛可访问的入门体验。
- 开箱即用，但当需求开始偏离默认值时，要迅速让开。
- 提供一系列大型项目中常见的非功能性特性(例如嵌入式服务器、安全性、度量、运行状况检查和外部化配置)。
- 绝对没有代码生成(当不针对本机映像时)，也不需要XML配置。

# 启动器(Starters)
启动器是一组方便的依赖描述符，可以包含在应用程序中。您可以一站式地获得所需的所有Spring和相关技术，而不必遍历示例代码并复制粘贴依赖描述符。例如，如果您想开始使用Spring和JPA进行数据库访问，请在您的项目中包含spring-boot-starter-data-jpa依赖项。

启动器包含大量依赖项，您可以使用这些依赖项来快速启动和运行项目，并使用一致的、受支持的托管传递依赖项集。

# 代码结构
虽然Spring Boot不要求固定的代码结构，但有一些很有用的最佳实践建议：

## 不要使用`default`包
如果一个类不指定包，则认为这个类的包为`default`。在Spring Boot应用中使用`default`包可能会导致一些特殊的问题，因为使用`@ComponentScan`, `@ConfigurationPropertiesScan`, `@EntityScan`, 或者`@SpringBootApplication`注解会读取Jar中的每个类。

## Main Application类的位置
建议将main application放在项目包的根目录。
`@SpringBootApplication`注释通常放在Main类上，它隐式地为某些项定义了一个基本的“搜索包”。
例如，如果您正在编写JPA应用程序，则使用`@SpringBootApplication`注释类的包来搜索`@Entity`项。
使用根包还允许组件扫描仅应用于您的项目。
以下列表展示了一个典型的目录结构:
```
com
 +- example
     +- myapplication
         +- MyApplication.java
         |
         +- customer
         |   +- Customer.java
         |   +- CustomerController.java
         |   +- CustomerService.java
         |   +- CustomerRepository.java
         |
         +- order
             +- Order.java
             +- OrderController.java
             +- OrderService.java
             +- OrderRepository.java
```
java文件将声明`main`方法，以及基本的`@SpringBootApplication`，如下所示:
``` java
@SpringBootApplication
public class MyApplication {

    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }

}
```

# 配置(Configuration)类
Spring Boot支持且偏向于基于java的配置。虽然可以将SpringApplication与XML源一起使用，但我们通常建议您的主要配置源是单个@Configuration类。通常，定义main方法的类很适合作为主@Configuration。

## 导入其他配置类
您不需要将所有的`@Configuration`放入单个类中。
`@Import`注释可用于导入其他配置类。或者，您可以使用`@ComponentScan`来自动加载所有Spring组件，包括`@Configuration`类。

## 导入XML配置
如果您绝对必须使用基于XML的配置，我们建议您仍然从`@Configuration`类开始。然后可以使用`@ImportResource`注释来加载XML配置文件。

# 自动配置
Spring Boot自动配置尝试根据您添加的jar依赖项自动配置Spring应用程序。
例如，如果HSQLDB在您的类路径中，并且您没有手动配置任何数据库连接bean，那么Spring Boot将自动配置一个内存数据库。

你需要通过在你的`@Configuration`类中添加`@EnableAutoConfiguration`或`@SpringBootApplication`注解来选择自动配置。

> 你应该只添加一个`@SpringBootApplication`或`@EnableAutoConfiguration`注释。我们通常建议您只将其中一个添加到主`@Configuration`类中。

## 逐步取代自动配置
自动配置是非侵入性的。在任何时候，您都可以开始定义自己的配置来替换自动配置的特定部分。例如，如果添加自己的`DataSource` bean，默认的嵌入式数据库支持就会被删除。

如果您需要找出当前正在应用的自动配置以及使用的原因，请使用`--debug`开关启动应用程序。这样做可以为选定的核心记录器启用调试日志，并将条件报告记录到控制台。

## 禁用特定的自动配置类
- 如果你发现你不想要的特定的自动配置类正在被应用，你可以使用`@SpringBootApplication`的`exclude`属性来禁用它们，如下面的例子所示:
``` Java
@SpringBootApplication(exclude = { DataSourceAutoConfiguration.class })
public class MyApplication {

}
```
- 如果类不在类路径上，则可以使用注释的excludeName属性并指定完全限定名。
- 如果你更喜欢使用`@EnableAutoConfiguration`而不是`@SpringBootApplication`，也可以使用`exclude`和`excludeName`。
- 您还可以使用`spring.autoconfigure.exclude`属性来控制要排除的自动配置类列表。
> 可以在注释(annotation)级别定义排除，也可以使用属性定义排除。

> 尽管自动配置类是公共的，但唯一被认为是公共API的识别是类名，它可以用于禁用自动配置。这些类的实际内容，如嵌套配置类或bean方法，仅供内部使用，我们不建议直接使用它们。

# Spring Beans和依赖注入(Dependency Injection)
虽然您可以自由地使用任何标准的Spring框架技术来定义bean及其注入的依赖项，但我们通常建议使用构造函数注入来连接依赖项，并使用`@ComponentScan`来查找bean。

如果你像上面建议的那样构建代码(将应用程序类定位在顶级包中)，你可以添加`@ComponentScan`而不带任何参数，或者使用`@SpringBootApplication`注释，它隐含包含`@ComponentScan`。
您的所有应用程序组件(`@Component`、`@Service`、`@Repository`、`@Controller`等)都会自动注册为Spring bean。

下面的例子展示了一个`@Service` Bean，它使用构造函数注入来获得所需的`RiskAssessor` Bean:
``` Java
@Service
public class MyAccountService implements AccountService {

    private final RiskAssessor riskAssessor;

    public MyAccountService(RiskAssessor riskAssessor) {
        this.riskAssessor = riskAssessor;
    }

    // ...

}
```

如果一个bean有多个构造函数，你需要用`@Autowired`标记你想让Spring使用的构造函数:
``` Java
@Service
public class MyAccountService implements AccountService {

    private final RiskAssessor riskAssessor;

    private final PrintStream out;

    @Autowired
    public MyAccountService(RiskAssessor riskAssessor) {
        this.riskAssessor = riskAssessor;
        this.out = System.out;
    }

    public MyAccountService(RiskAssessor riskAssessor, PrintStream out) {
        this.riskAssessor = riskAssessor;
        this.out = out;
    }

    // ...
}
```
> 请注意，使用构造函数注入是如何让`riskAssessor`字段被标记为`final`的，这表明它随后不能被更改。

# 使用@SpringBootApplication注释
许多Spring Boot开发者喜欢他们的应用使用自动配置、组件扫描，并且能够在他们的“应用类(application class)”中定义额外的配置。一个`@SpringBootApplication`注释可以用来启用这三个特性，即:
- `@EnableAutoConfiguration`:启用Spring Boot的自动配置机制
- `@ComponentScan`:在应用程序所在的包上启用@Component scan(参见最佳实践)
- `@SpringBootConfiguration`:允许在上下文中注册额外的bean或导入额外的配置类。Spring的标准`@Configuration`的替代方案，有助于在集成测试中进行配置检测。
``` Java
// Same as @SpringBootConfiguration @EnableAutoConfiguration @ComponentScan
@SpringBootApplication
public class MyApplication {

    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }

}
```
> `@SpringBootApplication`还提供了别名来定制`@EnableAutoConfiguration`和`@ComponentScan`的属性。

> 这些特性都不是强制性的，您可以选择用它支持的任何特性替换这个注释。例如，你可能不想在应用程序中使用组件扫描或配置属性扫描:
> ``` Java
> @SpringBootConfiguration(proxyBeanMethods = false)
> @EnableAutoConfiguration
> @Import({ SomeConfiguration.class, AnotherConfiguration.class })
> public class MyApplication {
> 
>     public static void main(String[] args) {
>         SpringApplication.run(MyApplication.class, args);
>     }
> 
> }
> ```
> 在这个例子中，`MyApplication`就像任何其他Spring Boot应用程序一样，除了`@Component`注释的类和`@ConfigurationProperties`注释的类不会被自动检测，用户定义的bean会被显式导入。


# 运行应用
将应用程序打包成`jar`并使用嵌入式HTTP服务器的最大优点之一是，您可以像运行其他应用程序一样运行应用程序。该示例适用于调试Spring Boot应用程序。您不需要任何特殊的IDE插件或扩展。

## 使用打包的jar运行应用
如果你使用Spring Boot Maven或Gradle插件来创建一个可执行的jar，你可以使用java -jar来运行你的应用程序，如下面的例子所示:
``` bash
java -jar target/myapplication-0.0.1-SNAPSHOT.jar
```
还可以运行启用了远程调试支持的打包应用程序。这样做可以将调试器附加到打包的应用程序中，如下例所示:
``` bash
$ java -Xdebug -Xrunjdwp:server=y,transport=dt_socket,address=8000,suspend=n \
       -jar target/myapplication-0.0.1-SNAPSHOT.jar
```
## 使用Maven插件
Spring Boot Maven插件包含一个运行目标，可用于快速编译和运行应用程序。应用程序以分解的形式运行，就像它们在IDE中一样。下面的例子展示了一个典型的Maven命令来运行Spring Boot应用程序:
``` bash
$ mvn spring-boot:run
```
您可能还想使用`MAVEN_OPTS`操作系统环境变量，如下例所示:
``` bash
$ export MAVEN_OPTS=-Xmx1024m
```

# 开发者工具
Spring Boot包含一组额外的工具，这些工具可以使应用程序开发体验更加愉快。
`spring-boot-devtools`模块可以包含在任何项目中，以提供额外的开发时特性。要包含devtools支持，请在构建中添加模块依赖项，如下所示，用于`Maven`:
``` XML
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-devtools</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```
## 诊断类加载问题
如重启与重新加载一节所述，重启功能是通过使用两个类加载器实现的。对于大多数应用程序，这种方法工作得很好。然而，它有时会导致类加载问题，特别是在多模块项目中。

要诊断类加载问题是否确实是由devtools及其两个类加载器引起的，请尝试禁用重启。如果这解决了您的问题，请自定义重新启动类加载器以包含整个项目。

## 属性默认值
Spring Boot支持的几个库使用缓存来提高性能。例如，模板引擎缓存已编译的模板，以避免重复解析模板文件。此外，Spring MVC可以在提供静态资源时向响应添加HTTP缓存头。
虽然缓存在生产环境中非常有益，但在开发过程中可能适得其反，使您无法看到刚刚在应用程序中所做的更改。出于这个原因，spring-boot-devtools**默认禁用缓存选项**。
缓存选项通常由`application.properties`配置文件来设置。例如，`Thymleaf`提供了`spring.thymeleaf.cache`属性。spring-boot-devtools模块不需要手动设置这些属性，而是自动应用合理的开发时配置。

如果你不希望应用默认属性，你可以在`application.properties`中将`spring.devtools.add-properties`设置为`false`。

因为在开发Spring MVC和Spring WebFlux应用程序时你需要更多关于web请求的信息，开发者工具建议你在web日志组中启用DEBUG日志。
这将为您提供有关传入请求、哪个处理程序正在处理请求、响应结果和其他详细信息的信息。如果希望记录所有请求详细信息(包括潜在的敏感信息)，可以打开`spring.mvc.log-request-details`或`spring.codec.log-request-details`配置属性。

## 自动重启
### 排除资源
某些资源在被更改时不一定需要触发重启。例如，`Thymeleaf`模板可以就地编辑。
默认情况下，更改`/META-INF/maven`, `/META-INF/resources`, `/resources`, `/static`, `/public`,或`/templates`中的资源不会触发重启，但会触发实时重新加载。
如果你想自定义这些排除，你可以使用`spring.devtools.restart.exclude`属性。例如，要排除`/static`和`/public`，你可以设置以下属性:
``` YAML
spring:
  devtools:
    restart:
      exclude: "static/**,public/**"
```
### 禁用自动重启
如果不想使用重启功能，可以使用`spring.devtools.restart.enabled`属性禁用它。在大多数情况下，您可以在`application.properties`中设置此属性。(这样做仍然会初始化重启类加载器，但它不会监视文件更改)。

如果你需要完全禁用重启支持(例如，因为它不能与特定的库一起工作)，你需要在调用`SpringApplication.run(…​)`之前将System属性`spring.devtools.restart.enabled`设置为`false`，如下例所示:
``` java
@SpringBootApplication
public class MyApplication {

    public static void main(String[] args) {
        System.setProperty("spring.devtools.restart.enabled", "false");
        SpringApplication.run(MyApplication.class, args);
    }

}
```
## 全局配置
您可以通过将以下任何文件添加到`$HOME/.config/spring-boot`目录中来配置全局devtools设置:
- spring-boot-devtools.properties
- spring-boot-devtools.yaml
- spring-boot-devtools.yml

添加到这些文件中的任何属性都适用于您机器上使用devtools的**所有Spring Boot应用程序**。例如，要配置重启总是使用触发器文件，你可以在`spring-boot-devtools`文件中添加以下属性:
``` YAML
spring:
  devtools:
    restart:
      trigger-file: ".reloadtrigger"
```
## 远程程序
Spring Boot开发人员工具并不局限于本地开发。在远程运行应用程序时，还可以使用一些功能。
远程支持是可选的，因为启用它可能存在**安全风险**。只有在**可信网络上**运行或**使用SSL保护**时才应该启用它。如果这两个选项都不可用，你不应该使用DevTools的远程支持。**永远不要在生产部署中启用支持**。

要启用它，你需要确保devtools包含在重新打包的存档文件中，如下面的清单所示:
``` XML
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
                <excludeDevtools>false</excludeDevtools>
            </configuration>
        </plugin>
    </plugins>
</build>
```
然后需要设置`spring.devtools.remote.secret`属性。像任何重要的密码或密钥一样，该值应该是唯一的强口令，这样它就不能被猜测或暴力破解。

远程devtools支持分为两部分:
- 接受连接的服务器端端点
- 在IDE中运行的客户端应用程序。
当设置了s`pring.devtools.remote.secret`属性时，服务器组件将自动启用。客户端组件必须手动启动。