---
title: RPC
tag: 
- rpc
- network
categories: distributed
---

1. 模块功能介绍
2. 核心代码记录
<!--more-->

## 项目介绍
在学习cmu440的RPC课程时，对RPC这种通信的认知感觉非常抽象模糊，于是动手尝试实现一个简单的rpc加深理解。

## 主要模块
- consumer
- rpc-user-api
- rpc-user-core
- rpc-example
- rpc-main

在序列化方面使用的是JSON。

## consumer
该模块客户端核心类
### client的handler处理类
ClientHandler.java

```java
public class ClientHandler extends ChannelInboundHandlerAdapter {
       @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        ctx.channel().attr(AttributeKey.valueOf("attrTest")).set(msg);
        System.out.println("Client read!");
        String str = (String) msg;
        Response response = JSONObject.parseObject(msg.toString(),Response.class);
        DefaultFuture.recive(response);
        //ctx.close();
    }
    //...
}
```

```java

```

- InvokeProxy

### InvokeProxy
该类实现Spring的``BeanPostProcessor``,重写了``postProcessBeforeInitialization``方法，利用反射的机制，遍历容器里注册Bean的所有拦截带有@RemoteInvoke的filed，带有@RemoteInvoke的对象是服务类实例，通过field.set方法将Bean的域指向代理对象，当调用到远程服务的api时，会进行一个远程通信的过程，远程服务进行实际的操作，通过tcp接受结果。

这里会阻塞一个请求timeout时间，否则会请求超时提醒。在timeout阻塞的时间内，netty客户端收到响应后，唤醒线程。

#### 动态代理
动态代理是使用cglib类实现的,实现了client的发送请求，得到返回请求结果的功能。

- request
client发送的请求内容有：请求id、请求接口类+方法、参数对象
- enhancer增强
```java
                Enhancer enhancer = new Enhancer();
                enhancer.setInterfaces(new Class[]{field.getType()});
                enhancer.setCallback(new MethodInterceptor() {
                    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
                        ClientRequest request = new ClientRequest();
                        request.setCommand(methodClassMap.get(method).getName() + "." + method.getName());
                        request.setContent(objects[0]);
                        Response resp = TcpClient.send(request);
                        return resp;
                    }
                });
```

``Response resp = TcpClient.send(request);``创建netty客户端发送请求，netty客户端启动在TcpClient里的静态代码里。

### 相关api

- public void set(Object obj, Object value)

> @param obj the object whose field should be modified
  @param value the new value for the field of {@code obj}
     
```java
@Component
public class InvokeProxy implements BeanPostProcessor {
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        System.out.println(beanName);
        Field[] fields = bean.getClass().getDeclaredFields();
        for (Field field : fields) {
            /**
             *isAnnotationPresent
             *如果此元素上存在指定类型的注释，则返回true，否则返回false。
             *此方法主要是为了方便访问标记注释而设计的。
             */
            if (field.isAnnotationPresent(RemoteInvoke.class)) {
                final Map<Method, Class> methodClassMap = new HashMap<Method, Class>();
                putMethodClass(methodClassMap, field);
                field.setAccessible(true);
                //动态代理
                Enhancer enhancer = new Enhancer();
                enhancer.setInterfaces(new Class[]{field.getType()});
                enhancer.setCallback(new MethodInterceptor() {
                    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
                        ClientRequest request = new ClientRequest();
                        request.setCommand(methodClassMap.get(method).getName() + "." + method.getName());
                        request.setContent(objects[0]);
                        Response resp = TcpClient.send(request);
                        return resp;
                    }
                });
                try {
                    field.set(bean, enhancer.create());
                } catch (IllegalAccessException e) {
                    e.printStackTrace();
                }
            }
        }
        return bean;
    }
    //...
}
```

## rpc-user-api
该模块是client远程调用要使用的api接口，真正实现接口的方法在server。

例:
```java
public interface UserRemote {
     Object saveUser(User user);
}
```

## rpc-example
该模块是测试用例启动类
```java
@Configuration
@ComponentScan("com")
public class BasicController {
    public static void main(String[] args) {
        ApplicationContext applicationContext = new AnnotationConfigApplicationContext(BasicController.class);
        BasicService basicService = applicationContext.getBean(BasicService.class);
        basicService.testSaveUser();
    }
}
```
- service
```java
@Service
public class BasicService {
    @RemoteInvoke
    private UserRemote userRemote; //注入的时候，将这个field换成enhancer.create
    public void testSaveUser() {
        User user = new User();
        user.setId(1);
        user.setName("zhangsan");
        Object resp = userRemote.saveUser(user);
        System.out.println(JSONObject.toJSONString(resp));
    }
}
```

## rpc-core
该模块主要是以下两个功能:
- 服务启动
- 提供服务，接口的具体实现方法。

### 服务启动
```java
@Configuration
@ComponentScan("com")
public class SpringServer {
    public static void main(String[] args) {
        ApplicationContext context =
                new AnnotationConfigApplicationContext(SpringServer.class);
    }
}
```
NettyInital的静态代码是server启动模块，会在Spring容器启动被自动注入到容器里的时候调用。


## rpc-main
该模块主要功能：
- 服务启动核心
- 反射处理业务

服务启动后，Spring容器里的带有@Remote的bean，在初始化后，通过反射将这个类的方法注入``Map<String,BeanMethod> beanMap``key是某个类的方法字符串，和client请求的command对应， BeanMethod里有两个field:bean、Method。

### 处理业务
响应请求，运行本地服务，返回计算结果

```java
    //反射处理业务
    public Response process(ServerRequest request) {
        String command = request.getCommand();
        BeanMethod beanMethod = beanMap.get(command);//接口
        Response result = null;
        if(beanMethod == null){
            return null;
        }
        Object bean = beanMethod.getBean();//接口实现类
        Method m = beanMethod.getM(); //接口里的方法实例
        Class paramType = m.getParameterTypes()[0];//参数类型
        Object content = request.getContent();//参数名

        Object args = JSONObject.parseObject(JSONObject.toJSONString(content),paramType);

        try {
            //bean 是具体实现类
            result = (Response) m.invoke(bean,args);
            result.setId(request.getId());
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }
        
        return result;
    }
```
``m.invoke(bean,args)``反射执行某个bean的方法。