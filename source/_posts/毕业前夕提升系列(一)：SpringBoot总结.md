---
title: 毕业前夕提升系列(一)：SpringBoot总结
tag: summary
categories: SpringBoot
---

新的客服系统使用的是SpringBoot框架，于是前来探究一下。

<!-- more -->

## 学习内容

途径主要是《SpringBoot揭秘：快速构建微服务体系》、网上的一些blog，以及跟随着这些资料查阅源码。

主要是以下几个方向的探究：

*   SpringBoot的启动
*   spring-boot-stater的了解

其中也会有Spring原来知识的补充。

## SpringBoot的启动

```java
@SpringBootApplication
public class XxxApplication {

    public static void main(String[] args) {
        SpringApplication.run(XxxApplication.class, args);
    }
}
```

这里XxxApplication是SpringBoot应用启动的入口,这里涉及两点:

### @SpringBootApplication

该注解其实是一个复合注解，

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {
		@Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })

```

我们只用重点关注其中的三条即可

*   @SpringBootConfiguration
*   @EnableAutoConfiguration
*   @ComponentScan

1.  @SpringBootConfiguration实际上就是`@Configuration`，@Configuration是之前Spring就存在的，javaConfig形式的IOC容器的配置类使用的注解。
2.  @EnableAutoConfiguration也是一个复合注解

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
```

其中`@Import(AutoConfigurationImportSelector.class)`，@Import的作用相当于<import source="XXX.xml">将多个分开的容器配置合到一个配置里。



**@Import只是将javaConfig形式的IoC配置引入，如果项目中还存在残留的xml形式的IOC容器配置，就需要用上@ImportResource引入xml文件**

例如:

```java
@Configuration
@ImportResource("classpath:spring/application-context.xml")
public class Config {
}
```



现在回头谈`@Import(AutoConfigurationImportSelector.class)`，@EnableAutoConfiguration借助了AutoConfigurationImportSelector将springboot应用中所有**符合@Configuration配置都加载到当前SpringBoot创建并使用的IoC容器。**这里借助了类似java的SPI方案：SpringFacoriesLoader从classpath中搜寻META-INF/里的spring.factories配置文件根据key-value()反射实例化为标注了@Configuration的JavaConfig形式的IoC容器配置类，然后汇总为一个并加载到IoC容器。



3. @ComponentScan
   这也是一个老标签，自动扫描加载符合条件的组件或者bean定义加载到容器中。

**tips：@Configuration实际也是@Component，所以@Configuration也会被@ComponentScan扫描到。**

### SpringApplication.run(XxxApplication.class, args)

SpringApplication就是SpringBoot的启动方案，模板化地启动Spring应用，创建并初始容器，同时提供了一些扩展点，以便于我们对SpringBoot程序的启动和关闭过程进行扩展。


#### ApplicationContext

先让我们回顾一下Spring里的容器，其中ApplicationContext是我们经常使用的。

