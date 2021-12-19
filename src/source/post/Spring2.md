---

title: Spring(2): IoC 实现
date: 2021-12-19 15:57
categories:
- Spring
tags:
- Spring
---

# Spring IoC

IoC：Inverse of Control（控制反转）

**将原本在程序中手动创建对象的控制权，交由Spring框架来管理。**

- **正控：**若要使用某个对象，需要**自己去负责对象的创建**
- **反控：**若要使用某个对象，只需要**从 Spring 容器中获取需要使用的对象，不关心对象的创建过程**，也就是把**创建对象的控制权反转给了Spring框架**

可用来降低对象之间的耦合。

# Spring IoC 容器

Spring 提供 **IoC 容器**来管理和容纳我们所开发的各种各样的 Bean，并且我们可以从中获取各种发布在 Spring IoC 容器里的 Bean，并且**通过描述**可以得到它。

Spring IoC 容器的设计主要是基于以下两个接口：

- **BeanFactory**
- **ApplicationContext**

其中 ApplicationContext 是 BeanFactory 的子接口之一

ApplicationContext 是其最高级接口之一，并对 BeanFactory 功能做了许多的扩展，所以在**绝大部分的工作场景下**，都会使用 ApplicationContext 作为 Spring IoC 容器。<!-- more -->

## BeanFactory

BeanFactory 位于设计的最底层，它提供了 Spring IoC 最底层的设计

- `getBean` 对应了多个方法来获取配置给 Spring IoC 容器的 Bean。
  ① 按照类型拿 bean：
  `bean = (Bean) factory.getBean(Bean.class);`
  **注意：**要求在 Spring 中只配置了一个这种类型的实例，否则报错。
  ② 按照 bean 的名字拿 bean:
  `bean = (Bean) factory.getBean("beanName");`
  **注意：**这种方法不太安全，IDE 不会检查其安全性（关联性）
  ③ 按照名字和类型拿 bean：**（推荐）**
  `bean = (Bean) factory.getBean("beanName", Bean.class);`
- `isSingleton` 用于判断是否单例，如果判断为真，其意思是该 Bean 在容器中是作为一个唯一单例存在的。而`isPrototype`则相反，如果判断为真，意思是当你从容器中获取 Bean，容器就为你生成一个新的实例。
  **注意：**在默认情况下，`isSingleton`为 ture，而`isPrototype`为 false
- 关于 type 的匹配，这是一个按 Java 类型匹配的方式
- `getAliases`方法是获取别名的方法

## ApplicationContext

ClassPathXmlApplicationContext 为 ApplicationContext 的子类

可以在 src 目录下创建一个 **applicationContext.xml** (后文介绍装配) 文件定义 **bean**，这样 Spring IoC 容器在初始化的时候就能找到它们

然后使用 ClassPathXmlApplicationContext 容器就可以将其初始化

```java
ApplicationContext context = new ClassPathXmlApplicationContext("applicationContext.xml");
Source source = (Source) context.getBean("source", Source.class);

System.out.println(source.getFruit());
System.out.println(source.getSugar());
System.out.println(source.getSize());
```

## ApplicationContext 的实现类

- **ClassPathXmlApplicationContext：**读取classpath中的资源

- **FileSystemXmlApplicationContext:** 读取指定路径的资源
- **XmlWebApplicationContext:** 需要在Web的环境下才可以运行

```java
XmlWebApplicationContext ac = new XmlWebApplicationContext(); // 这时并没有初始化容器
ac.setServletContext(servletContext); // 需要指定ServletContext对象
ac.setConfigLocation("/WEB-INF/applicationContext.xml"); // 指定配置文件路径，开头的斜线表示Web应用的根目录
ac.refresh(); // 初始化容器
```

# Bean 装配

## Bean 简介

1、所有属性为 private
2、提供默认构造方法
3、提供 getter 和 setter
4、实现 serializable 接口

## 方式

在 Spring 中提供了 3 种方法进行配置：

- 在 XML 文件中显式配置
- 在 Java 的接口和类中实现配置
- 隐式 Bean 的发现机制和自动装配原则

**最优先：通过隐式 Bean 的发现机制和自动装配的原则。**

