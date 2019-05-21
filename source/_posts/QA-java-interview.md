---
title: Java面试题收集
date: 2019-04-09 21:42:50
tags: [java]
---
## Question 为什么要使用Spring Boot？
### 分析
问题关键点在于回答 Spring Boot的核心特点，相较于Spring。
### Answer
Spring Boot 的开启注解是 @SpringBootApplication,其由如下三个注解组成：
- @Configuration
- @ComponentScan
- @EnableAutoConfiguration
前面两个都是Spring自带，与Spring Boot无关，核心是第三个@EnableAutoConfiguration注解，它能根据路径下的jar包和配置动态加载配置和注入bean。
具体一点就是可以和maven搭配使用，在pom.xml中引入依赖jar包，而在配置文件中设置依赖jar包的启动参数，Spring Boot会自动的将参数配置到引用jar包配置的bean中。
而在Spring中需要手动的将参数要么配置在XML中，要么配置在java bean中。