![容器有关类](https://blog-1257900554.cos.ap-beijing.myqcloud.com/applicationContext.jpg)



图中标注红框是我们重点关注的对象。

---


**ApplicationContext** 和 **BeanFactory** 都是用于加载 bean 的，但是 ApplicationContext 提供了更多的功能，包含了 BeanFactory 的所有功能。通常情况下，在大多数应用中都使用的是 ApplicationContext。

* * *

ApplicationContext的主要实现类是**ClassPathXmlApplicationContex**t和**FileSystemXmlApplicationContext**，前者默认从类路径加载配置文件，后者默认从文件系统中装载配置文件。

* * *

**ResourceLoader**：Spring里提供的一个实现不同的额Resource加载的接口，Spring容器的初始化需要该接口的实现。

* * *

**ConfigurableApplicationContext**：接口中定义了一些基本操作，比如设置上下文ID，设置父应用上下文，添加监听器和刷新容器相关的操作等。

* * *

#### 执行流程

##### 启动类初始化

调用run()方法后，实际上会new一个SpringApplication对象

```java
public static ConfigurableApplicationContext run(Class<?>[] primarySources,
		String[] args) {
	return new SpringApplication(primarySources).run(args);
}
```



构造函数如下：

```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    //通过classpath中，判断应用类型:REACTIVE NONE SERVLET
    setInitializers((Collection) getSpringFactoriesInstances(
        ApplicationContextInitializer.class));
    //实例化ApplicationContextInitializer
    setListeners((Collection) 
    //实例化ApplicationListener
    getSpringFactoriesInstances(ApplicationListener.class));
    this.mainApplicationClass = deduceMainApplicationClass();
}
```



`setInitializers`和`setListeners`都是利用的类似jdk的SPI查找服务机制，META-INF/spring.factories加载class并且实例化，分别是实例化key为ApplicationContextInitializer和ApplicationListener的类。`getSpringFactoriesInstances()`实际就是利用了**SpringFactoriesLoader(Spring实现的一种SPI机制)**加载和实例化类。

* * *

下面需要谈下上述加载的ApplicationContextInitializer、ApplicationListener

*   ApplicationContextInitializer
```java
public interface ApplicationContextInitializer<C extends ConfigurableApplicationContext> {

	/**
	 * Initialize the given application context.
	 * @param applicationContext the application to configure
	 */
	void initialize(C applicationContext);

}
```



> 在Spring上下文被刷新之前进行初始化的操作。典型地比如在Web应用中，注册Property Sources或者是激活Profiles。Property Sources比较好理解，就是配置文件。Profiles是Spring为了在不同环境下(如DEV，TEST，PRODUCTION等)，加载不同的配置项而抽象出来的一个实体。



*   ApplicationListener

```java
@FunctionalInterface
public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {

	/**
	 * Handle an application event.
	 * @param event the event to respond to
	 */
	void onApplicationEvent(E event);

}
```



如果在容器部署一个实现该接口的bean，那么每当在一个ApplicationEvent发布到 ApplicationContext时，这个bean就会收到通知。

##### 真正执行run()方法

```java
public ConfigurableApplicationContext run(String... args) {
        StopWatch stopWatch = new StopWatch();
        stopWatch.start();
        ConfigurableApplicationContext context = null;
        Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
        configureHeadlessProperty();

        SpringApplicationRunListeners listeners = getRunListeners(args);
        //通过SpringFactoriesLoader加载SpringApplicationRunListeners

        listeners.starting();
        //遍历调用SpringApplicationRunListener的starting()

        try {
          ApplicationArguments applicationArguments = new DefaultApplicationArguments(
            args);

          ConfigurableEnvironment environment = prepareEnvironment(listeners,
                                                                   applicationArguments);
          //创建并配置当前SpringBoot应用将要使用的环境(包括ProertySource和Profile)，并遍历调用SpringApplicationRunListener的environmentPrepared()
          configureIgnoreBeanInfo(environment);

          Banner printedBanner = printBanner(environment);
          // 准备Banner打印器 - 就是启动Spring Boot的时候打印在console上的ASCII艺术字体

          context = createApplicationContext();
          //根据是否设置applicationContextClass，以及初始化阶段的webApplicationType来创建。

          exceptionReporters = getSpringFactoriesInstances(
            SpringBootExceptionReporter.class,
            new Class[] { ConfigurableApplicationContext.class }, context);

          prepareContext(context, environment, listeners, applicationArguments,printedBanner);
          //Spring上下文前置处理:为上下文设置环境，配置Bean生成器以及资源加载器(如果它们非空),调用之前加载initialize()

          refreshContext(context);
          afterRefresh(context, applicationArguments);
          stopWatch.stop();
          if (this.logStartupInfo) {
            new StartupInfoLogger(this.mainApplicationClass)
              .logStarted(getApplicationLog(), stopWatch);
          }
          listeners.started(context);
          callRunners(context, applicationArguments);
        }
        catch (Throwable ex) {
          handleRunFailure(context, ex, exceptionReporters, listeners);
          throw new IllegalStateException(ex);
        }

        try {
          listeners.running(context);
        }
        catch (Throwable ex) {
          handleRunFailure(context, ex, exceptionReporters, null);
          throw new IllegalStateException(ex);
        }
        return context;
}
```



1.  加载SpringApplicationRunListeners，并调用它们的starting()方法

*   SpringApplicationRunListener

      **q:前面有加载一些ApplicationListener，跟这个有什么关系吗？**

```java
public class EventPublishingRunListener implements SpringApplicationRunListener, Ordered {

      private final SimpleApplicationEventMulticaster initialMulticaster;

      public EventPublishingRunListener(SpringApplication application, String[] args) {
        this.application = application;
        this.args = args;
        this.initialMulticaster = new SimpleApplicationEventMulticaster();
        for (ApplicationListener<?> listener : application.getListeners()) {
          this.initialMulticaster.addApplicationListener(listener);
        }
      }

      @Override
      public void starting() {
        this.initialMulticaster.multicastEvent(
            new ApplicationStartingEvent(this.application, this.args));
      }

      //...
}
```

```java
# Run Listeners
org.springframework.boot.SpringApplicationRunListener=\
org.springframework.boot.context.event.EventPublishingRunListener
```




EventPublishingRunListener是SpringApplicationRunListener一个默认实现，EventPublishingRunListener的构造函数就将之前SpringApplication设置的ApplicationListener加入到广播对象的列表里。

调用SpringApplicationRunListener的监听事件的方法时，相当于一个事件的中转，接调用 ApplicationListener。由SimpleApplicationEventMulticaster广播事件，通知ApplicationListeners集合里的监听器。

1.  创建并配置当前SpringBoot应用将要使用的环境(包括ProertySource和Profile)，并遍历调用SpringApplicationRunListener的environmentPrepared()

2.  createApplicationContext():创建并且初始化ApplicationContext。

3.  prepareContext():Spring上下文前置处理:为上下文设置环境，配置Bean生成器以及资源加载器(如果它们非空),调用之前加载的ApplicationContextInitializer的initialize()方法。

4.  refreshContext():刷新上下文，完成IoC容器最后的一道工序。调用SpringApplicationRunListener的started()，表明上下文已经刷新，但是CommandLineRunners、ApplicationRunner还未执行。

5.  执行CommandLineRunners、ApplicationRunner的run方法，这两个接口就是在容器启动完毕时执行的，run方法的参数不同。

## spring-boot-stater

提供了针对日常企业应用研发各种场景的spring-boot-stater自动配置依赖模块，”开箱即用“。

*   spring-boot-stater-web

      之前都是SpringMVC来开发web应用，现在SpringBoot提供了spring-boot-stater-web自动配置模块。

      我们只要引入该maven依赖，就可以得到一个可执行的web应用，mvn spring-boot:run就可以直接启动一个嵌入式tomcat服务请求的web应用。

*   spring-boot-stater-aop

    - 使用@Aspect注解将一个java类定义为切面类
    
    *   使用@Pointcut定义一个切入点，可以是一个规则表达式，比如下例中某个package下的所有函数，也可以是一个注解等。
*   根据需要在切入点不同位置的切入内容
    
    - 使用@Before在切入点开始处切入内容
    - 使用@After在切入点结尾处切入内容
    - 使用@AfterReturning在切入点return内容之后切入内容（可以用来对处理返回值做一些加工处理） 
    - 使用@Around在切入点前后切入内容，并自己控制何时执行切入点自身的内容
    - 使用@AfterThrowing用来处理当切入内容部分抛出异常之后的处理逻辑
          
          

