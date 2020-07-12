## Dubbo 一款高性能的java rpc框架



### 1.项目架构

#### 1.1 单体架构

优点：

>- 小项目开发快 成本低
>
>- 架构简单
>
>- 容易测试、部署

缺点：

>- 大项目模块耦合严重，维护沟通成本高
>- 新增业务困难(项目过大，涉及面广)
>- 核心业务和边缘业务容易相互影响(如果某一个服务出现问题容易导致其他正常的业务受性能之类的影响)



#### 1.2 垂直架构

根据业务把项目**垂直切割**成多个项目，因此这种架构称之为垂直架构。

优点：

>- 系统拆分实现了流量分担，解决了并发问题
>
>- 可以针对不同系统进行优化
>
>- 方便水平扩展，负载均衡，容错率提高
>
>- 系统间相互独立，互不影响，新的业务迭代时更加高效

缺点：

>- 服务系统之间**接口调用**硬编码（服务器改ip就得改配置）
>
>- 搭建集群之后，实现负载均衡比较复杂
>
>- 服务系统接口调用监控不到位 调用方式不统一（http、webservice等接口实现方式不统一）
>
>- 服务监控不到位
>
>- 数据库资源浪费，充斥慢查询，主从同步延迟大（子系统多了连接数增大）



#### 1.3 分布式架构（SOA、Service Oriented Architecture 面向服务的架构）

SOA全称为Service Oriented Architecture，即面向服务的架构 。它是在垂直划分的基础上,将每个项目拆分出多个具备松耦合的服务,一个服务通常以独立的形式存在于操作系统进程中。各个服务之间通过网络调用，**这使得构建在各种各样的系统中的服务可以 以一种统一和通用的方式进行交互。**我们在做了垂直划分以后，模块随之增多，系统之间的RPC逐渐增多，维护的成本也越来越高，一些通用的业务和模块重复的也**越来越多**，这个时候上面提到的**接口协议不统一、服务无法监控、服务的负载均衡等问题更加突出。**

**优点：**

>- 服务以接口为粒度，为开发者屏蔽远程调用底层细节 使用Dubbo 面向接口远程方法调用屏蔽了底层调用细节
>- 业务分层以后架构更加清晰 并且每个业务模块职责单一 扩展性更强
>
>- 数据隔离，权限回收，数据访问都通过接口 让系统更加稳定 
>- 安全服务应用本身无状态化 这里的无状态化指的是应用本身不做内存级缓存 而是把数据存入db
>
>- 服务责任易确定 每个服务可以确定责任人 这样更容易保证服务质量和稳定

缺点：

>- 粒度控制复杂 如果没有控制好服务的粒度 服务的模块就会越来越多 就会引发 超时 分布式事务等问题
>
>- 服务接口数量不宜控制 容易引发接口爆炸 所以服务接口建议以业务场景进行单位划分 并对相近的业务做抽象 防止接口爆炸
>
>- 版本升级兼容困难 尽量不要删除方法 字段 枚举类型的新增字段也可能不兼容
>
>- 调用链路长 服务质量不可监控 调用链路变长 下游抖动可能会影响到上游业务 最终形成连锁反应 服务质量不稳定 同时链路的变成使得服务质量的监控变得困难



#### 1.4微服务

微服务架构是一种将单个应用程序 作为一套小型服务开发的方法，每种应用程序都在其自己的进程中独立运行，并使用轻量级机制(通常是HTTP资源的API)进行通信。这些服务是围绕业务功能构建的，可以通过全自动部署机制进行独立部署。这些服务的集中化管理非常少，它们可以用不同的编程语言编写，并使用不同的数据存储技术。微服务是在SOA上做的升华 , 粒度更加细致，微服务架构强调的一个重点是“**业务需要彻底的组件化和服务化**”



### 2.dubbo实战

