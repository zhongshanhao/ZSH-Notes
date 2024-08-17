## Spring

Spring是一个开源框架，主要核心有IOC（Inversion of Control）和AOP（Aspect Oriented Programming）。

使用Spring框架开发的优势：

- 方便解耦，简化开发：通过Spring提供的IOC容器，可以将对象间的依赖关系交由Spring进行控制，降低程序之间的耦合性。

- 使用Spring提供的AOP方便进行面向切面编程。

### IOC

IOC控制反转将创建对象的权利交给框架，通过这种方式，能够降低程序的耦合性。

它包括依赖注入和依赖查找。

如何理解IOC是怎么给程序解耦的？

程序的耦合性：程序之间的耦合性就像下图的齿轮间的组合，在java程序中，各个对象之间相互关联，随着对象的增多，这种对象之间的关联会越来越多，程序之间的耦合性就会越来越高，其中一个齿轮的变动将会导致牵一发而动全身的状况。

<img src="https://pic002.cnblogs.com/images/2011/230454/2011052709382686.jpg"></img>

IOC容器

IOC容器就是一个map集合，在程序运行的开始先将需要的对象加入到IOC容器中，在程序运行时，不需要对象A主动创建对象B，而是被动接收由IOC容器创建的对象B。通过IOC容器，能够将程序中对象之间的依赖关系降低，从而大大降低程序之间的耦合性。

<img src="https://pic002.cnblogs.com/images/2011/230454/2011052709391014.jpg">

为什么叫控制反转？
上述说道，在程序运行时，不需要对象A主动创建对象B，而是被动接收由IOC容器创建的对象B，这种获取对象方式从主动到被动的转变就叫控制反转。

IOC容器的技术原理

IOC中使用最基本的技术就是反射，反射就是根据给出的类名动态地生成对象。

可以把IOC容器的工作模式看做是工厂模式的升华，可以把IOC容器看作是一个工厂，这个工厂里要生产的对象都在配置文件中给出定义，然后利用编程语言的的反射编程，根据配置文件中给出的类名生成相应的对象。从实现来看，IOC是把以前在工厂方法里写死的对象生成代码，改变为由配置文件来定义，也就是把工厂和对象生成这两者独立分隔开来，目的就是提高灵活性和可维护性。

### AOP

是什么？

面向切面编程，简单的说就是将我们程序中重复的代码抽取出来，在需要执行的时候，通过动态代理技术，在不修改源码的基础上，对我们已有的方法进行增强。

AOP(Aspect-Oriented Programming:面向切面编程)能够将那些与业务无关，但是又在每个业务方法中都要调用的方法（例如事务处理、日志管理、权限控制等）封装起来，便于减少系统的重复代码，降低模块间的耦合度，并有利于未来的可拓展性和可维护性。

实现原理

如果要代理的对象

- 实现了某接口：使用JDK原生动态代理
- 没有实现：使用CGlib技术生成一个被代理对象的子类来作为代理

代理模式

就是为其他对象提供一个代理类来控制对某个对象的访问，代理类可以为被代理类预处理消息、过滤消息并在此之后将消息转发给被代理类，之后还能进行消息的后置处理，代理类本身不实现服务，只是通过调用被代理类中的方法来提供服务。

静态代理

动态代理

使用

- 记录日志
- 权限检查
- 事务管理

```java
@Component
@Aspect
public class AlphaAspect {

    //                   * 任何返回值                      //任何业务任何方法任何参数列表
    @Pointcut("execution(* com.nowcoder.community.service.*.*(..))")
    public void pointcut() {

    }

    // 连接点之前调用
    @Before("pointcut()")
    public void before() {
        System.out.println("before");
    }

    // 切入点之后执行
    @After("pointcut()")
    public void after() {
        System.out.println("after");
    }

    // 有了返回值之后才执行
    @AfterReturning("pointcut()")
    public void afterRetuning() {
        System.out.println("afterRetuning");
    }

    // 抛异常之后执行
    @AfterThrowing("pointcut()")
    public void afterThrowing() {
        System.out.println("afterThrowing");
    }

    // 前后都织入逻辑
    @Around("pointcut()")
    public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
        System.out.println("around before");
        Object obj = joinPoint.proceed();
        System.out.println("around after");
        return obj;
    }
}
```

常用注解：

```java 
@Component  //在类上注明这个注解，在程序开始时就能够被装配到ioc容器，在程序运行时就能被ioc容器所管理
@Controller("可以起别名，否者默认类名，首字母改小写")     // 标识请求层，功能同上
@Service           // 业务层，功能同Component
@Repository   // 数据库访问层，功能同Component
@Primary        // 两个具有相同名字的类，在类上加了该注解就能优先使用该类

// 加在被ioc容器所管理的类的方法上
@PostConstruct  // 在构造器之后调用被该注解修饰的方法
@PreDestroy			// 在销毁对象之前调用被该注解修饰的方法

// 被ioc容器所管理的对象默认是单例的
@Scope("prototype")   //加在类上，这时候每次容器创建出来的都是不同的对象，默认是singleton,即单例的。

// 如何使用ioc容器装配第三方bean
// 实现一个配置类
@Configuration // 表示这个类是一个配置类
@Bean //加在方法上，方法返回你想要用ioc容器管理的第三方bean

// 依赖注入
@Autowired // 加在成员变量之前，ioc容器将bean自动注入
@Qualifier("myBatis") //想要指定特定的bean的名字注入
```

spring的事务管理

编程式事务

声明式事务

底层是通过 AOP 实现的，可以在需要加事务保护的方法上加上@Transactional，并配置隔离级别和传播行为就可以。

spring有5种隔离级别，多了个默认的隔离级别

```java
public enum Isolation {
    DEFAULT(-1),
    READ_UNCOMMITTED(1),
    READ_COMMITTED(2),
    REPEATABLE_READ(4),
    SERIALIZABLE(8);
}
```

有7种传播行为

```java
public enum Propagation {
    REQUIRED(0),
    SUPPORTS(1),
    MANDATORY(2),
    REQUIRES_NEW(3),
    NOT_SUPPORTED(4),
    NEVER(5),
    NESTED(6);
}
```

举例：REQUIRED(0)事务传播行为

使用的最多的一个事务传播行为，我们平时经常使用的`@Transactional`注解默认使用就是这个事务传播行为。如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务。

```java
Class A {
    @Transactional(propagation=propagation.PROPAGATION_REQUIRED)
    public void aMethod {
        //do something
        B b = new B();
        b.bMethod();
    }
}

Class B {
    @Transactional(propagation=propagation.PROPAGATION_REQUIRED)
    public void bMethod {
       //do something
    }
}
```

Bean 的生命周期

作用域

参考链接

[1] https://www.cnblogs.com/wang-meng/p/5597490.html

[2] https://www.jianshu.com/p/9bcac608c714