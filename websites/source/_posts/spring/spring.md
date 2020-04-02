---
title: Spring小结(1)
date: 2018-10-01 19:27:56
categories: 
- Spring技术小结
tags: 
- Spring技术小结
---

<!-- more -->

> Spring降低Java开发成本，采取以下4种关键策略

* 基于POJO的轻量级和最小侵入性编程
* 通过依赖注入和面向接口实现松耦合
* 基于切面和惯例进行声明式编程
* 通过切面和模板减少样板式代码

###### 依赖注入（DI）

* 作用：通过将对象的依赖关系交由系统中负责协调各对象的第三方组件（Spring容器）在创建对象的时候进行设定，对象无需自行创建或管理它们的依赖关系
* 装配：创建应用组件之间协作的行为通常称为装配
* 工作原理：Spring通过应用上下文(Application Context)装载bean的定义并把它们组装起来。Spring应用上下文全权负责对象的创建和组装。Spring自带了多种应用上下文的实现，它们之间主要的区别仅仅 在于如何加载配置

###### 应用切面(AOP)

* 作用：允许你把遍布应用各处的功能分离出来形成可重用的组件
* 组成：横切关注点+切点+切面+通知
* 横切关注点：系统由许多不同的组件组成，每一个组件各负责一块特定功能。除了实现自身核心的功能之外，这些组件还经常承担着额外的职责。诸如日志、事务管理和安全这样的系统服务经常融入到自身具有核心业务逻辑的组件中去，这些系统服务通常被称为横切关注点，因为它们会跨越系统的多个组件
* 切面：定义横切关注点的JavaBean对象
* 切点：业务组件对象中需要引入切面定义的系统服务，增强原先业务组件对象的方法服务，即业务组件原先的服务方法称为切点
* 通知：即在切点前后以及抛出异常时捕获住的切面方法

###### 使用模板消除样板式代码

* 通常为了实现通用的和简单的任务，你不得不一遍遍地重 复编写这样的代码，如使用JDBC访问查询数据的操作（打开连接+拼接sql+发送sql+解析结果集+关闭连接），其中样板式代码就是打开连接+发送sql+关闭连接，这些都可以通过模板来消除进而实现只关注核心的业务代码的编写
	
###### Spring容器

