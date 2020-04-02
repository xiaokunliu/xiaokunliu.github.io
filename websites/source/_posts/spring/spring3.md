---
title: Spring小结(3)
date: 2019-07-24 10:51:03
categories: 
- Spring技术小结
tags: 
- Spring技术小结
---

<!-- more -->

##### Spring面向切面AOP编程
###### 1. AOP编程原理
> 使用背景
* 需要在原有的应用功能模块增加额外的辅助功能,但并不是所有的应用功能模块所必需的
* 使用面向对象继承或者依赖技术增加辅助功能,会导致部分甚至所有的应用功能模块声明的类都会与辅助功能类在代码上存在耦合关系
* 使用代理委托方式,存在对委托对象进行复杂调用,容易导致原来的功能模块出现问题

> AOP切面作用
* 可以将上述增加辅助功能的问题转换为一个特殊类,集中处理,无须分散到各处的应用模块中
* 当前的特殊类只关注该辅助功能的实现,更为集中也便于维护

> AOP原理

**AOP可以比喻为我们要完成的一项任务工作, 一般做任何一项任务都具备目标,任务分解以及利用现有的工具资源来帮助任务推进,简言之可以将AOP对应的概念理解如下**:
* 目标: 最终要完成任务的目标 -- **AOP的通知Advice,设定的程序目标**
* 工具: 完成一项任务的目标我们需要借助一项工具来帮助我们更有效地完成  --  **AOP的连接点,提供信息/方法**
* 任务分解: 任务比较多的时候我们可以采用任务分解的方式来完成,将任务逐一划分为可执行的区域 -- **AOP的切点,类似任务分解的区域,也就是where**
* 最后,就是在做一项任务之前,我们都知道要完成这项任务的目标以及如何去划分任务,这个就是**AOP的切面,包含了通知和切点,定义任务的全部内容(准确地说,还包含工具提供的信息在里面)**

> AOP 术语
* 切面: 定义要完成一项任务的全部内容,即声明一个切面的类定义任务内容信息
* 通知: 定义要完成任务的目标,即在一个切面类下的目标方法
	* 前置通知（Before）：在目标方法被调用之前调用通知功能；
	* 后置通知（After）：在目标方法完成之后调用通知，此时不会关心方法的输出是什么；
	* 返回通知（After-returning）：在目标方法成功执行之后调用通知；
	* 异常通知（After-throwing）：在目标方法抛出异常后调用通知；
	* 环绕通知（Around）：通知包裹了被通知的方法，在被通知的方法调用之前和调用之后执行自定义的行为
* 切点: 定义要在什么地方执行,即在一个切面类下的目标方法中,告诉目标方法要在什么地方执行
* 连接点: 定义要完成任务的工具,即从程序角度而言,是利用应用程序中已有的功能为完成任务提供辅助,从我们定义的业务而言,是指我们原有的业务功能
* 引入: 在无须修改类的情况下, 向现有的业务功能的类添加方法或者属性, 让它们具有新的行为和状态
* 织入: 是把切面应用到目标对象并创建新的代理对象的过程.切面在指定的连接点被织入到目标对象中
	* 编译期：切面在目标类编译时被织入。这种方式需要特殊的编译器。AspectJ的织入编译器就是以这种方式织入切面的
	* 类加载期：切面在目标类加载到JVM时被织入。这种方式需要特殊的类加载器（ClassLoader），它可以在目标类被引入应用之前增强该目标类的字节码。AspectJ 5的加载时织入（load-timeweaving，LTW）就支持以这种方式织入切面
	* 运行期：切面在应用运行的某个时刻被织入。一般情况下，在织入切面时，AOP容器会为目标对象动态地创建一个代理对象。Spring AOP就是以这种方式织入切面

###### 2. AOP编程创建切面
> AOP 基于注解的方式  

 Spring AOP是基于方法去做增强,也就是在**类或对象的方法上做增强,需要区分AOP引入**