- **好处：**减少程序开发者的决定权，简单又不失灵活。

**其次：Java 接口和类中配置实现配置**
在没有办法使用自动装配原则的情况下应该优先考虑此类方法

- **好处：**避免 XML 配置的泛滥，也更为容易。

3.**最后：XML 方式配置**
在上述方法都无法使用的情况下，那么也只能选择 XML 配置的方式。

## 通过 XML 配置装配 Bean

使用 XML 装配 Bean 需要定义对应的 XML，这里需要引入对应的 XML 模式（XSD）文件

一个简单的 XML 配置文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

</beans>
```

### 装配简易值

```xml
<bean id="c" class="pojo.Category">
    <property name="name" value="测试" />
</bean>
```

- `id` 属性是 Spring 能找到当前 Bean 的一个依赖的编号, 使用 `name` 属性来定义 bean 的名称
- `class` 属性是一个类的全限定名。缺省采用 **“全限定名#{number}”** 的格式生成编号。
- `property` 元素是定义类的属性，其中的 `name` 属性定义的是属性的名称，而 `value` 是它的值。

### 注入对象

使用 `ref` 属性

```xml
<!-- 配置 srouce 原料 -->
<bean name="source" class="pojo.Source">
    <property name="fruit" value="橙子"/>
    <property name="sugar" value="多糖"/>
    <property name="size" value="超大杯"/>
</bean>

<bean name="juickMaker" class="pojo.JuiceMaker">
    <!-- 注入上面配置的id为srouce的Srouce对象, 使用 ref 属性 -->
    <property name="source" ref="source"/>
</bean>
```

### 装配集合

```xml
<bean id="complexAssembly" class="pojo.ComplexAssembly">
    <!-- 装配Long类型的id -->
    <property name="id" value="1"/>
    
    <!-- 装配List类型的list -->
    <property name="list">
        <list>
            <value>value-list-1</value>
            <value>value-list-2</value>
            <value>value-list-3</value>
        </list>
    </property>
    
    <!-- 装配Map类型的map -->
    <property name="map">
        <map>
            <entry key="key1" value="value-key-1"/>
            <entry key="key2" value="value-key-2"/>
            <entry key="key3" value="value-key-2"/>
        </map>
    </property>
    
    <!-- 装配Properties类型的properties -->
    <property name="properties">
        <props>
            <prop key="prop1">value-prop-1</prop>
            <prop key="prop2">value-prop-2</prop>
            <prop key="prop3">value-prop-3</prop>
        </props>
    </property>
    
    <!-- 装配Set类型的set -->
    <property name="set">
        <set>
            <value>value-set-1</value>
            <value>value-set-2</value>
            <value>value-set-3</value>
        </set>
    </property>
    
    <!-- 装配String[]类型的array -->
    <property name="array">
        <array>
            <value>value-array-1</value>
            <value>value-array-2</value>
            <value>value-array-3</value>
        </array>
    </property>
</bean>
```

使用多个 `<ref>` 元素的 Bean 属性去引用之前定义好的 Bean

```xml
<property name="list">
    <list>
        <ref bean="bean1"/>
        <ref bean="bean2"/>
    </list>
</property>

<property name="map">
    <map>
        <entry key-ref="keyBean" value-ref="valueBean"/>
    </map>
</property>

<property name="set">
    <set>
        <ref bean="bean"/>
    </set>
</property>
```

### 命名空间

Spring 还提供了对应的命名空间的定义，只是在使用命名空间的时候要先引入对应的命名空间和 XML 模式（XSD）文件。

#### c-命名空间

它是在 XML 中更为简洁地描述构造器参数的方式，要使用它的话，必须要在 XML 的顶部声明其模式

```xml
xmlns:c="http://www.springframework.org/schema/c"
```

假设有这么一个类

```java
package pojo;

public class Student {

	int id;
	String name;

	public Student(int id, String name) {
		this.id = id;
		this.name = name;
	}
    // setter and getter
}
```

那么 xml 里

```xml
<!-- 引入 c-命名空间之前 -->
<bean name="student1" class="pojo.Student">
    <constructor-arg name="id" value="1" />
    <constructor-arg name="name" value="学生1"/>
</bean>