* 作用：Spring容器负责创建对象，装配它们，配置它们并管理它们的整个生命周期，从生存到死亡(在这里，可能 就是new到finalize()）
* bean工厂：org.springframework.beans.factory.BeanFactory，最简单的容器，提供基本的DI支持
* 应用上下文：org.springframework.context.ApplicationContext，基于BeanFactory构建，并提供应用框架级别的服务，例如从属性文件解析文本信息以及发布应用事件 给感兴趣的事件监听者

###### Spring应用上下文

* AnnotationConfigApplicationContext:从一个或多个基于Java的配置类中加载 Spring应用上下文
* AnnotationConfigWebApplicationContext:从一个或多个基于Java的配置类中加 载Spring Web应用上下文
* ClassPathXmlApplicationContext:从类路径下的一个或多个XML配置文件中加载 上下文定义，把应用上下文的定义文件作为类资源
* FileSystemXmlapplicationcontext:从文件系统下的一个或多个XML配置文件中加载上下文定义
* XmlWebApplicationContext:从Web应用下的一个或多个XML配置文件中加载上下文定义
**FileSystemXmlApplicationContext在指定的文件系统 路径下查找knight.xml文件;ClassPathXmlApplicationContext是在所有的类路径(包含 JAR文件)下查找 knight.xml文件**

###### Spring 管理bean的生命周期

1. Spring对bean进行实例化
2. Spring将值和bean的引用注入到bean对应的属性中
3. 如果bean实现了BeanNameAware接口，Spring将bean的ID传递给setBeanName()方法
4. 如果bean实现了BeanFactoryAware接口，Spring将调用setBeanFactory()方法，将 BeanFactory容器实例传入
5. 如果bean实现了ApplicationContextAware接口，Spring将调用setApplicationContext()方法，将bean所在的应用上下文的引用传入进来
6. 如果bean实现了BeanPostProcessor接口，Spring将调用它们的postProcessBeforeInitialization()方法
7. 如果bean实现了InitializingBean接口，Spring将调用它们的after- PropertiesSet()方法。类似地，如果bean使用init-method声明了初始化方法，该方法也会被调用
8. 如果bean实现了BeanPostProcessor接口，Spring将调用它们的postProcessAfterInitialization()方法
9. 此时，bean已经准备就绪，可以被应用程序使用了，它们将一直驻留在应用上下文中，直到该应用上下文被销毁
10. 如果bean实现了DisposableBean接口，Spring将调用它的destroy()接口方法。同样， 如果bean使用destroy-method声明了销毁方法，该方法也会被调用

###### Spring 模块

* 核心容器：管理着Spring应用中bean的创建、配置和管理；在该模块中， 包括了Spring bean工厂，它为Spring提供了DI的功能，同时也提供了许多企业服务，例如E-mail、JNDI访问、EJB集成和调度
* AOP模块：AOP可以帮助应用对象解耦，借助于AOP，可以将遍布系统的关注点(例如 事务和安全)从它们所应用的对象中解耦出来
* 数据访问与集成：
	* 本模块同样包含了在JMS(Java Message Service)之上构建的Spring抽象层，它会使用消息以异 步的方式与其他应用集成
   *  为多个ORM框架提供一种构建DAO的简便方式
* Web与远程调用：Spring的Web和远程调用自带了一个强大的MVC框架(MVC模式是一种普遍被接受的构建Web应用的方法，它可以帮助用户将 界面逻辑与应用逻辑分离)
* Instrumentation：提供了为JVM添加代理(agent)的功能。具体来讲，它为Tomcat提供了一个织入代理，能够为Tomcat传递类文件，就像这些文件是被类加载器加载的一样
* 测试：Spring为使用JNDI、Servlet和Portlet编写单元测试提供了一系列的mock对 象实现。对于集成测试，该模块为加载Spring应用上下文中的bean集合以及与Spring上下文中的 bean进行交互提供了支持

###### Spring Portfolio

* Spring Web Flow：为基于流程的会话式Web应用(可以想一下购物 车或者向导功能)提供了支持，[参考Spring WebFlow](http://projects.spring.io/spring-webflow/)
* Spring Web Service：提供了契约优先的Web Service模型，服务的实现都是为了满 足服务的契约而编写的，[参考Spring WebService](http://docs.spring.io/spring-%20ws/site/)
* Spring Security：安全对于许多应用都是一个非常关键的切面。利用Spring AOP，Spring Security为Spring应用提供 了声明式的安全机制，[参考Spring Security](http://projects.spring.io/spring-security/)
* Spring Integration：许多企业级应用都需要与其他应用进行交互。Spring Integration提供了多种通用应用集成模式的 Spring声明式风格实现，[参考Spring Integration](http://projects.spring.io/spring-integration/)
* Spring Batch：当我们需要对数据进行大量操作时，没有任何技术可以比批处理更胜任这种场景。如果需要开发一个批处理应用，你可以通过Spring Batch，使用Spring强大的面向POJO的编程模型，[参考Spring Batch](http://projects.spring.io/%20spring-batch/)
* Spring Data：不管你使用文档数据库，如MongoDB，图数据库，如Neo4j，还是传统的关系型数据库，Spring Data都为持久化提供了一种简单的编程模型。这包括为多种数据库类型提供了一种自动化的 Repository机制，它负责为你创建Repository的实现
* Spring Social：Spring的一个社交网络扩展模块，更多的是关注连 接(connect)，而不是社交(social)。它能够帮助你通过REST API连接Spring应用，其中有些 Spring应用可能原本并没有任何社交方面的功能目标，[Spring facebook](https://spring.io/guides/gs/accessing-facebook/)，[Spring twitter](https://spring.io/guides/gs/accessing-twitter/)
* Spring Mobile：Spring Mobile是Spring MVC新的扩展模块，用于支持移动Web应用开发
* Spring for Android：旨在通过Spring框架为开发基于 Android设备的本地应用提供某些简单的支持，[参考Spring Android](http://projects.spring.io%20/spring-android/)
* Spring Boot：Spring Boot大量依赖于自动配置技术，它能够消除大部分(在很多场景中，甚至是全部)Spring 配置。它还提供了多个Starter项目，不管你使用Maven还是Gradle，这都能减少Spring工程构建文件的大小

###### 耦合的两面性

* 一方面，紧密耦合的代码难以测试、难以复用、难以理解，并且典型地表现出“打地鼠”式的bug特性(修复一个bug，将会出现一个或者更多新的bug)。
*  另一方面，一定程度的耦合又是必须的——完全没有耦合的代码什么也做不了。为了完成有实际意 义的功能，不同的类必须以适当的方式进行交互。总而言之，耦合是必须的，但应当被小心谨慎地管理