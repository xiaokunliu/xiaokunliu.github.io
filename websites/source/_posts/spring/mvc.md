---
title: SpringMVC(1)
date: 2018-10-01 19:27:56
categories: 
- Spring技术小结
tags: 
- Spring技术小结
---

<!-- more -->


> SpringMVC 原理

![springMVC 执行流程](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwOTI4MTUyOTMwMTY5?x-oss-process=image/format,png)

* s1:浏览器携带请求数据request发送到SpringMVC前端控制器DispacherServlet中
* s2：DispacherServlet将请求转发给SpringMVC处理器映射器
* s3：处理器映射器(handler mapping)根据request携带的url信息来进行决策要将request请求给哪个控制器处理并告知DispacherServlet
* s4：DispacherServlet根据handler mapping决策将request发送到控制器，请求会卸下其负载的信息并等待控制器将业务委托给一个或者多个服务对象处理返回
* s5：控制器将数据模型（委托的服务返回给用户显示的信息）打包并以特定的视图形式发送回DispatcherServlet
* s6：DispatcherServlet将一个视图的逻辑名传递给视图解析器来确定返回给用户显示的指定视图实现
* s7：视图将使用模型数据渲染输出，这个输出会通过响应对象传递给客户端

> AbstractAnnotationConfigDispatcherServletInitializer解析

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcwOTI5MTM0NTAzOTM0?x-oss-process=image/format,png)

* 在servlet3.0的环境中，容器会在类路径中查找实现接口类的javax.servlet.ServletContainerInitializer，如果找到就用它来配置servlet容器
* Spring的SpringServletContainerInitializer实现这个接口并将由所有实现WebApplicationInitializer接口的类来做一系列的配置任务
* Spring中的AbstractAnnotationConfigDispatcherServletInitializer实现了WebApplicationInitializer接口，因此我们也可以在Java Config中配置先前web.xml的配置

> 使用java的config来配置servlet容器
```java
public class WebInitializer extends AbstractAnnotationConfigDispatcherServletInitializer{
    /**
     * 配置web映射路径
     * @return
     */
    @Override
    protected String[] getServletMappings() {
        return new String[]{"/"};
    }

    @Override
    protected Class<?>[] getRootConfigClasses() {
        return new Class<?>[0];
    }

    /**
     * 指定Web的配置类
     * @return
     */
    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class<?>[]{WebConfig.class};
    }
}
```

> SpringMVC 两个上下文对象

* DispatcherServlet：加载包含Web组件的bean，如控制器、视图解析器以及处理器映射
* ContextLoaderListener： 加载应用中的其他bean，这些bean通常是驱动应用后端的中间层和数据层组件

> DispatcherServlet 使用Java Config来配置并启动容器

```java
@Configuration
@EnableWebMvc
@ComponentScan("com.dtrees.springmvc.web")      // 启动扫描器
public class WebConfig extends WebMvcConfigurerAdapter{

    /**
     * 配置JSP视图解析器
     * @return
     */
    @Bean
    public ViewResolver viewResolver(){
        InternalResourceViewResolver resolver = new InternalResourceViewResolver();
        resolver.setPrefix("/WEB-INF/views/");
        resolver.setSuffix(".jsp");
        resolver.setExposeContextBeansAsAttributes(true);
        return resolver;
    }

    /**
     * 配置静态资源的处理
     * @param configurer
     */
    @Override
    public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {
        configurer.enable();
    }
}
```

> web以外的Java配置RootConfig.java

```java
@Configuration
@EnableAspectJAutoProxy
@ComponentScan(basePackages = {"com.keithl.project.core"})
public class RootConfig {
    // 注册一系列初始化beans
}
```