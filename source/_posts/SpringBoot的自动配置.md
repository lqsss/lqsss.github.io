---
title: SpringBoot的自动配置
tag: 
- autoconfig
categories: SpringBoot
---



自动配置是SpringBoot的一大亮点，在应用中无时无刻不在使用。

<!--more-->

## 自动配置原理

1.  SpringBoot启动的时候，开启了自动配置功能@EnableAutoConfiguration

2.  @EnableAutoConfiguration将自动配置类加载到容器里


```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
```



该注解是一个复合注解，重点在通过@Import引入的AutoConfigurationImportSelector

ApplicationContext的`refresh()`里的`invokeBeanFactoryPostProcessors(beanFactory);`这行代码里会解析处理各种注解，包括：@Configuration、@ComponentScan、@Import、@PropertySource、@ImportResource、@Bean，当处理**@Import**注解的时候，会调用`AutoConfigurationImportSelector`的`selectImports()`完成自动配置。

我们看一下`selectImports()`-&gt;`getAutoConfigurationEntry()`-&gt;`getCandidateConfigurations()` 获取候选的配置

```java
List<String> configurations = getCandidateConfigurations(annotationMetadata,attributes);
```

```java
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata,
		AnnotationAttributes attributes) {
	List<String> configurations = SpringFactoriesLoader.loadFactoryNames(
			getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader());
	Assert.notEmpty(configurations,
			"No auto configuration classes found in META-INF/spring.factories. If you "
					+ "are using a custom packaging, make sure that file is correct.");
	return configurations;
}
```



`SpringFactoriesLoader.loadFactoryNames()`：Spring Boot在启动的时候从类路径下的META-INF/spring.factories变成一个properties对象，然后从中获取EnableAutoConfiguration指定的值，将这些值作为自动配置类导入到容器中，自动配置类就生效，帮我们进行自动配置工作。



1.  每一个自动配置类进行配置功能(以HttpEncodingAutoConfiguration为例)
```java
@Configuration
@EnableConfigurationProperties(HttpProperties.class)
//开启ConfigurationProperties功能,将配置文件数值绑定，将HttpProperties加入到容器中
@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
//根据制定的条件，整个配置类里的配置就会生效；判断当前是否Web应用，如果是，则当前配置类生效。
@ConditionalOnClass(CharacterEncodingFilter.class)
//判断当前项目有没有这个类；CharacterEncodingFilter是SpringMVC中进行乱码解决的过滤器
@ConditionalOnProperty(prefix = "spring.http.encoding", value = "enabled", matchIfMissing = true)
//判断spring.http.encoding.enabled = true；如果不存在也成立

//以上条件都成立，则配置类生效
public class HttpEncodingAutoConfiguration {
  
  private final HttpProperties.Encoding properties;

 	//只有一个有参构造函数，从容器里拿properties进行
  public HttpEncodingAutoConfiguration(HttpProperties properties) {
		this.properties = properties.getEncoding();
	}
  
  @Bean 
  // 加入IoC容器中，函数里若是有参数的话，则相当于constructor-arg=”${参数}“...
  // 函数里的setXXX(xxx)，类似xml文件<property name="XXX" ref="xxx">
	@ConditionalOnMissingBean
	public CharacterEncodingFilter characterEncodingFilter() {
		CharacterEncodingFilter filter = new OrderedCharacterEncodingFilter();
		filter.setEncoding(this.properties.getCharset().name());
		filter.setForceRequestEncoding(this.properties.shouldForce(Type.REQUEST));
		filter.setForceResponseEncoding(this.properties.shouldForce(Type.RESPONSE));
		return filter;
	}
}
```

```java
@ConfigurationProperties(prefix = "spring.http")
public class HttpProperties {
  //...
}
```



## 查看项目配置情况

自动配置类必须在一定的条件下才能生效。

可以在项目配置文件中，设置`debug=true`，启动SpringBoot时，可以查看自动配置的报告，便可以知道哪些match成功(即启动自动配置成功)。
