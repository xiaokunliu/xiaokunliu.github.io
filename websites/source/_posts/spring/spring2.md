---
title: Spring小结(2)
date: 2019-07-24 10:51:03
categories: 
- Spring
tags: 
- Spring小结(2)
---

<!-- more -->

创建应用对象之间协作关系的行为通常称为装配(wiring)，这也是依赖注入(DI)的本质
##### 三种装配机制

* 在XML中进行显式配置
* 在Java中进行显式配置
* 隐式的bean发现机制和自动装配

##### 自动化装配

![这里写图片描述](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTcxMDE3MTE1MjU4ODIw)
* 组件扫描(component scanning):Spring会自动发现应用上下文中所创建的bean

```java
// 启动自动扫描
@Configuration
@ComponentScan  // 默认扫描当前的java配置类下的包
public class CDPlayerConfig {}

// 定义javabean
public interface CompactDisc {
    void play();
}

// 实现类并声明组件
@Component("peppersBean")   // peppersBean为当前bean指定id名称
public class SgtPeppers implements CompactDisc{
	 /**
     * @Component:该类会作为组件类，并告知Spring要为这个类创建bean,前提是要启动扫描器
     * Spring应用上下文中所有的bean都会给定一个ID:将类名的第一个字母变为小写,即sgtPeppers
     * 自定义指定bean的Id:@Component(idName)
     * 也可以通过@Named注解来为bean设置ID,但是命名不好,不推荐使用
     */
    @Override
    public void play() {
        System.out.println(String.format("playing %s by %s",title,artist));
    }
}
```
* 自动装配(autowiring):Spring自动满足bean之间的依赖

```java
// 启动spring测试类测试
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = CDPlayerConfig.class)
public class CDPlayerTest {
    /**
     * SpringJUnit4ClassRunner:在测试开始的时候自动创建Spring的应用上下文
     * 注解@ContextConfiguration:告诉它需要在CDPlayerConfig中加载配置
     */

    /**
     * @Autowired:自动装配依赖的bean,即Spring自动满足bean依赖的一种方法，在满足依赖的过程中，会在 Spring应用上下文中寻找匹配某个bean需求的其他bean
     * 如果没有匹配的bean，那么在应用上下文创建的时候，Spring会抛出一个异常。为了避免异常的出现，你可以将@Autowired的required属性设置为false
     * 如果有多个bean都能满足依赖关系的话，Spring将会抛出一个异常，表明没有明确指定要选择哪 个bean进行自动装配
     */

    @Autowired
    private CompactDisc compactDisc;

    @Autowired
    private MediaPlayer cdPlayer;

    @Test
    public void testCompactDiscNotNull(){
        assertNotNull(compactDisc);
    }
}
```

##### 基于Java显式的配置

```java
// CDPlayer.java
@Configuration
@ComponentScan(basePackageClasses = {CompactDisc.class})
public class CDPlayerConfig {

    /**
     * @ComponentScan:注解能够在Spring中启用组件扫描,默认会扫描与配置类相同的包,即会扫描这个包以及这个包下的所有子包,查找带有@Component注解的类
     * 为何设置基础包:有一个原因会促使我们明确地设置基础包，那就是我们想要将配置类放在单独的包中，使其与其他的应用代码区分开
     * 使用String设置组件扫描的基础包:
     *      单个:basePackages = "com.dtrees.spring.bean"
     *      多个:basePackages = {"com.dtrees.spring.bean","com.dtrees.spring.bean2",...}
     * 使用Class来设置组件扫描的基础包(推荐):
     *      basePackageClasses = {},basePackageClasses属性所设置的数组中包含了类,这些类所在的包将会作为组件扫描的基础包
     */

    // 声明简单的bean并指定bean的id名称
    @Bean(name = "sgtPeppers")
    public CompactDisc getCompactDisc(){
        return new SgtPeppers();
    }
}
```

##### Java配置和xml配置混合

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:c="http://www.springframework.org/schema/c"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                           http://www.springframework.org/schema/beans/spring-beans-4.0.xsd
                           http://www.springframework.org/schema/context
                           http://www.springframework.org/schema/context/spring-context-4.0.xsd
                           http://www.springframework.org/schema/aop
                           http://www.springframework.org/schema/aop/spring-aop-4.0.xsd
                           http://www.springframework.org/schema/tx
                           http://www.springframework.org/schema/tx/spring-tx-4.0.xsd">

 <bean id="compactDisc" class="com.dtrees.spring.bean.SgtPeppers"></bean>
</beans>
```

##### Java配置

```java
//beans.xml在maven项目的resources目录
@Import(CDConfig.class)
@ImportResource("classpath:beans.xml")	
public class CDPlayerConfig {
	/**
     * @Import({}):将多个配置类进行组合一起
     * @ImportResource("classpath:beans.xml"):将spring配置的xml组合在一起
     */
@Bean(name = "cdPlayer")
public MediaPlayer getMediaPlayer(CompactDisc compactDisc){
        return new CDPlayer(compactDisc);
    }
}
```

##### spring装配的命名空间

```xml
## 开头省略...
<!-- 
c-命名空间来声明构造器参数
beans.xml中必须引入xmlns:c="http://www.springframework.org/schema/c"的声明
-->
<!--
        使用命名
        c:compactDisc-ref="sgtPeppers1"
        属性名称c:命名空间的前缀
        compactDisc:装配的构造器参数名
        -ref:这是一个命名的约定,它会告诉Spring,正在装配的是一个bean的引用,这个bean的名字是compactDisc,而不是字面量“compactDisc”
        sgtPeppers1:要注入的bean的ID名称
    -->
<bean id="cdPlayer2" class="com.dtrees.spring.bean.CDPlayer" c:compactDisc-ref="sgtPeppers1"></bean>
```
##### 使用属性的命名空间

```xml
 <!--
        使用属性命名空间（p-命名空间的属性）:
        p:title-ref="title-ref"
        属性的名字使用了“p:”前缀，表明我们所设置的是一个属性
        "title":接下来就是要注入的属性名。
        -ref:这会提示Spring要进行装配的是引用，而不是字面量
        title-ref:要注入的bean的ID名称
    -->
// 引入属性的命名空间xmlns:p="http://www.springframework.org/schema/p"

//设置字面值
<bean id="compactDisc" class="com.dtrees.spring.bean.SgtPeppers" p:title="xxx value"></bean>

// 设置引用
<bean id="compactDisc" class="com.dtrees.spring.bean.SgtPeppers" p:title-ref="title-ref"></bean>
```
**小结：c与p的命名空间，在装配bean引用与装配字面量的唯一区别在于是否带有“-ref”后缀，如果没 有“-ref”后缀的话，所装配的就是字面量**

##### bean的注入方式(这里只用JavaConfig注入方式，不用xml配置)

```java
@Component
public class CDPlayer implements MediaPlayer{

    private CompactDisc compactDisc;

    // 通过构造器自动装配
    @Autowired
    public CDPlayer(CompactDisc compactDisc){
        this.compactDisc = compactDisc;
    }

    // 通过设置器自动装配
    @Autowired
    public void setCompactDisc(CompactDisc compactDisc){
        this.compactDisc = compactDisc;
    }

    // 其他方法
    @Autowired
    public void insertCompactDisc(CompactDisc compactDisc){
        this.compactDisc = compactDisc;
    }

    public CompactDisc getCompactDisc(){
        return this.compactDisc;
    }
}
```