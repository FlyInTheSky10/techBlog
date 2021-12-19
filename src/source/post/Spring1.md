---
title: Spring(1): 初识 Spring
date: 2021-12-19 14:57
categories:
- Spring
tags:
- Spring
---

# 简介

Spring 框架是 Java 应用最广的框架。它的理念包括 **IoC (Inversion of Control，控制反转)** 和 **AOP(Aspect Oriented Programming，面向切面编程)**。

Spring 提倡以**“最少侵入”**的方式来管理应用中的代码，这意味着我们可以随时安装或者卸载 Spring

# 常用术语

- **JavaBean：**即**符合 JavaBean 规范**的 Java 类
- **POJO：**即 **Plain Old Java Objects，简单老式 Java 对象**
  它可以包含业务逻辑或持久化逻辑，但**不担当任何特殊角色**且**不继承或不实现任何其它Java框架的类或接口。**
- **容器：**在日常生活中容器就是一种盛放东西的器具，从程序设计角度看就是**装对象的的对象**，因为存在**放入、拿出等**操作，所以容器还要**管理对象的生命周期**。
<!-- more -->

# 优势

- **低侵入 / 低耦合** （降低组件之间的耦合度，实现软件各层之间的解耦）
- **声明式事务管理**（基于切面和惯例）
- **方便集成其他框架**（如MyBatis、Hibernate）

# IoC 和 DI

## IoC

IoC：Inverse of Control（控制反转）

读作**“反转控制”**，更好理解，不是什么技术，而是一种**设计思想**，就是**将原本在程序中手动创建对象的控制权，交由Spring框架来管理。**

## DI

Dependency Injection（依赖注入）

指 Spring 创建对象的过程中，**将对象依赖属性（简单值，集合，对象）通过配置设值给该对象**

# AOP

## 介绍

AOP 即 Aspect Oriented Program 面向切面编程

在面向切面编程的思想里面，把功能分为核心业务功能，和周边功能。

- **所谓的核心业务**，比如登陆，增加数据，删除数据都叫核心业务
- **所谓的周边功能**，比如性能统计，日志，事务管理等等

周边功能在 Spring 的面向切面编程AOP思想里，即被定义为切面

在面向切面编程AOP的思想里面，核心业务功能和切面功能分别独立进行开发，然后把切面功能和核心业务功能 "编织" 在一起，这就叫AOP

## 目的

AOP能够将那些与业务无关，**却为业务模块所共同调用的逻辑或责任（例如事务处理、日志管理、权限控制等）封装起来**，便于**减少系统的重复代码**，**降低模块间的耦合度**，并**有利于未来的可拓展性和可维护性**。

## 概念

- 切入点（Pointcut）
  在哪些类，哪些方法上切入（**where**）
- 通知（Advice）
  在方法执行的什么实际（**when:**方法前/方法后/方法前后）做什么（**what:**增强的功能）
- 切面（Aspect）
  切面 = 切入点 + 通知，通俗点就是：**在什么时机，什么地方，做什么增强！**
- 织入（Weaving）
  把切面加入到对象，并创建出代理对象的过程。（由 Spring 来完成）

# test 类

```java
package test;

import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.testng.annotations.Test;
import pojo.Student;
import pojo.StudentConfig;


public class TestSpring {

    @Test
    public void test() {
        ApplicationContext context = new AnnotationConfigApplicationContext(StudentConfig.class);
        Student student = (Student) context.getBean("student1", Student.class);
        student.printInformation();
    }
}
```



本文引用于：[https://www.cnblogs.com/wmyskxz/p/8820371.html](https://www.cnblogs.com/wmyskxz/p/8820371.html)

(本文作为该文章重点提取以及总结笔记)

