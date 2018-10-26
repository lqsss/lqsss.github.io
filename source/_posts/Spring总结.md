---
title: Spring总结
tag: summary
categories: Spring
---

## IOC原理
### 过程
1. 资源定位Resource，ApplicationContext的三个实现类找到applicationContetx.xml等配置文件
2. BeanDefinition载入
- 构造一个BeanFactory,也就是IOC容器
- 调用XML解析器得到document对象（XmlBeanDefinitionReader） 
- 按照Spring的规则解析BeanDefition
3. 向IOC容器注册这些BeanDefinition
在IOC容器内部其实是将第二个过程解析得到的BeanDefinition注入到一个HashMap容器中，IOC容器就是通过这个HashMap来维护这些BeanDefinition的。

![](http://op7scj9he.bkt.clouddn.com/5753761-b013d9a241816f43.png)
### lazy-init
>ApplicationContext实现的默认行为就是在启动时将所有singleton bean提前进行实例化。lazy-init设置只对scop属性为singleton的bean起作用

而设置了lazy-init = "true"
BeanDefinition只是通过Spring的规则解析XML文件的dom结构生成的，实际Bean的生成要等应用调用getBean的时候才会反射生成。

## BeanFactory 和 FactoryBean的区别？
- BeanFactory是个Factory，也就是IOC容器或对象工厂，在Spring中，所有的Bean都是由BeanFactory(也就是IOC容器)来进行管理的，提供了实例化对象和拿对象的功能。

- FactoryBean是一个接口，实现这个接口的Bean，可以由BeanFactory管理，它是一个能够生产或修饰对象的工厂Bean。IOC容器中获取FactoryBean对象，需要getBean(&加上FactoryBean的名字),否则会依旧内部实现返回工厂里生产的实例。

## BeanFactory和ApplicationContext的区别？
1. BeanFactory
是Spring里面最低层的接口，提供了最简单的容器的功能，只提供了实例化对象和拿对象的功能。在启动的时候不会去实例化Bean
2. ApplicationContext
扩展了一些新功能，对于Web方面，（同时加载多个配置文件，资源访问，以声明式方式启动并创建Spring容器）在启动的时候就把所有的Bean全部实例化了。它还可以为Bean配置lazy-init=true来让Bean延迟实例化

## Spring Bean 的生命周期
>一旦把一个Bean纳入Spring IOC容器之中，这个Bean的生命周期就会交由容器进行管理，一般担当管理角色的是BeanFactory或者ApplicationContext

**Bean的建立， 由BeanFactory读取Bean定义文件，并生成各个实例;**

Setter注入，执行Bean的属性依赖注入;

BeanNameAware的setBeanName(), 如果实现该接口，则执行其setBeanName方法;

BeanFactoryAware的setBeanFactory()，如果实现该接口，则执行其setBeanFactory方法;

BeanPostProcessor的processBeforeInitialization()，如果有关联的processor，则在Bean初始化之前都会执行这个实例的processBeforeInitialization()方法;

InitializingBean的afterPropertiesSet()，如果实现了该接口，则执行其afterPropertiesSet()方法;

**Bean定义文件中定义init-method;**

**BeanPostProcessors的processAfterInitialization()，如果有关联的processor，则在Bean初始化之前都会执行这个实例的processAfterInitialization()方法;**

**DisposableBean的destroy()，在容器关闭时，如果Bean类实现了该接口，则执行它的destroy()方法;**

**Bean定义文件中定义destroy-method，在容器关闭时，可以在Bean定义文件中使用“destory-method”定义的方法;**

## bean的范围，作用域
Bean的作用域（每个作用域都是在同一个Bean容器中）

1. singleton：单例，指一个Bean容器中只存在一份（默认）

2. prototype：每次请求（每次使用）创建新的实例，destory方式不生效

3. request：每次http请求创建一个实例且仅在当前request内生效（只能在web中使用）

4. session：同上，每次http请求创建一个实例，当前session内有效（只能在web中使用）

5. global session：基于portlet的web中有效（portlet定义了global session），如果是在单个web中，同session。（只能在web中使用）

## @Autowire
1. required = false 避免因为找不到bean 而抛出异常
2. 每个类只有一个构造器被标注（required = true）
在接口上注入，可能有多个实现类，@Qualifier("实现类")指定 

## Spring AOP
这种在运行时，动态地将代码切入到类的指定方法、指定位置上的编程思想就是面向切面的编程。

### AOP核心概念
1、横切关注点

对哪些方法进行拦截，拦截后怎么处理，这些关注点称之为横切关注点

2、切面（aspect）

类是对物体特征的抽象，切面就是对横切关注点的抽象,面向切面编程，就是指 对很多功能都有的重复的代码抽取，再在运行的时候往业务方法上动态植入“切面类代码”。

3、连接点（joinpoint）

被拦截到的点，因为Spring只支持方法类型的连接点，所以在Spring中连接点指的就是被拦截到的方法，实际上连接点还可以是字段或者构造器

4、切入点（pointcut）

切入点在AOP中的通知和切入点表达式关联，指定拦截哪些类的哪些方法, 给指定的类在运行的时候动态的植入切面类代码。

5、通知（advice）

所谓通知指的就是指拦截到连接点之后要执行的代码，通知分为前置、后置、异常、最终、环绕通知五类

6、目标对象

被一个或者多个切面所通知的对象。

7、织入（weave）

将切面应用到目标对象并导致代理对象创建的过程

8、引入（introduction）

在不修改代码的前提下，引入可以在运行期为类动态地添加一些方法或字段

9、AOP代理（AOP Proxy）

在Spring AOP中有两种代理方式，JDK动态代理和CGLIB代理。

## Schema配置
所有切面和通知器必须放在一个<aop:config>中，pointcut，advisor，aspect

<aop:aspect>配置切面类，切面的具体逻辑实现
<aop:pointcut>配置切入点，在哪些地方进行切入
<aop:before method="">前置通知（切面类里的方法）
<aop:after-returning>返回后通知
<aop:after-throwing>抛出异常通知
<aop:after> 目标方法之后执行（始终执行）
<aop:around>环绕通知
```java
	public Object around(ProceedingJoinPoint pjp) {
		Object obj = null;
		try {
			System.out.println("Aspect around 1.");
			obj = pjp.proceed();
			System.out.println("Aspect around 2.");
		} catch (Throwable e) {
			e.printStackTrace();
		}
		return obj;
	}
```

## 注解
@Aspect 需要和@Component注释一起，（扫描不到）。
@Pointcut 一个切入点通过一个普通的方法定义来提供 （@Pointcut("xxx.xxx")）
@Before("@Pointcut下的xxx方法")等

execution表示某个方法的执行
within来指定类型的

## Spring 是如何管理事务的，事务管理机制？
事务管理可以帮助我们保证数据的一致性

### 声明式事务和编程式事务
1. 编程式事务管理：Spring推荐使用TransactionTemplate，实际开发中使用声明式事务较多。

2. 声明式事务管理：将我们从复杂的事务处理中解脱出来，获取连接，关闭连接、事务提交、回滚、异常处理等这些操作都不用我们处理了，Spring都会帮我们处理。

声明式事务管理使用了AOP面向切面编程实现的，本质就是在目标方法执行前后进行拦截。在目标方法执行前加入或创建一个事务，在执行方法执行后，根据实际情况选择提交或是回滚事务。

### 如何管理的

### 事务回滚属性
1. 默认情况下只有未检查异常(RuntimeException和Error类型的异常)会导致事务回滚. 而受检查异常不会.
2. 事务的回滚规则可以通过@Transactional 注解的 rollbackFor 和 noRollbackFor 属性来定义，这两个属性被声明为 Class[] 类型的, 因此可以为这两个属性指定多个异常类。

### 事务传播行为
>当事务方法被另一个事务方法调用时, 必须指定事务应该如何传播.

外部事务A方法调用事务B方法。

REQUIRED：如果存在一个事务，则支持当前事务。如果没有事务则开启一个新的事务。（A有了，就不用创建；A没有，B就单独创建一个共用。在ServiceA.methodA或者在ServiceB.methodB内的任何地方出现异常，事务都会被回滚。）

SUPPORTS： 如果存在一个事务，支持当前事务。如果没有事务，则非事务的执行。但是对于事务同步的事务管理器，PROPAGATION_SUPPORTS与不使用事务有少许不同。

NOT_SUPPORTED：总是非事务地执行，并挂起任何存在的事务。

REQUIRESNEW：总是开启一个新的事务。如果一个事务已经存在，则将这个存在的事务挂起。（B事务执行时新起一个事务，A事务需要等待B事务完成。存在两个不同的事务，B失败 不会影响A的提交，A的失败 不会影响B的提交）

MANDATORY：如果已经存在一个事务，支持当前事务。如果没有一个活动的事务，则抛出异常。

NEVER：总是非事务地执行，如果存在一个活动事务，则抛出异常

NESTED：如果一个活动的事务存在，则运行在一个嵌套的事务中。如果没有活动事务，则按REQUIRED属性执行。


## jdk的动态代理
```java
public class LogProxy implements InvocationHandler {
    private Object targetObject;

    public Object newProxyInstance(Object targetObject) {
        this.targetObject = targetObject;
        //第三个参数表明这些被拦截的方法在被拦截时需要执行哪个InvocationHandler的invoke方法
        return Proxy.newProxyInstance(targetObject.getClass().getClassLoader(), targetObject.getClass().getInterfaces(), this);
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("invoke() call!");
        System.out.println();
        Object res = null;
        System.out.println("before realSubject!");
        res = method.invoke(targetObject, args);
        System.out.println("after realSubject!");
        return null;
    }
}
```


## cglib的动态代理
```java
public class LogProxy implements MethodInterceptor {

    /**定义一个拦截器。在调用目标方法时，CGLib会回调MethodInterceptor接口方法拦截，来实现你自己的代理逻辑，类似于JDK中的InvocationHandler接口。
     * 代理对象调用方法时，被此拦截
     * @param o 动态生成的代理对象
     * @param method 委托对象的方法
     * @param objects 委托对象的参数
     * @param methodProxy  CGlib方法代理对象的方法引用 
     * @return 调用方法返回值
     * @throws Throwable
     */
    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        Object result = null;
        System.out.println("before realSubject");
        //调用代理类实例上的proxy方法的父类方法(即委托类的方法)
        result = methodProxy.invokeSuper(o,objects);
        System.out.println("after realSubject");
        return result;
    }
}

public class TestRunner {
    public static void main(String[] args) {
        //Enhancer 这里Enhancer类是CGLib中的一个字节码增强器，它可以方便的对你想要处理的类进行扩展
        Enhancer enhancer = new Enhancer();
        //将委托类设置父类
        enhancer.setSuperclass(RealSubject.class);
        //设置回调拦截
        enhancer.setCallback(new LogProxy());
        //生成代理对象，需要转型
        Subject subject  = (RealSubject) enhancer.create();
        subject.otherFun();
    }
}
```

## jdk和cglib的不同
### 实现方式
1. java动态代理:代理类实现InvocationHandler接口的invoke方法，将代理类、接口、类加载器，通过Proxy.newProxyInstance()生成代理对象。
2. CGLIB：实现MethodInterceptor接口里的intercept方法，通过Enhancer类进行扩展。

### 原理
1. java动态代理是利用反射机制生成一个实现代理接口的匿名类，在调用具体方法前调用InvokeHandler来处理，只能对实现了接口的类生成代理
2. cglib动态代理是利用asm开源包，对代理对象类的class文件加载进来，通过修改其字节码生成子类来处理，因为是继承，所以该类或方法最好不要声明成final 。针对类实现代理

   