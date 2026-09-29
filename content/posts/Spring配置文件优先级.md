---
title: Spring配置文件优先级
id: ead1fecb-9ef0-4381-839e-cb8ec7d50459
date: 2024-02-22 14:21:06
auther: limxwu
cover: 
excerpt: SpringBoot配置文件优先级 3.2.2官方文档：Spring Boot 允许您外部化您的配置，以便您可以在不同的环境中使用相同的应用程序代码。您可以使用各种外部配置源，包括 Java 属性文件、YAML 文件、环境变量和命令行参数。 Spring Boot 使用一种非常特殊的 Propert
permalink: /archives/1708582866458
categories:
 - 文章
tags: 
 - java
---

# SpringBoot配置文件优先级

> [3.2.2官方文档：](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.external-config)Spring Boot 允许您外部化您的配置，以便您可以在不同的环境中使用相同的应用程序代码。您可以使用各种外部配置源，包括 Java 属性文件、YAML 文件、环境变量和命令行参数。

Spring Boot 使用一种非常特殊的 `PropertySource`顺序，旨在允许合理地覆盖值。后面的属性源可以覆盖前面定义的值。按以下顺序考虑来源：

1. 默认属性（由设置指定 `SpringApplication.setDefaultProperties`）。
2. [`@PropertySource`](https://docs.spring.io/spring-framework/docs/6.1.3/javadoc-api/org/springframework/context/annotation/PropertySource.html)类上的注释 `@Configuration`。`Environment`请注意，在刷新应用程序上下文之前，不会将此类属性源添加到中。现在配置某些属性为时已晚，例如在刷新开始之前读取的 `logging.*`和。`spring.main.*`
3. 配置数据（例如 `application.properties`文件）。
4. `RandomValuePropertySource`仅在 中具有属性 `random.*`。
5. 操作系统环境变量。
6. Java 系统属性 ( `System.getProperties()`)。
7. JNDI 属性来自 `java:comp/env`.
8. `ServletContext`初始化参数。
9. `ServletConfig`初始化参数。
10. 来自 `SPRING_APPLICATION_JSON`（嵌入环境变量或系统属性中的内联 JSON）的属性。
11. 命令行参数。
12. `properties`您的测试的属性。可用于[测试应用程序的特定部分的测试](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.testing.spring-boot-applications.autoconfigured-tests)[`@SpringBootTest`](https://docs.spring.io/spring-boot/docs/3.2.2/api/org/springframework/boot/test/context/SpringBootTest.html)注释。
13. [`@DynamicPropertySource`](https://docs.spring.io/spring-framework/docs/6.1.3/javadoc-api/org/springframework/test/context/DynamicPropertySource.html)测试中的注释。
14. [`@TestPropertySource`](https://docs.spring.io/spring-framework/docs/6.1.3/javadoc-api/org/springframework/test/context/TestPropertySource.html)对你的测试进行注释。
15. `$HOME/.config/spring-boot`当 devtools 处于活动状态时目录中的Devtools[全局设置属性](https://docs.spring.io/spring-boot/docs/current/reference/html/using.html#using.devtools.globalsettings)。

配置数据文件按以下顺序考虑：

1. 打包在 jar 内的[应用程序属性](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.external-config.files) `application.properties`（和 YAML 变体）。
2. [特定于配置文件的应用程序属性](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.external-config.files.profile-specific)打包在 jar（`application-{profile}.properties`和 YAML 变体）内。
3. 打包的 jar（和 YAML 变体）之外的[应用程序属性](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.external-config.files) `application.properties`。
4. 打包的 jar（和 YAML 变体）之外的[特定于配置文件的应用程序属性](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.external-config.files.profile-specific) `application-{profile}.properties`。