* 定义连接点
```java
/**
 * 定义连接点 -- 业务处理功能
 * 项目是基于SpringBoot
 */
 // AnnPerformance.java
public interface AnnPerformance {

    public void perform();
}

// 实现类AnnPerformanceImpl.java
@Component
public class AnnPerformanceImpl implements AnnPerformance {

    @Override
    public void perform() {
        System.out.println("AnnPerformanceImpl performance ... ");
    }
}
```
* 定义切面
```java
// AnnAudience.java
/**
 * 使用注解定义一个切面类，包含定义通知以及切点，使用注解的切面声明必须能够为通知类添加注解，为此要做到这点，必须有源码
 * 使用原则，基于注解的配置优于Java的配置，基于Java的配置优于xml的配置
 * @Aspect 使用该注解，Spring会动态为该Bean以及对应的目标bean创建代理对象，通过代理对象来实现目标，也就是通知
 */
@Aspect
public class AnnAudience {

    private static final Logger LOGGER = LoggerFactory.getLogger(AnnAudience.class);

	// 定义切点
    @Pointcut("execution(* com.xiaokunliu.study.springinaction.aop.annotation.AnnPerformance.perform(..))")
    public void perform(){
        // 方法内容并不重要，只是作为注解的附体
    }

	// 定义通知方法
    @Before("perform()")
    public void silenceCellPhones(){
        System.out.println("silencing cell phones ... ");
    }

    @After("perform()")
    public void takeSeats(){
        System.out.println("taking seats ...");
    }

    @AfterReturning("perform()")
    public void applause(){
        System.out.println("applause ...");
    }

    @AfterThrowing("perform()")
    public void demandRefund(){
        System.out.println("demand refund ...");
    }

    @Around("perform()")
    public void watchPerformance(ProceedingJoinPoint proceedingJoinPoint){
        try{
            LOGGER.info("silencing cell phones ....");
            LOGGER.info("taking seats ....");
            proceedingJoinPoint.proceed();
            LOGGER.info("demand refund ..... ");
        }catch (Throwable e){
            LOGGER.error("catching error from watchPerformance,messages=%s", e.getMessage());
            //demandRefund();
        }
    }
```
* 定义配置
```java
// AnnConcertConfig.java
@Configuration
@EnableAspectJAutoProxy    // 启动AspectJ自动代理
@ComponentScan(basePackages = {"com.xiaokunliu.study.springinaction.aop.annotation"})
public class AnnConcertConfig {

    @Bean
    public AnnAudience audience(){
        return new AnnAudience();
    }
}
```
* 编写单元测试
```java
// TestAnnAop.java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = AnnConcertConfig.class)
public class TestAnnAop {

    @Autowired
    private AnnPerformance annPerformance;

    @Test
    public void testAnnAop(){
        System.out.println(annPerformance.getClass());
        // 运行结果: class com.sun.proxy.$Proxy26
    }
}
```
* AOP 注解注入参数定义切面
```java
// 定义另一个切面来处理方法调用的统计
@Aspect
public class AopTrackCounter {

    /**
     * 处理通知中的参数
     * 目标： 需要记录每个磁道被播放的次数，此时需要将记录被播放的次数与播放本身分离出来
     */
    private Map<String, Integer> trackedCounters = new ConcurrentHashMap<>();

    @Pointcut("execution(* com.xiaokunliu.study.springinaction.aop.annotation.AopCompactDisc.play(java.lang.String)) && args(trackedNumber)")
    public void trackedPlay(String trackedNumber){}

    @Before("trackedPlay(trackedNumber)")
    public void countTracked(String trackerNumberKey){
        int counter = getCountTrack(trackerNumberKey);
        trackedCounters.put(trackerNumberKey, counter + 1);
    }
	
	// 获取根据trackerNumberKey统计数据
	// 每一个方法对应一个key
    public int getCountTrack(String trackerNumberKey){
        // 1.8 版本 Map.getOrDefault
        return trackedCounters.getOrDefault(trackerNumberKey, 0);
    }
}
```
* AOP注解注入参数定义连接点
```java
// AopCompactDisc.java
public interface AopCompactDisc {
    void play(String trackedNumber);
}

// AopBlankDisc.java
public class AopBlankDisc implements AopCompactDisc {
	 @Override
    public void play(String trackNumberKey) {
        System.out.println(trackNumberKey + " AOP AopBlankDisc play .... ");
    }
}
```
* 注入参数单元测试
```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = AnnConcertConfig.class)
public class TestAspectAop {

    @Autowired
    private AopCompactDisc aopCompactDisc;

    @Autowired
    private AopTrackCounter trackCounter;

    @Test
    public void testTrackCounter(){
        System.out.println(aopCompactDisc.getClass());
        String trackNumber = "com.xiaokunliu.study.springinaction.aop.TestAspectAop.testTrackCounter";
        aopCompactDisc.play(trackNumber);   // 3
        System.out.println(trackCounter.getCountTrack(trackNumber)); //4
		
		// 重复3,4代码
		// ....
    }
}	
```
* 运行结果
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190924131341971.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
* AOP 注解引入,即切面能够为**现有的对象或者类**增加新的方法
```java
// 比如现在需要为现有的类或者对象增加监控的方法
// 定义一个监控的接口以及对应的实现类处理监控 -- 注意只是处理监控相关的,和Performance没有关系
public interface AnnMonitor {

    // 将这个接口的调用应用到Performance实现中
    void performMonitor();
}

public class AnnDefaultMonitor implements AnnMonitor {

    @Override
    public void performMonitor() {
        // 处理监控的方法
        System.out.println("AnnDefaultMonitor performMonitor ... ");
    }
}
```
* 定义监控的切面
```java
@Aspect
public class AnnMonitorAspect {
    @DeclareParents(value = "com.xiaokunliu.study.springinaction.aop.annotation.AnnPerformance+", defaultImpl = AnnDefaultMonitor.class)
    private static AnnMonitor monitor;
}
```
* 单元测试
```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = AnnConcertConfig.class)
public class TestAnnAop {

    @Autowired
    private AnnPerformance annPerformance;

    @Autowired
    private AnnMonitor monitor;

    @Test
    public void testAnnAop(){
        System.out.println(annPerformance.getClass());
        System.out.println(monitor.getClass());
        System.out.println(monitor.getClass() == annPerformance.getClass());
    }
}
```
* 运行结果
![在这里插入图片描述](https://img-blog.csdnimg.cn/2019092414043748.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
* AOP引入可以用下图表示(引入Spring实战的图片)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190924140543762.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
> AOP 基于xml声明式
```xml
// java 代码省略,基本与上述一致,这里仅做xml相关的配置
// bean_aop.xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                           http://www.springframework.org/schema/beans/spring-beans-4.0.xsd
                           http://www.springframework.org/schema/context
                           http://www.springframework.org/schema/context/spring-context-4.0.xsd
                           http://www.springframework.org/schema/aop
                           http://www.springframework.org/schema/aop/spring-aop-4.0.xsd">


    <!--AOP 基于Xml配置, 自动扫描指定的包目录下的java -->
    <context:component-scan base-package="com.xiaokunliu.study.springinaction.aop.xml"/>

    <!-- 声明面向切面，与注解@EnableAspectJAutoProxy对应,启动AspectJ自动代理 -->
    <aop:aspectj-autoproxy />

     <!--
        声明需要的bean
     -->
    <bean id="aopPerformance" class="com.xiaokunliu.study.springinaction.aop.xml.AopPerformanceImpl"/>

    <!--声明切面为一个bean-->
    <bean id="audience" class="com.xiaokunliu.study.springinaction.aop.xml.AopAudience"/>


    <!--
        定义spring配置的切面
    -->
    <aop:config>
        <!-- 切面 -->
        <aop:aspect ref="audience">
            <!--  切点 -->
            <aop:pointcut id="performance" expression="execution(* com.xiaokunliu.study.springinaction.aop.xml.AopPerformance.perform(..))"/>
            <!-- 通知方法 -->
            <aop:before method="silenceCellPhones" pointcut-ref="performance"/>
            <aop:after method="takeSeats" pointcut-ref="performance"/>
            <aop:after-returning method="applause" pointcut-ref="performance"/>
            <aop:after-throwing method="demandRefund" pointcut-ref="performance"/>

            <!-- 声明环绕通知 -->
            <aop:around method="watchPerformance" pointcut-ref="performance"/>
        </aop:aspect>
    </aop:config>

    <!--
        AOP 传递通知参数
    -->
    <!-- 定义切面bean -->
    <bean id="aopBlankDisc" class="com.xiaokunliu.study.springinaction.aop.xml.AopBlankDisc"/>

    <bean id="tracker" class="com.xiaokunliu.study.springinaction.aop.xml.AopTrackCounter"/>

    <!--
           注意参数名称要方法名称对应上去
    -->
    <aop:config>
        <aop:aspect ref="tracker">
            <aop:pointcut id="tracker_performance"
                          expression="execution(* com.xiaokunliu.study.springinaction.aop.xml.AopCompactDisc.play(java.lang.String)) and args(trackedNumber)"/>
            <aop:before method="trackedPlay" pointcut-ref="tracker_performance" arg-names="trackedNumber"/>
        </aop:aspect>
    </aop:config>

    <!--
        AOP 引入新的功能
    -->
    <aop:config>
        <aop:aspect>
            <aop:declare-parents types-matching="com.xiaokunliu.study.springinaction.aop.xml.AopPerformance+"
                                 implement-interface="com.xiaokunliu.study.springinaction.aop.xml.AopMonitor"
                                 default-impl="com.xiaokunliu.study.springinaction.aop.xml.AopDefaultMonitor"/>
        </aop:aspect>
    </aop:config>
</beans>
```
* 基于SpringBoot定义配置
```java
@Configuration
@ImportResource(value = "classpath:spring/beans_aop.xml")
public class AopConfig {
}
```
* 单元测试
```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = AopConfig.class)
public class TestAspectXml {

    @Autowired
    private AopPerformance annPerformance;

    @Autowired
    private AopMonitor monitor;

    @Autowired
    private AopCompactDisc aopCompactDisc;

    @Autowired
    private AopTrackCounter trackCounter;

    @Test
    public void testAnnAop(){
    	// 运行结果如下:
    	//  class com.sun.proxy.$Proxy20
		//  class com.sun.proxy.$Proxy20
		// true
      	System.out.println(annPerformance.getClass());
        System.out.println(monitor.getClass());
        System.out.println(monitor.getClass() == annPerformance.getClass());
    }

    @Test
    public void testTrackCounter(){
    	// 执行结果如下:
    	/**
    	*	class com.sun.proxy.$Proxy21
			trackingNumber AOP AopBlankDisc play .... 
			1
			trackingNumber AOP AopBlankDisc play .... 
			2
			trackingNumber AOP AopBlankDisc play .... 
			3
			trackingNumber AOP AopBlankDisc play .... 
			4
			trackingNumber AOP AopBlankDisc play .... 
			5
			trackingNumber AOP AopBlankDisc play .... 
			6
			trackingNumber AOP AopBlankDisc play .... 
			7
			trackingNumber AOP AopBlankDisc play .... 
			8
    	*/
        System.out.println(aopCompactDisc.getClass());
        String trackNumber = "trackingNumber";
        aopCompactDisc.play(trackNumber);		// 2
        System.out.println(trackCounter.getCountTrack(trackNumber)); //3
		// 重复2,3代码 ... 
	}
}
```

