---
title: Spring Cloud Task
tags:
category:
---
[参考](https://docs.spring.io/spring-cloud-task/docs/current/reference/html/)

Spring Cloud Task使创建`短期(short-lived)微服务`变得容易。它提供的功能允许在**生产环境**中**按需**执行短期的JVM进程。

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
1. 法TaskRepository
# 