<!-- 引入 c-命名空间之后 -->
<bean name="student2" class="pojo.Student"
      c:id="2" c:name="学生2"/>
```

在此之后如果需要注入对象的话则要跟上 `-ref`

另一种写法

```xml
<bean name="student2" class="pojo.Student"
      c:_0="3" c:_1="学生3"/>
```

在 XML 中不允许数字作为属性的第一个字符，因此必须要添加一个下划线来作为前缀。

#### p-命名空间

p-命名空间则是用setter的注入方式来配置 bean ，同样的，我们需要引入声明：

```xml
xmlns:p="http://www.springframework.org/schema/p"
```

然后

```xml
<!-- 引入p-命名空间之前 -->
<bean name="student1" class="pojo.Student">
    <property name="id" value="1" />
    <property name="name" value="学生1"/>
</bean>

<!-- 引入p-命名空间之后 -->
<bean name="student2" class="pojo.Student" 
      p:id="2" p:name="学生2"/>
```

#### util-命名空间

工具类的命名空间，可以简化集合类元素的配置，同样的我们需要引入其声明

```xml
xmlns:util="http://www.springframework.org/schema/util"
```
那么
```xml
<!-- 引入util-命名空间之前 -->
<property name="list">
    <list>
        <ref bean="bean1"/>
        <ref bean="bean2"/>
    </list>
</property>

<!-- 引入util-命名空间之后 -->
<util:list id="list">
    <ref bean="bean1"/>
    <ref bean="bean2"/>
</util:list>
```

### 引入其他配置文件

让 applicationContext.xml 文件包含其他配置文件，使用 `<import>` 元素引入其他配置文件

例：

```xml
<import resource="bean.xml" />
```

## 通过注解装配 Bean

### 优势
1.可以减少 XML 的配置，当配置项多的时候，臃肿难以维护
2.功能更加强大，既能实现 XML 的功能，也提供了自动装配的功能，采用了自动装配后，程序猿所需要做的决断就少了，更加有利于对程序的开发，这就是“约定由于配置”的开发原则

### 方式

- **组件扫描：**通过定义资源的方式，让 Spring IoC 容器扫描对应的包，从而把 bean 装配进来。
- **自动装配：**通过注解定义，使得一些依赖关系可以通过注解完成。

## 使用@Compoent 装配 Bean

```java
package pojo;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

@Component(value = "student1")
public class Student {

	@Value("1")
	int id;
	@Value("student_name_1")
	String name;

    // getter and setter
}
```

- **@Component注解：**
  表示 Spring IoC 会把这个类扫描成一个 bean 实例，而其中的 `value` 属性代表这个类在 Spring 中的 `id`

  这就相当于在 XML 中定义的 Bean 的 id：`<bean id="student1" class="pojo.Student" />`

  也可以简写成 `@Component("student1")`，甚至直接写成 `@Component` ，对于不写的，Spring IoC 容器就默认以类名来命名作为 `id`，只不过首字母小写，配置到容器中。

- **@Value注解：**
  表示值的注入，跟在 XML 中写 `value` 属性是一样的。

然后要使用一个 StudentConfig 类告诉 Spring IoC 这个 Bean 的存在

```java
package pojo;
import org.springframework.context.annotation.ComponentScan;

@ComponentScan
public class StudentConfig {
}
```

- **该类和 Student 类位于同一包名下**
- **@ComponentScan注解：**
  代表进行扫描，**默认是扫描当前包的路径**，扫描所有带有 `@Component` 注解的 POJO。

然后通过 Spring 定义好的 Spring IoC 容器的实现类——AnnotationConfigApplicationContext 去生成 IoC 容器

```java
ApplicationContext context = new AnnotationConfigApplicationContext(StudentConfig.class);
Student student = (Student) context.getBean("student1", Student.class);
student.printInformation();
```

**弊端：**

- 对于 `@ComponentScan` 注解，它只是扫描所在包的 Java 类，但是更多的时候我们希望的是**可以扫描我们指定的类**
- 上面的例子只是注入了一些简单的值，测试发现，通过 `@Value` 注解并**不能注入对象**

重构之前写的 StudentConfig 类

```java
package pojo;

import org.springframework.context.annotation.ComponentScan;

