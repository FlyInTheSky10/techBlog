---
title: Spring MVC 初探
date: 2021-12-23 14:38
categories:
- Spring
tags:
- Spring
---

# MVC 模式

MVC模式中，M是指业务模型，V是指用户界面，C则是控制器，使用MVC的目的是将M和V的实现代码分离，从而使同一个程序可以使用不同的表现形式。

- **M 模型（Model）**
   dao, bean
- **V 视图（View）**
  网页, JSP
- **C 控制器（controller)**
  Servlet<!-- more -->

# 使用注解配置 Spring MVC

## 第一步：编写 HelloController

在 Package controller 下创建  HelloController 类，并实现 org.springframework.web.servlet.mvc.Controller 接口：

```java
package controller;

import org.springframework.web.servlet.ModelAndView;
import org.springframework.web.servlet.mvc.Controller;

public class HelloController implements Controller{
	public ModelAndView handleRequest(javax.servlet.http.HttpServletRequest httpServletRequest, javax.servlet.http.HttpServletResponse httpServletResponse) throws Exception {
		ModelAndView mav = new ModelAndView("index.jsp");
		mav.addObject("message", "Hello Spring MVC");
		return mav;
	}
}
```

## 第二步：准备 index.jsp

```jsp
<%@ page language="java" contentType="text/html; charset=UTF-8"
    pageEncoding="UTF-8" isELIgnored="false"%>
 
<h1>${message}</h1>
```

## 第三步：为 HelloController 添加注解

```java
package controller;

import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.servlet.ModelAndView;

@Controller
public class HelloController {

	@RequestMapping("/hello")
	public ModelAndView handleRequest(javax.servlet.http.HttpServletRequest httpServletRequest, javax.servlet.http.HttpServletResponse httpServletResponse) throws Exception {
		ModelAndView mav = new ModelAndView("index.jsp");
		mav.addObject("message", "Hello Spring MVC");
		return mav;
	}
}
```

- `@Controller` 注解：
  用来声明控制器（仅仅是辅助实现组件扫描）
- `@RequestMapping` 注解：
  表示路径 `/hello` 会映射到该方法上

## 第四步：XML 注释扫描

修改 web.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"        xmlns:context="http://www.springframework.org/schema/context"        xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">      

	<context:component-scan base-package="controller"/>
</beans>
```

## @RequestMapping 注解细节

如果 `@RequestMapping` 作用在类上，那么就相当于是给该类所有配置的映射地址前加上了一个地址

```java
@Controller
@RequestMapping("/wmyskxz")
public class HelloController {
	@RequestMapping("/hello")
	public ModelAndView handleRequest(....) throws Exception {
		....
	}
}
```

则访问地址： `localhost/wmyskxz/hello`



https://www.cnblogs.com/wmyskxz/p/8848461.html

(本文作为上述文章重点提取以及总结笔记)
