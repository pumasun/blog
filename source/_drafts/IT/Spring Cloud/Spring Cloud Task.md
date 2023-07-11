---
title: Spring Cloud Task
tags:
category:
---
[参考](https://docs.spring.io/spring-cloud-task/docs/current/reference/html/)

Spring Cloud Task使创建`短期(short-lived)微服务`变得容易。它提供的功能允许在**生产环境**中**按需**执行短期的JVM进程。


# 应用场景
- 微服务中，通过MQ处理短期任务或长期任务
- 本地Cron任务

# 系统要求
Spring Cloud Task使用**关系数据库**来存储执行任务的结果.
- 开发环境，可选择
  - H2
- 生产环境，可选择
  - DB2
  - HSQLDB
  - MySql
  - Oracle
  - Postgres
  - SqlServer

> 如果不设定数据源url或者内嵌数据库,则程序无法启动.

# Hello World

添加Spring Cloud Task依赖
``` xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-task</artifactId>
</dependency>
```

``` Java
package io.spring.Helloworld;

@SpringBootApplication
@EnableTask
public class HelloworldApplication {

    @Bean
    public ApplicationRunner applicationRunner() {
        return new HelloWorldApplicationRunner();
    }

    public static void main(String[] args) {
        SpringApplication.run(HelloworldApplication.class, args);
    }

    public static class HelloWorldApplicationRunner implements ApplicationRunner {

        @Override
        public void run(ApplicationArguments args) throws Exception {
            System.out.println("Hello, World!");

        }
    }
}
```
启用Spring Cloud Task的Debug日志
``` properties
logging.level.org.springframework.cloud.task=DEBUG
```

`TaskLifecycleListener` 记录task的开始和结束




# Spring Cloud Task的生命周期

Spring Cloud Task只记录`CommandLineRunner`或`ApplicationRunner`启动的Task

- start event
  - 由TaskRepository记录
  - 由SmartLifecycle#start触发
  - `ApplicationContext`成功启动之后
  - 在`CommandLineRunner`或`ApplicationRunner`运行之前
  - 初始化一类的bean

- end event
  - 由TaskRepository记录运行结果(成功亦或失败)
    > 如果应用程序要求`ApplicationContext`在任务完成时关闭（所有`*Runner#run`方法已调用并且已更新任务存储库），请将属性`spring.cloud.task.closecontextEnabled` 设置为 `true`。

# TaskExecution
`TaskRepository`中存储的任务信息的类。
|Name|Description|
|-- | --|
| executionid | 任务运行的唯一 ID |
| exitCode | 从`ExitCodeExceptionMapper`实现生成的退出代码。如果没有生成退出代码但抛出了`ApplicationFailedEvent`异常，则设置为 1。否则，假定为 0。|
| taskName | 任务的名称，由配置的`TaskNameResolver`决定。 |
| startTime | 任务开始的时间，在调用`SmartLifecycle#start`时决定。 |
| endTime | 任务完成的时间，由`ApplicationReadyEvent`决定。 |
| exitMessage | 退出时可用的任何信息。这可以通过编程方式在`TaskExecutionListener`中设定。 |
| errorMessage | 如果任务因异常结束（如 `ApplicationFailedEvent`），则该此处存储异常的堆栈Trace信息。 |
| arguments | 传递到可执行引导应用程序时的字符串命令行参数List。|

# 退出代码
任务结束时，指定的退出代码将被返回到OS中。
可以通过`ExitCodeExceptionMapper`接口将未捕捉的异常映射为对应的推出代码，在返回到OS的同时，也将被Spring Cloud Task记录下来。
如果应用由 `SIG-INT` 或 `SIG-TERM`信号终止，则退出代码为0,除非在代码中指定。

# 配置
## 数据源

## 数据表前缀

## 启用/禁用数据表初始化

## 外部生成的Task ID

## 外部Task ID

## 父Task ID

## TaskConfigurer


# TaskExecutionListener