@ComponentScan(basePackageClasses = pojo.Student.class)
public class StudentConfig {
}
```

`@ComponentScan` 注解存在着两个配置项：

- **basePackages：它可以配置一个 Java 包的数组**，Spring 会根据它的配置扫描对应的包和子包，将配置好的 Bean 装配进来
- **basePackageClasses**：它可以配置多个类， Spring 会根据配置的类所在的包，为包和子包进行扫描装配对应配置的 Bean

采用 basePackageClasses 会有错误提示，请尽量使用 basePackageClasses

## 自动装配——@Autowired

自动装配技术是**由 Spring 自己发现对应的 Bean，自动完成装配工作的方式，**它会应用到一个十分常用的注解 `@Autowired` 上，这个时候 **Spring 会根据类型去寻找定义的 Bean 然后将其注入**

### 实例

创建一个 StudentServiceImp 实现类

```java
package service;

import org.springframework.beans.factory.annotation.Autowired;
import pojo.Student;

@Component("studentService")
public class StudentServiceImp implements StudentService {

	@Autowired
	private Student student = null;

     // getter and setter

	public void printStudentInfo() {
		System.out.println("学生的 id 为：" + student.getId());
		System.out.println("学生的 name 为：" + student.getName());
	}
}
```

这里的 `@Autowired` 注解，表示**在 Spring IoC 定位所有的 Bean 后，这个字段需要按类型注入**，这样 IoC 容器就会**寻找资源**，然后将其注入。

修改 StudentConfig 类，告诉 Spring IoC 在哪里去扫描它

```java
package pojo;

import org.springframework.context.annotation.ComponentScan;

@ComponentScan(basePackages = {"pojo", "service"})
public class StudentConfig {
}
```

测试

```java
public class TestSpring {

	public static void main(String[] args) {
		// 通过注解的方式初始化 Spring IoC 容器
		ApplicationContext context = new AnnotationConfigApplicationContext(StudentConfig.class);
		StudentService studentService = context.getBean("studentService", StudentServiceImp.class);
		studentService.printStudentInfo();
	}
}
```

- **理解：** `@Autowired` 注解表示在 Spring IoC 定位所有的 Bean 后，再根据类型寻找资源，然后将其注入。
- **过程：** 定义 Bean → 初始化 Bean（扫描） → 根据属性需要从 Spring IoC 容器中搜寻满足要求的 Bean → 满足要求则注入
- **问题：** IoC 容器可能会寻找失败，此时会抛出异常（默认情况下，Spring IoC 容器会认为一定要找到对应的 Bean 来注入到这个字段，但有些时候并不是一定需要，比如日志）
- **解决：** 通过配置项 `required` 来改变，比如 `@Autowired(required = false)`

常见的 Bean 的 setter 方法也可以使用它来完成注入，一切需要 Spring IoC 去寻找 Bean 资源的地方都可以用到

```java
/* 包名和import */
public class JuiceMaker {
    ......
    @Autowired
    public void setSource(Source source) {
        this.source = source;
    }
}
```

@Autowired 会自动找到对应的 Bean 注入。

### 歧义性

如果我们现在有两个 Srouce 类型的资源，Spring IoC 就会不知所措，不知道究竟该引入哪一个 Bean

通过 Source.class 作为参数**无法判断使用哪个类实例进行返回**

消除歧义性，Spring 提供了两个注解：

- **@Primary 注解：**
  代表首要的，当 Spring IoC 检测到有多个相同类型的 Bean 资源的时候，会优先注入使用该注解的类。**该注解只是解决了首要的问题，但是并没有选择性的问题**
- **@Qualifier 注解：**
  Spring IoC 容器最底层的接口 BeanFactory 还提供了按名字查找的方法。

```java
/* 包名和import */
public class JuiceMaker {
    ......
    @Autowired
    @Qualifier("source1")
    public void setSource(Source source) {
        this.source = source;
    }
}
```

## @Bean 装配 Bean

`@Component` 只能注解在类上，当需要引用第三方包的（jar 文件），而且往往并没有这些包的源码，这时候将无法为这些包的类加入 `@Component` 注解，让它们变成开发环境中的 Bean 资源

**使用 `@Bean` 注解，注解到方法之上**，使其成为 Spring 中返回对象为 Spring 的 Bean 资源。

### 实例

新建一个用 `@Bean` 注解的类：

```java
@Configuration
public class BeanTester {