​		Dubbo 采用全 Spring 配置方式，透明化接入应用，对应用没有任何 API 侵入，只需用 Spring 加载 Dubbo 的配置即可，Dubbo 基于 [Spring 的 Schema 扩展](https://docs.spring.io/spring/docs/4.2.x/spring-framework-reference/html/xsd-configuration.html) 进行加载。如果不想使用 Spring 配置，可以通过 [API 的方式](http://dubbo.apache.org/zh-cn/docs/user/configuration/api.html) 进行调用。

#### 2.1快速入门

##### 2.1.1 服务端开发

- **定义服务接口**

```java
package org.apache.dubbo.demo;
//该接口需单独打包，在服务提供方和消费方共享
public interface DemoService {
    String sayHello(String name);
}
```

- **在服务提供方实现接口**

```java
package org.apache.dubbo.demo.provider;
import org.apache.dubbo.demo.DemoService;

//对服务消费方隐藏实现
public class DemoServiceImpl implements DemoService {
    public String sayHello(String name) {
        return "Hello " + name;
    }
}
```

- **在spring配置中暴露服务**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
    xsi:schemaLocation="http://www.springframework.org/schema/beans        
                        http://www.springframework.org/schema/beans/spring-beans-4.3.xsd        
                        http://dubbo.apache.org/schema/dubbo        
                        http://dubbo.apache.org/schema/dubbo/dubbo.xsd">
 
    <!-- 提供方应用信息，用于计算依赖关系 -->
    <dubbo:application name="hello-world-app"  />
 
    <!-- 使用multicast广播注册中心暴露服务地址 -->
    <dubbo:registry address="multicast://224.5.6.7:1234" />
 
    <!-- 用dubbo协议在20880端口暴露服务 -->
    <dubbo:protocol name="dubbo" port="20880" />
 
    <!-- 声明需要暴露的服务接口 -->
    <dubbo:service interface="org.apache.dubbo.demo.DemoService" ref="demoService" />
 
    <!-- 和本地bean一样实现服务 -->
    <bean id="demoService" class="org.apache.dubbo.demo.provider.DemoServiceImpl" />
</beans>
```

- **开启服务端**

```java
import org.springframework.context.support.ClassPathXmlApplicationContext;
 
public class Provider {
    public static void main(String[] args) throws Exception {
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(new String[] {"http://10.20.160.198/wiki/display/dubbo/provider.xml"});
        context.start();
        System.in.read(); // 按任意键退出
    }
}
```

##### 2.1.2 消费方开发

- **通过spring配置远程服务**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
    xsi:schemaLocation="http://www.springframework.org/schema/beans        
                        http://www.springframework.org/schema/beans/spring-beans-4.3.xsd        
                        http://dubbo.apache.org/schema/dubbo        
                        http://dubbo.apache.org/schema/dubbo/dubbo.xsd">
 
    <!-- 消费方应用名，用于计算依赖关系，不是匹配条件，不要与提供方一样 -->
    <dubbo:application name="consumer-of-helloworld-app"  />
 
    <!-- 使用multicast广播注册中心暴露发现服务地址 -->
    <dubbo:registry address="multicast://224.5.6.7:1234" />
 
    <!-- 生成远程服务代理，可以和本地bean一样使用demoService -->
    <dubbo:reference id="demoService" interface="org.apache.dubbo.demo.DemoService" />
</beans>
```

- **加载spring配置 并调用远程服务**

```java
import org.springframework.context.support.ClassPathXmlApplicationContext;
import org.apache.dubbo.demo.DemoService;
 
public class Consumer {
    public static void main(String[] args) throws Exception {
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(new String[] {"http://10.20.160.198/wiki/display/dubbo/consumer.xml"});
        context.start();
      	//也可以使用 IoC 注入
        DemoService demoService = (DemoService)context.getBean("demoService"); // 获取远程服务代理
        String hello = demoService.sayHello("world"); // 执行远程方法
        System.out.println( hello ); // 显示调用结果 out: hello world
    }
}
```

#### 2.2 dubbo-admin

**项目地址**：https://github.com/apache/dubbo-admin 

**作用**：服务管理 、 路由规则、动态配置、服务降级、访问控制、权重调整、负载均衡等管理功能



### 3.dubbo配置项说明

**官方地址**：http://dubbo.apache.org/zh-cn/docs/user/references/xml/introduction.html

#### 3.1 dubbo:application

 **对应类**：org.apache.dubbo.config.ApplicationConfig。保存了当前应用的信息

- **name** ：当前应用的名字
- **owner** ： 当前应用负责人
- **qosEnable**  ：是否启用qos（Quality of Service 用于动态控制服务，以及查询）
-  **qosPort**  ：启动qos绑定的端口，默认22222
- **qosAcceptForeignIp** ：是否允许远程访问。默认false

#### 3.2 dubbo:registry

**对应类**：org.apache.dubbo.config.RegistryConfig,代表该模块的**注册中心配置**，一个模块可以将自己的服务注册到多个中心。

- **id** ：当服务中provider或consume中存在多个注册中心时。需要增加该配置项 标识注册中心
- **address** ：当前注册中心的访问地址
- **protocol**：当前注册中心使用的协议 也可以写在address 如 zookeeper://xxxxx
- **Timeout** : 超时时间，按照服务与注册中心的网络情况配置

#### 3.3 dubbo:protocol

**对应类**：org.apache.dubbo.config.ProtocolConfig 指定服务在进行数据传输所使用的协议 

**与registry中的协议不同，registry中的协议是服务与注册中心之间的协议。这里的协议是provider和consume之间传输数据的协议**

- **id**：有可能在服务中需要有**多种传输协议**，所以需要一个id指定协议

- **name**： 指定协议名称。默认使用 dubbo

#### 3.4 dubbo:service

**对应类**：org.apache.dubbo.config.ServiceConfig  用于指定当前需要对外暴露的服务信息

- **interface** ：指定当前需要进行对外暴露的接口（服务内容）
- **ref**：具体实现对象的引用（具体实现），一般我们在生产级别都是使用Spring去进行Bean托管的，所以这里面一般也指的是Spring中的BeanId。 
- **version** ：**对外暴露的版本号**。不同的版本号，消费者在消费的时候**只会根据固定的版本号**进行消费。

#### 3.5 dubbo:reference

**对应类**：org.apache.dubbo.config.ReferenceConfig 消费端配置

- **id**： 指定该Bean在注册到Spring中的id。 
- **interface**：服务接口名
- **version** ：指定当前服务版本，与服务提供者的版本一致。
- **registry**： 指定所具体使用的注册中心地址。这里面也就是使用上面在 dubbo:registry 中所声明的id。

#### 3.6  dubbo:method

**对应类**：org.apache.dubbo.confifig.MethodConfifig, 用于在指定的 dubbo:service 或者 dubbo:reference 中的更具体一个层级。指定具体**方法级别**在进行RPC操作时候的配置

- **name** ：指定方法名称 用于对这个方法名称的RPC调用进行特殊配置。
- **async**：异步调用 默认false

```xml
<dubbo:reference interface="com.xxx.XxxService">
    <dubbo:method name="findXxx" timeout="3000" retries="2" />
</dubbo:reference>
```



#### 3.7 dubbo:service和dubbo:reference详解

- mock：用于在方法调用出现错误时，当做**服务降级**来统一对外返回结果。
- timeout：用于指定当前方法或者接口中所有方法的超时时间
-  check:用于在**启动时**，检查生产者是否有该服务，一般都会将这个值设置为false，**因为当出现循坏检查时，可能导致两个服务器都启动不了**
- retries：用于指定当前服务在执行时出现错误或者超时时的重试机制。
  - 注意提供者是否有**幂等**，否则可能出现数据一致性问题 如果是update才做的话有可能导致值一直被修改发生错误
  -  注意提供者是否有类似**缓存机制**，如出现大面积错误时，可能因为不停重试导致雪崩
- executes： 用于在**提供者**做配置，来确保最大的**并行度**。
  -  可能导致集群功能无法充分利用或者堵塞
  - 但是也可以启动部分对应用的保护功能
  - 可以不做配置，结合后面的熔断限流使用

### 4.dubbo 高级使用

#### 4.1 SPI

​		SPI 全称为 (Service Provider Interface) ，是**JDK内置的**一种服务提供发现机制。 目前有不少框架用它来做服务的扩展发现，简单来说，它就是一种**动态替换发现**的机制。使用SPI机制的优势是**实现解耦**，使得第三方服务模块的装配控制逻辑与调用者的业务代码分离。

##### **4.1.1 JAVA SPI**

>1. 在META-INF/services目录下创建一个以“**接口全限定名**”为命名的文件，内容为**实现类的全限定名**
>
>2. 接口实现类所在的jar包放在主程序的classpath中
>3. 主程序通过java.util.ServiceLoader动态装载实现模块，它通过扫描META-INF/services目录下的配置文件找到实现类的全限定名，把类加载到JVM； 
>4. SPI的实现类**必须携带一个无参构造方法** 用于实例化对象

##### 4.1.2 DUBBO SPI

​	dubbo中大量的使用了SPI来作为扩展点，通过实现同一接口的前提下，可以进行**定制自己的实现类**。比如比较常见的协议，负载均衡，都可以通过SPI的方式进行定制化，自己扩展

**创建步骤**：

- 导入dubbo依赖

- 创建接口 SayHelloService **在接口上使用dubbo 的@SPI注解** 可以带一个value参数 标识默认实现，如@SPI("dog");

  ```java
  @SPI("dog")
  public interface SayHelloService {
    	
      String sayHello();
    
      @Adaptive
      String sayHello(URL url);
  }
  ```

  

- 创建实现类 DogSayHelloServiceImpl,CatSayHelloServiceImpl

- 在resource 下创建MATE-INF/dubbo目录。并创建一个XXX.SayHelloService（全限定类名）文件，内容为

  ```properties
  cat=xxx.CatSayHelloServiceImpl
  dog=xxx.DogSayHelloServiceImpl
  ```

  ```java
  //cat say hello impl
  @Service
  public class CatSayHelloImpl implements SayHelloService {
  
      public String sayHello() {
          return "miao miao ~~";
      }
  
      public String sayHello(URL url) {
          return "miao url ~~";
      }
  }
  //dog say hello impl
  @Service
  public class DogSayHelloImpl implements SayHelloService {
  
      public String sayHello() {
          return "wang wang ~~";
      }
  
      public String sayHello(URL url) {
          return "wang url ~~";
      }
  }
  ```

  

- 调用，使用ExtensionLoader类调用对应的实现。

```java

public static void main(String[] args) throws IOException {
  //根据名称获取实现
  SayHelloService cat =ExtensionLoader.getExtensionLoader(SayHelloService.class).getExtension("cat");
  System.out.println(cat.sayHello());//miao
  //获取默认实现
  SayHelloService dog = ExtensionLoader.getExtensionLoader(SayHelloService.class).getDefaultExtension();
  System.out.println(dog.sayHello());//wang
}


```

##### 4.1.3 dubbo 实现自己的spi原因（java spi 缺点）

-  JDK 标准的 SPI 会一次性实例化扩展点所有实现，如果有扩展实现初始化很耗时，但如果没用上也加载，会很浪费资源 
- 如果有扩展点加载失败，则所有扩展点无法使用 
- 提供了对扩展点包装的功能(Adaptive)，并且还支持通过set的方式对其他的扩展点进行注入



##### 4.1.4 Dubbo SPI中的Adaptive功能

​		Dubbo中的Adaptive功能，主要解决的问题是**如何动态的选择具体的扩展点**，通过**getAdaptiveExtension** 统一对指定接口对应的所有扩展点进行封装，通过**URL的方式**对扩展点来进行**动态选择**。

**对于4.1.2中的例子调用**

```java
public class DubboAdaptiveMain { 
  public static void main(String[] args) { 
    //因为在这里只是临时测试，所以为了保证URL规范，前面的信息均为测试值即可，关键的点在于
    //hello.service 参数，这个参数的值指定的就是具体的实现方式。关于为什么叫
		//hello.service 是因为这个接口的名称，其中后面的大写部分被dubbo自动转码为 . 分割。
    URL url = URL.valueOf("test://localhost/hello?hello.service=dog"); 
    //通过 getAdaptiveExtension 来提供一个统一的类来对所有的扩展点提供支持(底层对所有的扩展点进行封装)
    //如果URL没有提供该参数，则该方法会使用默认在 SPI 注解中声明的实现
    final HelloService adaptiveExtension = ExtensionLoader.getExtensionLoader(HelloService.class)
      .getAdaptiveExtension(); 
    adaptiveExtension.sayHello(url); 
  } 
}
```

#### 4.2 Dubbo调用时拦截操作(Filter)

​		Dubbo的Filter机制，是专门为服务提供方和服务消费方调用过程进行拦截设计的，每次远程方法执行，该拦截都会被执行。这样就为开发者提供了非常方便的扩展性，比如为dubbo接口实现ip白名单功能、监控功能 、日志记录等。

**开发步骤**：

- 定义一个实现org.apache.dubbo.rpc.Filter 接口的类

- 使用 **org.apache.dubbo.common.extension.Activate** 接口进行对类进行注册 通过group 可以指定生产端 消费端

  ```
  @Activate(group = {CommonConstants.CONSUMER)
  ```

  ```java
  
  @Activate(group = {CommonConstants.CONSUMER,CommonConstants.PROVIDER})
  public class DubboInvokeFilter   implements Filter {
      @Override
      public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
          long   startTime  = System.currentTimeMillis();
          try {
              // 执行方法
              return  invoker.invoke(invocation);
          } finally {
              System.out.println("invoke time:"+(System.currentTimeMillis()-startTime) + "毫秒");
          }
      }
  }
  ```

  

- 在 META-INF.dubbo 中新建 org.apache.dubbo.rpc.Filter 文件，并将当前类的全名写入

  ```
  timerFilter=包名.过滤器的名字
  ```

  ```
  timeFilter=com.lagou.filter.DubboInvokeFilter
  ```

**一般类似于这样的功能都是单独开发依赖的，所以再使用方的项目中只需要引入依赖，在调用接口时，该方法便会自动拦截。**



#### 4.3 **负载均衡策略**

负载均衡（Load Balance）, 其实就是将请求分摊到多个操作单元上进行执行，从而共同完成工作任务。

负载均衡策略主要用于客户端存在多个提供者时进行选择某个提供者。在集群负载均衡时，Dubbo 提供了多种均衡策略（包括随机、轮询、最少活跃调用数、一致性Hash），**缺省为random随机调用**

##### 4.3.1 设置负载均衡策略

> 既可以在服务提供者一方配置，也可以在服务消费者一方配置

**消费端设置**：

```java
@Reference(check = false,loadbalance = "random")
```

**客户端方法级别**:

```xml
<dubbo:reference interface="...">
    <dubbo:method name="..." loadbalance="roundrobin"/>
</dubbo:reference>
```

**服务端设置**:

```java
@Service(loadbalance = "random") 
public class HelloServiceImpl implements HelloService { 
  public String sayHello(String name) { 
    return "hello " + name; 
  } 
}
```

**服务端方法级别**:

```xml
<dubbo:service interface="...">
    <dubbo:method name="..." loadbalance="roundrobin"/>
</dubbo:service>
```



##### 4.3.2 自定义负载均衡

负载均衡器在Dubbo中的SPI接口是 **org.apache.dubbo.rpc.cluster.LoadBalance** 

```java
@SPI(RandomLoadBalance.NAME)
public interface LoadBalance {
    @Adaptive("loadbalance")
    <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation) throws RpcException;
}
```

- 创建实现了**org.apache.dubbo.rpc.cluster.LoadBalance** 接口的实现类，重写其中的select()。

- 在META-INF/dubbo中配置

  ```properties
  name=自定义负载均衡器全限定类名
  ```

- 按照4.3.1中的设置方法设置自定义负载均衡



#### 4.4 异步调用

Dubbo不只提供了堵塞式的的同步调用，同时提供了**异步调用**的方式。这种方式主要应用于提供者接口响应耗时明显，消费者端可以利用调用接口的时间去做一些其他的接口调用,**利用 Future 模式来异步等待和获取结果即可**。这种方式可以大大的提升消费者端的利用率。 目前这种方式**可以通过XML的方式进行引入**。

- 在消费者端，配置异步调用 注意消费端**默认超时时间1000毫秒** 如果提供端耗时大于1000毫秒会出现超时

- ```xml
  <dubbo:reference id="helloService" interface="com.lagou.service.HelloService"> 
    <dubbo:method name="sayHello" async="true" /> 
  </dubbo:reference>
  ```

- 使服务端在接口调用是休眠一段时间，则方法在异步调用时的返回值是空，我们可以通过 RpcContext.getContext().getFuture() 来进行获取Future对象来进行后续的结果等待操作。

**特殊说明：**

> 需要特别说明的是，该方式的使用，**请确保dubbo的版本在2.5.4及以后的版本使用**。 原因在于在2.5.3及之前的版本使用的时候，会出现异步状态传递问题。比如我们的服务调用关系是 A -> B -> C , 这时候如果A向B发起了异步请求，在错误的版本时，B向C发起的请求也会连带的产生异步请求。这是因为在底层实现层面，他是通过 **RPCContext 中的attachment 实现的**。在A向B发起异步请求时，会在 attachment 中增加一个异步标示字段来表明异步等待结果。B在接受到A中的请求时，会通过该字段来判断是否是异步处理。但是由于值传递问题，B向 C发起时同样会将该值进行传递，导致C误以为需要异步结果，导致返回空。



#### 4.5 **线程池**

dubbo在使用时，都是通过创建真实的业务线程池进行操作的。dubbo中有两种线程池

- **fix**

>表示创建固定大小的线程池。也是Dubbo默认的使用方式，默认创建的执行线程数为200，并
>
>且是没有任何等待队列的。所以再极端的情况下可能会存在问题，**比如某个操作大量执行时，可能**
>
>**存在堵塞的情况**。后面也会讲相关的处理办法。

- **cache**

>创建非固定大小的线程池，**当线程不足时，会自动创建新的线程**。但是使用这种的时候需
>
>要注意，如果突然有高TPS的请求过来，方法没有及时完成，则会造成大量的线程创建，对系统的
>
>CPU和负载都是压力，**执行越多反而会拖慢整个系统**。

- **自定义线程池（具备报告使用率功能）**

**场景：**在真实的使用过程中可能会因为使用fix模式的线程池，导致具体某些业务场景因为线程池中的线程数量不足而产生错误，而很多业务研发是对这些无感知的。所以可以在创建线程池的时，通过某些手段对这个线程池进行监控，这样就可以进行及时的扩缩容机器或者告警。

**实现**：

1. 基于对 FixedThreadPool 中的实现做扩展出线程监控的部分，并且实现 Runnable 接口，从而可以后台一直执行消耗统计

```java
public class WatchingThreadPool extends FixedThreadPool implements Runnable {
  private static final double ALARM_PERCENT = 0.90;
  //使用hashmap存储使用中的线程
	private final Map<URL, ThreadPoolExecutor> THREAD_POOLS = new ConcurrentHashMap<>();
  //新建实例时创建一个统计任务每3s执行一次
  public WatchingThreadPool() { 
    Executors.newSingleThreadScheduledExecutor() .scheduleWithFixedDelay(this, 1,3, TimeUnit.SECONDS); 
  }
  
  @Override public Executor getExecutor(URL url) { 
    //从父类中创建线程池 
    final Executor executor = super.getExecutor(url); 
    //存储使用中的线程
    if (executor instanceof ThreadPoolExecutor) { 
      THREAD_POOLS.put(url, ((ThreadPoolExecutor) executor)); 
    }
    return executor; 
  }
  
  @Override 
  public void run() {
 	// 遍历线程池，如果超出指定的部分，进行操作，比如接入公司的告警系统或者短信平台
  for (Map.Entry<URL, ThreadPoolExecutor> entry : THREAD_POOLS.entrySet()) { 
    final URL url = entry.getKey(); 
    final ThreadPoolExecutor executor = entry.getValue(); 
    // 当前执行中的线程数 
    final int activeCount = executor.getActiveCount(); 
    // 总计线程数 f
    inal int poolSize = executor.getCorePoolSize(); 
    double used = (double)activeCount / poolSize; 
    final int usedNum = (int) (used * 100); 
    LOGGER.info("线程池执行状态:[{}/{}]:{}%", activeCount, poolSize, usedNum);
    //当操作警戒值时处理通知逻辑
    if (used >= ALARM_PERCENT) {
      LOGGER.error("超出警戒值！host:{}, 当前已使用量:{}%, URL:{}", url.getIp(), usedNum, url); 
    }
  	}
  }
  
}
```

2. SPI声明，创建文件 META-INF/dubbo/org.apache.dubbo.common.threadpool.ThreadPool

   内容为watching=包名.线程池名 

3. 在服务提供方项目中设置使用该线程池生成器 dubbo.provider.threadpool=watching



#### 4.6 **路由规则**

路由是决定**一次请求中需要发往目标机器的重要判断**，通过对其控制可以决定请求的目标机器。我们可以通过创建这样的规则**来决定一个请求**会交给哪些服务器去处理。

官方定义：http://dubbo.apache.org/zh-cn/docs/user/demos/routing-rule-deprecated.html

实现 **org.apache.dubbo.rpc.cluster.Router**

**场景**：当有很多服务器提供服务时，其中一台或多台需要重启，这时候不能再让客户端的请求发送到准备重启的机器上（灰度发布）

**实现思路**：

- 服务器在zk中创建一个临时节点，上线后创建。客户端根据zk中的在线服务器列表生成路由方案。只去查询存活的机器节点
- 或者
- 服务器在准备下线重启前，在zk中维护一个待重启机器ip。当客户端监听到这个节点数据发生变化时，将列表中的机器几点ip排除。待重启完成删除，对应节点的ip信息。晚上上线过程

#### 4.7 **服务动态降级**

**什么是服务降级**：

> 服务降级，当服务器压力剧增的情况下，根据当前业务情况及流量对一些服务有策略的降低服务级别，以释放服务器资源，**保证核心任务的正常运行。**

**为什么需要服务降级**：

> 这是防止分布式服务发生雪崩效应，什么是雪崩？就是蝴蝶效应，当一个请求发生超时，一直等待着服务响应，那么在高并发情况下，很多请求都是因为这样一直等着响应，直到服务资源耗尽产生宕机，而宕机之后会导致分布式其他服务调用该宕机的服务也会出现资源耗尽宕机，这样下去将导致整个分布式服务都瘫痪，这就是雪崩。

 **dubbo 服务降级实现方式**:

- **方式一**：在 dubbo 管理控制台（dubbo-admin）配置服务降级

  **mock=force:return+null** 表示消费方对该服务的方法调用**都直接**返回 null 值，不发起远程调用。用来屏蔽不重要服务不可用时对调用方的影响

  **mock=fail:return+null** 表示消费方对该服务的方法**调用在失败**后，再返回 null 值，不抛异常。用来容忍不重要服务不稳定时对调用方的影响

- **方式二**：指定返回简单值或者null

  ```xml
  <!--返回空-->
  <dubbo:reference id="xxService" 
                   check="false" 
                   interface="com.xx.XxService" 
                   timeout="3000" 
                   mock="return null" 
                   /> 
  <!--返回简单值-->
  <dubbo:reference id="xxService2" 
                   check="false" 
                   interface="com.xx.XxService2" 
                   timeout="3000" 
                   mock="return 1234" 
                   />
  ```

  如果是标注 则使用**@Reference(mock="return null") @Reference(mock="return 简单值")**，也支持 **@Reference(mock="force:return null")**

- **方式三**：使用java代码 动态写入配置中心

  ```java
  RegistryFactory registryFactory = ExtensionLoader.getExtensionLoader(RegistryFactory.class)
    .getAdaptiveExtension() ;
  Registry registry = registryFactory.getRegistry(URL.valueOf("zookeeper://IP:端 口")); registry.register(URL.valueOf("override://0.0.0.0/com.foo.BarService? category=configurators&dynamic=false&application=foo&mock=force:return+null"));
  ```

- **方式四**：整合整合 hystrix 



### 5.dubbo源码解析

