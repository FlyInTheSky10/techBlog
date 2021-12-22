---
title: Spring(3): AOP 模块
date: 2021-12-22 21:57
categories:
- Spring
tags:
- Spring#
---

# Spring AOP

AOP 即 Aspect Oriented Program：面向切面编程

在面向切面编程的思想里面，把功能分为核心业务功能，和周边功能。

- 比如登陆，增加数据，删除数据都是**核心业务**
- 性能统计，日志，事务管理都是**周边业务**

周边功能被定义为切面

在面向切面编程AOP的思想里面，核心业务功能和切面功能分别独立进行开发，然后把切面功能和核心业务功能 "编织" 在一起，这就叫AOP<!-- more -->

# 概念

- 切入点（Pointcut）
  在哪些类，哪些方法上切入（**where**）

  **注：需要实现切入的方法**

- 通知（Advice）
  在方法执行的什么实际（**when:**方法前/方法后/方法前后）做什么（**what:**增强的功能）

- 切面（Aspect）
  切面 = 切入点 + 通知，通俗点就是：**在什么时机，什么地方，做什么增强！**

  **注：即周边功能**

- 织入（Weaving）
  把切面加入到对象，并创建出代理对象的过程。（由 Spring 来完成）

  **注：即最终的结果**

# 使用注解来开发 Spring AOP

## 第一步：选择连接点

**选择哪一个类的哪一方法用以增强功能。**

```java
package pojo;

import org.springframework.stereotype.Component;

@Component("landlord")
public class Landlord {

	public void service() {
        // 仅仅只是实现了核心的业务功能
		System.out.println("签合同");
		System.out.println("收房租");
	}
}
```

我们在这里就选择上述 Landlord 类中的 service() 方法作为连接点。

## 第二步：创建切面

我们可以把切面理解为一个拦截器，当程序运行到连接点的时候，被拦截下来，在开头加入了初始化的方法，在结尾也加入了销毁的方法而已

在 Spring 中只要使用 `@Aspect` 注解一个类，那么 Spring IoC 容器就会认为这是一个切面了

```java
package aspect;

import org.aspectj.lang.annotation.After;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.springframework.stereotype.Component;

@Component
@Aspect
class Broker {

    @Before("execution(* pojo.Landlord.service())")
    public void before(){
        System.out.println("带租客看房");
        System.out.println("谈价格");
    }

    @After("execution(* pojo.Landlord.service())")
    public void after(){
        System.out.println("交钥匙");
    }
}
```

**注意：** 被定义为切面的类仍然是一个 Bean ，需要 `@Component` 注解标注

Spring 中的 AspectJ 注解：

| 注解              | 说明                                                         |
| :---------------- | :----------------------------------------------------------- |
| `@Before`         | 前置通知，在连接点方法前调用                                 |
| `@Around`         | 环绕通知，它将覆盖原有方法，但是允许你通过反射调用原有方法，后面会讲 |
| `@After`          | 后置通知，在连接点方法后调用                                 |
| `@AfterReturning` | 返回通知，在连接点方法执行并正常返回后调用，要求连接点方法在执行过程中没有发生异常 |
| `@AfterThrowing`  | 异常通知，当连接点方法异常时调用                             |

## 第三步：定义切点

上面的注解中定义了 execution 的正则表达式，Spring 通过这个正则表达式判断具体要拦截的是哪一个类的哪一个方法

通过使用 `@Pointcut` 注解来定义一个切点

```java
package aspect;

import org.aspectj.lang.annotation.After;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.aspectj.lang.annotation.Pointcut;
import org.springframework.stereotype.Component;

@Component
@Aspect
class Broker {

	@Pointcut("execution(* pojo.Landlord.service())")
	public void lService() {
	}

	@Before("lService()")
	public void before() {
		System.out.println("带租客看房");
		System.out.println("谈价格");
	}

	@After("lService()")
	public void after() {
		System.out.println("交钥匙");
	}
}
```

## 第四步：测试 AOP

在 applicationContext.xml 中配置自动注入，并告诉 Spring IoC 容器去哪里扫描这两个 Bean

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd">

    <context:component-scan base-package="aspect" />
    <context:component-scan base-package="pojo" />

    <aop:aspectj-autoproxy/>
</beans>
```

在 Package test 下编写测试代码：

```java
package test;

import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;
import pojo.Landlord;

public class TestSpring {

	public static void main(String[] args) {

		ApplicationContext context =
				new ClassPathXmlApplicationContext("applicationContext.xml");
		Landlord landlord = (Landlord) context.getBean("landlord", Landlord.class);
		landlord.service();

	}
}
```

## 环绕通知

```java
package aspect;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.stereotype.Component;

@Component
@Aspect
class Broker {

//  注释掉之前的 @Before 和 @After 注解以及对应的方法
//	@Before("execution(* pojo.Landlord.service())")
//	public void before() {
//		System.out.println("带租客看房");
//		System.out.println("谈价格");
//	}
//
//	@After("execution(* pojo.Landlord.service())")
//	public void after() {
//		System.out.println("交钥匙");
//	}

    //  使用 @Around 注解来同时完成前置和后置通知
	@Around("execution(* pojo.Landlord.service())")
	public void around(ProceedingJoinPoint joinPoint) {
		System.out.println("带租客看房");
		System.out.println("谈价格");

		try {
			joinPoint.proceed(); // 调用原事件
		} catch (Throwable throwable) {
			throwable.printStackTrace();
		}

		System.out.println("交钥匙");
	}
}
```

# 使用 XML 配置开发 Spring AOP

此处略去，具体可以看原文

本文引用于：
https://www.cnblogs.com/wmyskxz/p/8835243.html

(本文作为上述文章重点提取以及总结笔记)
