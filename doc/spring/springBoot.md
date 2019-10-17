# Spring 架构
![image](https://user-images.githubusercontent.com/12162133/54071753-2b504600-42ac-11e9-8831-6b211fd2c0bf.png)
# Spring Core
Spring 框架其中最重要的包含 IOC 和 AOP
## Spring IOC
IOC（Inversion of Control），即控制反转，是面向对象编程中的一种设计模式，在 Java 开发中，IOC 将你设计好的对象交给容器控制，而不是传统的在你的对象内部进行控制。那么如何理解 IOC 呢？如何理解”谁控制谁，控制什么，为何是反转（正转）“？
- 谁控制谁，控制什么：传统的程序设计中，我们直接在对象内部通过 new 进行创建对象，是程序主动去创建依赖对象；而 IOC 是有专门的容器来创建这些对象，即由 IOC 容器来控制对象的创建。
- 为何是反转（正转）：有反转就有正转，传统的程序设计中，是由我们自己在对象中主动控制去直接获取依赖对象，也就是正转；而反转则是由容器来帮买创建及注入依赖对象，因为由 IOC 容器帮助我们查找及注入依赖对象，对象只是被动的接受依赖对象，所以是反转。

有了 IOC 容器后，由容器进行注入组合对象，所以对象与对象之间是耦合的，这样有利于程序的解耦。

**DI**
DI（Dependency Injection），即依赖注入，是 IOC 的常见形式，即由容器动态的将某个依赖关系注入到组件之中。依赖注入的目的并非为软件系统带来更多功能，而是为了提升组件重用的频率。通过依赖注入机制，我们只需要通过简单的配置，而无需任何代码就可指定目标需要的资源，完成自身的业务逻辑。[IOC 和 DI 的区别？](https://www.martinfowler.com/articles/injection.html)
**BeanFactory**
BeanFactory 是 IOC 容器的实际代表者，通过读取配置文件中的配置元数据，一般是基于 xml 配置文件进行配置元数据，而且 Spring 与配置文件完全是解耦的，也可以使用其他任何可能的方式进行配置元数据，比如 xml、注解都可以。
**Bean**
Java Bean 是一种对于 Java 语言写成的可重用组件，为写成 JavaBean，类必须是具体和公开的，并且具有无参数的构造器，Java Bean 对外提供 getter/setter 方法来访问成员。Bean 是由 Spring 容器初始化、装配及管理的对象。

在 Spring 中，有两种 IOC 容器：BeanFactory 和 ApplicationContext
- BeanFactory：Spring 实例化、配置和管理对象的最基本接口
- ApplicationContext：BeanFactory 的子接口，它还扩展了其他一些接口，以支持更丰富的功能，如国际化、访问资源、事件机制、应用层上下文、AOP 支持等。

**ApplicationContext**
- ClassPathXmlApplicationContext：ApplicationContext 的实现，从 classpath 获取配置文件：
```
BeanFactory beanFactory = new ClassPathXmlApplicationContext("classpath.xml")
```
- FileSystemXmlApplicationContext：ApplicationContext 的实现，从文件系统获取配置文件：
```
BeanFactory beanFactory = new FileSystemXmlApplicationContext("fileSystemConfig.xml")
```
IOC 容器工作步骤
- 配置元数据：需要配置一些元数据来告诉 Spring，希望容器如何工作，具体来说，就是如何去初始化、配置、管理 JavaBean 对象。
- 实例化容器：由 IOC 容器解析配置的元数据，IOC 容器的 Bean Reader 读取并解析配置文件，根据定义生成 BeanDefinition 配置元数据对象，根据 BeanDefinition 进行实例化、配置及组装 Bean。
- 使用容器：由客户端实例化容器，获取需要 Bean。
```
元数据：基于 xml 、注解、Java 配置
使用注解方式还是 xml 方式见仁见智，注解的优点是大大减少了配置，但是缺点是不可避免产生了侵入式编程
启动注解的方式是
<context:annotation-config/>
```
依赖注入的方式主要有以下几种：构造器、Setter

Spring 注解
- @Required：作用于 setter 方法上，表示此属性是必须的
- @Autowired：field、setter、constructor 上，显示地声明依赖
- @Configuration：用于放在 class 上，其作用和 xml 配置文件相同，表示此 Bean 是一个 Spring 配置。
- @Component：用来声明一个 Spring 组件，加入到应用上下文中
- @Controller：用于放在 Controller 上
- @Service：用于放在一个服务类、执行业务逻辑、计算、调用内部 API 等
- @Repository：用于放在一个 DAO 上，用于访问数据库

IOC 控制反转，是一种思想，按照Java程序的设计，我们直接在对象内部通过 new 创建对象，是程序主动去创建对象，而IOC是专门一个容器来创建对象的，即IOC容器来控制对象的创建，传统程序是由我们自己在对象中主动去获取依赖对象，也就是正转，而反转则是由容器来帮忙创建及注入依赖对象。

容器中每一个Bean都有一个BeanDefinition实例，该实例负责保存Bean对象所有必要信息，包括Bean对象参数、方法、class类型等，容器启动时，会通过BeanDefinitionReader进行解析和分析，并将分析后的信息组装成BeanDefinition实例，注册到相应的BeanDefinitionRegistry上。经过这一个阶段，然后通过调用getBean()方法获取对应的Bean实例。

作用域：
在默认情况下，Spring应用以单例模式创建的
单例（singleton）：在整个应用中，只创建bean的一个实例。
原型（prototype）：每次注入或者通过spring应用上下文获取的时候，都会创建一个新的bean实例。
会话（session）：在web应用中，为每个会话创建一个bean实例。
请求（request）：在web应用中，为每个请求创建一个bean实例。

## Spring AOP
AOP（Aspect-Oriented Programing），即面向切面编程，在 AOP 中的基本单元是 Aspect（切面）。Aspect 是由 point cut 和 advice 组成，这里包含两个工作：
- 如何通过 point cut 和 advice 定位到特定的 join point 上
- 如何在 advice 中编写切面代码

使用 @Aspect 注解的类就是切面
**advice（增强）**
由 aspect 添加到指定的 join point（即满足 point cut 规则的 join point）的一段代码，会将 advice 模拟成一个拦截器，并且在 join point 上维护多个advice，进行层层拦截 。例如 HTTP 鉴权的实现。

**join point（连接点）**
在 Spring AOP 中，join point 总是方法执行的点，即方法连接点。

**point cut**
Advice 是和特定的 point cut 关联的，并且在 point cut 相匹配的 join point 中执行，**在 Spring 中，所有的方法都可以认为是 join point，但是我们并不希望所有的方法都添加 Advice，而 pointcut 的作用就是提供一组规则（使用 AspectJ pointcut expression language ）来匹配 joinpoint，给满足规则的 join point 添加 Advice。**

**Proxy**
AOP 的技术实现是通过 Proxy 实现的，Proxy 可以分为静态代理、动态代理、CGLIB 代理。
1. 静态代理
静态代理的实现比较简单，代理类通过实现与目标对象相同的接口，并在类中维护一个代理对象，通过构造器设置目标对象，赋值给代理对象，进而执行代理对象实现的接口方法，并实现前拦截、后拦截所需的业务。
```
public interface IPerson {

    public void speak();

}

public class Person implements IPerson{
    
    @Override
    public void speak() {
        System.out.println("say helloworld.");
    }
}

public class PersonProxy {

    private IPerson iPerson;

    public PersonProxy(IPerson iPerson) {
        this.iPerson = iPerson;
    }

    public void speak(){
        System.out.println("hi");
        iPerson.speak();
        System.out.println("goodbye");
    }

    public static void main(String[] args) {
        PersonProxy personProxy = new PersonProxy(new Person());
        personProxy.speak();
    }

}
```
2. 动态代理
动态代理的原理是利用 Java 反射在运行阶段动态生成任意类型的动态代理类，首先实现一个 InvocationHandler，方法调用会转发到该类的 invoke 方法，newProxyInstance 会返回一个实现了指定接口的代理对象，对该对象的所有方法调用都会转发给 InvocationHandler.invoke() 方法。
```
public class DynamicProxy implements InvocationHandler {

    private Object target;

    public DynamicProxy(Object target) {
        this.target = target;
    }

    public Object proxyInstance(){
        return Proxy.newProxyInstance(target.getClass().getClassLoader(), target.getClass().getInterfaces(), this);
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("transcation begin.");
        // 当代理对象调用真实对象的方法时，其会自动的跳转到代理对象关联的 handler 对象的 invoke 方法进行调用
        method.invoke(target, args);
        System.out.println("transcation end.");
        return null;
    }

    public static void main(String[] args) {
        DynamicProxy dynamicProxy = new DynamicProxy(new Person());
        IPerson iPerson = (IPerson)dynamicProxy.proxyInstance();
        iPerson.speak();
    }
}
```
3. CGLIB 动态代理
CGLIB（Code Generation Library），是一个基于 ASM 的字节码生成库，它允许我们在运行时对字节码进行修改和动态生成，CGLIB 通过继承方式实现代理。

### Spring Boot Starter
Starter 的理念就是 Starter 会把所有的依赖都包含进来，避免了开发者自己去引入依赖所带来的麻烦，Starter 的本质就是 synthesize，Starter 不同的实现是有所差异的，但是他们基本上会用到两个相同的内容：ConfigurationProperties、AutoConfiguration，因为Spring Boot 坚信”约定大于配置“。所以我们使用 ConfigurationProperties 来保存我们的配置，并且这些配置都有默认值，一般Spring Boot 的配置会在一个目录下 resources/application.properties 或者 yml 文件下。

![image](https://user-images.githubusercontent.com/12162133/66492204-9a15ac00-eae6-11e9-9d46-02776e34bcb8.png)

其实 Starter 的核心就是条件注解，@Conditional 当classpath下存在某一个class类时，某个配置才会生效。
想要创建一个starter，如下：
1. 创建一个maven项目
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-autoconfigure</artifactId>
    <version>2.1.4.RELEASE</version>
</dependency>
```
2. 创建一个配置类
```
@ConfigurationProperties(prefix = "javaboy")
public class HelloProperties {
    private static final String DEFAULT_NAME = "江南一点雨";
    private static final String DEFAULT_MSG = "牧码小子";
    private String name = DEFAULT_NAME;
    private String msg = DEFAULT_MSG;
    public String getName() {
        return name;
    }
    public void setName(String name) {
        this.name = name;
    }
    public String getMsg() {
        return msg;
    }
    public void setMsg(String msg) {
        this.msg = msg;
    }
}
```
然后在 application.propertie 配置中配置：
```
javaboy.name=***
javaboy.msg=***
```
3. 写业务类
```
public class HelloService {
    private String msg;
    private String name;
    public String sayHello() {
        return name + " say " + msg + " !";
    }
    public String getMsg() {
        return msg;
    }
    public void setMsg(String msg) {
        this.msg = msg;
    }
    public String getName() {
        return name;
    }
    public void setName(String name) {
        this.name = name;
    }
}
```
4. 写自动配置类
```
@Configuration
@EnableConfigurationProperties(HelloProperties.class)
@ConditionalOnClass(HelloService.class)
public class HelloServiceAutoConfiguration {
    @Autowired
    HelloProperties helloProperties;

    @Bean
    HelloService helloService() {
        HelloService helloService = new HelloService();
        helloService.setName(helloProperties.getName());
        helloService.setMsg(helloProperties.getMsg());
        return helloService;
    }
}
```
(1). 首先 @Configuration 注解表明这是一个配置类。
(2). @EnableConfigurationProperties 注解是使我们之前配置的 @ConfigurationProperties 生效，让配置的属性成功的进入 Bean 中。
(3). @ConditionalOnClass 表示当项目当前 classpath 下存在 HelloService 时，后面的配置才生效。
(4). 自动配置类中首先注入 HelloProperties ，这个实例中含有我们在 application.properties 中配置的相关数据。
(5). 提供一个 HelloService 的实例，将 HelloProperties 中的值注入进去。

做完这一个步骤后，我们的自动化配置类就完成了，接下来就需要一个spring. factories 文件，那么这个文件是干嘛的呢，我们来看下Spring Boot 的启动类配置：
```
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {
        @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
        @Filter(type = FilterType.CUSTOM,
                classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
}
```
其中有一个注解是@EnableAutoConfiguration，那么这个注解是干嘛的呢？@EnableAutoConfiguration 注解会自动导入一个名为AutoConfigurationImportSelector这个类，而这个类会读取一个名为 spring. factories 的文件，spring.factories 中则定义需要加载的自动化配置类，我们可以看到任意一个框架的starter，都能看到它有一个spring.factories。那么我们自定义的starter也需要加一个这样的文件：
```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=org.javaboy.mystarter.HelloServiceAutoConfiguration
```
如此自动化配置就完成了。



