    @Bean(name = "testBean")
    public String test() {
        String str = "测试@Bean注解";
        return str; // 返回值为 Bean
    }
}
```

**注意：** `@Configuration` 注解相当于 XML 文件的根元素，**必须要**，有了才能解析其中的 `@Bean` 注解

在测试类中编写代码，从 Spring IoC 容器中获取到这个 Bean

```java
// 在 pojo 包下扫描
ApplicationContext context = new AnnotationConfigApplicationContext("pojo");
// 因为这里获取到的 Bean 就是 String 类型所以直接输出
System.out.println(context.getBean("testBean"));
```

`@Bean` 的配置项中包含 4 个配置项：

- **name：** 是一个字符串数组，允许配置多个 BeanName
- **autowire：** 标志是否是一个引用的 Bean 对象，默认值是 Autowire.NO
- **initMethod：** 自定义初始化方法
- **destroyMethod：** 自定义销毁方法

使用 `@Bean` 注解的好处就是能够动态获取一个 Bean 对象，能够根据环境不同得到不同的 Bean 对象。或者说将 Spring 和其他组件分离（其他组件不依赖 Spring，但是又想 Spring 管理生成的 Bean）

# Bean 的作用域

**在默认的情况下，Spring IoC 容器只会对一个 Bean 创建一个实例**，但有时候，我们希望能够通过 Spring IoC 容器获取多个实例，我们可以通过 `@Scope` 注解或者 `<bean>` 元素中的 `scope` 属性来设置

```java
// XML 中设置作用域
<bean id="" class="" scope="prototype" />
// 使用注解设置作用域
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
```

Spring 提供了 5 种作用域，它会根据情况来决定是否生成新的对象

| 作用域类别              | 描述                                                         |
| :---------------------- | :----------------------------------------------------------- |
| singleton(单例)         | 在Spring IoC容器中仅存在一个Bean实例 （默认的scope）         |
| prototype(多例)         | 每次从容器中调用Bean时，都返回一个新的实例，即每次调用getBean()时 ，相当于执行new XxxBean()：不会在容器启动时创建对象 |
| request(请求)           | 用于web开发，将Bean放入request范围 ，request.setAttribute("xxx") ， 在同一个request 获得同一个Bean |
| session(会话)           | 用于web开发，将Bean 放入Session范围，在同一个Session 获得同一个Bean |
| globalSession(全局会话) | 一般用于 Porlet 应用环境 , 分布式系统存在全局 session 概念（单点登录），如果不是 porlet 环境，globalSession 等同于 Session |

在开发中主要使用 `scope="singleton"`、`scope="prototype"`，**对于MVC中的Action使用prototype类型，其他使用singleton**，Spring容器会管理 Action 对象的创建,此时把 Action 的作用域设置为 prototype.

# Spring 表达式语言

- 使用 Bean 的 id 来引用 Bean
- 调用指定对象的方法和访问对象的属性
- 进行运算
- 提供正则表达式进行匹配
- 集合配置

例子：

```java
package pojo;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

@Component("elBean")
public class ElBean {
    // 通过 beanName 获取 bean，然后注入 
    @Value("#{role}")
    private Role role;
    
    // 获取 bean 的属性 id
    @Value("#{role.id}")
    private Long id;
    
    // 调用 bean 的 getNote 方法
    @Value("#{role.getNote().toString()}")
    private String note;
    /* getter and setter */
}
```

与属性文件中读取使用的 “`$`” 不同，在 Spring EL 中则使用 “`#`”

本文引用于：
[https://www.cnblogs.com/wmyskxz/p/8824597.html](https://www.cnblogs.com/wmyskxz/p/8824597.html)
[https://www.cnblogs.com/wmyskxz/p/8830632.html](https://www.cnblogs.com/wmyskxz/p/8830632.html)
[https://www.zhihu.com/question/19773379](https://www.zhihu.com/question/19773379)

(本文作为上述文章重点提取以及总结笔记)

