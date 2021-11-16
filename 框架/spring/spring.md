# Spring 笔记

### Spring模块
![3](.\image\3.jpg)
1. 核心容器（Core Container）

    Spring-Core：核心工具类，Spring其他模块大量使用Spring-Core。
    Spring-Beans：Spring定义Bean的支持。
    Spring-Context：运行时Spring容器。
    Spring-Context-Support：Spring对第三方包的集成支持。
    Spring-Expression：使用表达式语言在运行时查询和操作对象。

2. AOP

    Spring-AOP：基于代理的AOP支持。
    Spring-Aspects：基于AspectJ的AOP支持。

3. 消息（messaging）

    Spring-Messaging：对消息架构和协议的支持。

4. WEB

    Spring-Web： 提供基础的Web集成功能，再Web项目中提供Spring的容器。
    Spring-Webmvc：提供基于Servlet的Spring MVC。
    Spring-WebSocket：提供WebSocket功能。
    Spring-Webmvc-Protlet：提供Protlet的支持。

5. 数据访问/集成（Data Access/Integration）

    Spring-JDBC：提供以JDBC访问数据库的支持。
    Spring-TX：提供编程式是声明式事务的支持。
    Spring-ORM：提供对对象/关系映射技术的支持。
    Spring-OXM：提供对对象/XML映射技术的支持。
    Spring-JMS：提供对JMS的支持。

链接：https://www.jianshu.com/p/722e020150ef


### 监听器接口ServletContextListener,在Web项目启动关闭执行方法
```java
public interface ServletContextListener extends EventListener {
    default void contextInitialized(ServletContextEvent sce) {
    }

    default void contextDestroyed(ServletContextEvent sce) {
    }
}

```

### 三级缓存   
* spring只对单例的bean缓存，多例模式下每一个对象都是最新生成的。

* 解决的问题:
1. 属性注入的循环依赖(无法解决构造器循环依赖注入)   
2. 缓存bean，实现bean的复用   

```java
public class DefaultSingletonBeanRegistry extends SimpleAliasRegistry implements SingletonBeanRegistry {
	...
	private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256); //一级缓存
	private final Map<String, Object> earlySingletonObjects = new HashMap<>(16); // 二级缓存
	private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16); // 三级缓存
	...
	
	/** Names of beans that are currently in creation. */
	// 这个缓存也十分重要：它表示bean创建过程中都会在里面呆着~
	// 它在Bean开始创建时放值，创建完成时会将其移出~
	private final Set<String> singletonsCurrentlyInCreation = Collections.newSetFromMap(new ConcurrentHashMap<>(16));
 
	/** Names of beans that have already been created at least once. */
	// 当这个Bean被创建完成后，会标记为这个 注意：这里是set集合 不会重复
	// 至少被创建了一次的  都会放进这里~~~~
	private final Set<String> alreadyCreated = Collections.newSetFromMap(new ConcurrentHashMap<>(256));
}

```

* 为什么一级缓存是ConcurrentHashMap，二三级是HashMap?
因为一级缓存需要和类外部交互，涉及到多线程且并发可能比较大，使用ConcurrentHashMap来提高速度，而二三级只是spring内部生成bean的中介缓存，会涉及到线程安全，但并发量不高，使用synchronized去保证线程安全，速度更高。   

* 单例bean生成过程在三级缓存中位置的变化?
1. 创建bean对象，调用bean的构造器，完成后放入三级缓存   
2. 进行属性依赖注入，获取对应依赖的bean，寻找的顺序:一级->二级->三级(allowEarlyReference=true)，若bean是从三级缓存获取的，从三级缓存删除，放入二级缓存(若需要被属性注入的bean还在三级缓存的话，说明出现了依赖注入，将三级缓存中的bean返回，并删除放入二级缓存)   


3. 依赖注入完成后从三级缓存放入一级缓存   

* 为什么三级缓存无法解决构造器循环注入依赖的问题?   
因为放入三级缓存的bean是在运行完类构造器后才放入的，若在执行构造器的时候就已经出现循环依赖，则无法解决。   

4. 第二级缓存有什么意义？第一三级缓存不就能解决循环依赖的问题吗？     
在获取的bean没有被代理的情况下，使用第一三级的缓存可以解决循环依赖的问题。但是当获取的bean使用了代理时，会出现问题，因为三级缓存存放的是最原始的bean，当bean被获取出来且被代理时，会重新生成一个代理对象，每次获取都会重新生成一个新的对象，这不符合单例的特点，所以使用二级缓存来存放这些代理对象，当需要再次获取bean的时候直接从二级缓存获取，不需要再重新生成新的代理对象。   

```java
//获取对象的方法
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
	//一级缓存
    Object singletonObject = this.singletonObjects.get(beanName);
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        synchronized(this.singletonObjects) {
			//二级缓存
            singletonObject = this.earlySingletonObjects.get(beanName);
            if (singletonObject == null && allowEarlyReference) {
				//三级缓存
                ObjectFactory < ?>singletonFactory = this.singletonFactories.get(beanName);
                if (singletonFactory != null) {
					//生成对应的对象  如果对象被代理，这里每次会重新生成一个新的对象
                    singletonObject = singletonFactory.getObject();
                    this.earlySingletonObjects.put(beanName, singletonObject);
                    this.singletonFactories.remove(beanName);
                }
            }
        }
    }
    return singletonObject;
}

```
![1](.\image\1.jpg)

原博客地址:https://blog.csdn.net/m0_43448868/article/details/113578628



### 依赖注入的方式  
1. set方法注入   
2. 构造器注入  
3. 静态方法注入  
```xml
<!--(3)此处获取对象的方式是从工厂类中获取静态方法--> 
<bean name="staticFactoryDao" class="com.bless.springdemo.factory.DaoFactory" factory-method="getStaticFactoryDaoImpl"></bean> 
```

参考自https://www.cnblogs.com/java-class/p/4727775.html  

4. 实例方法注入  
```xml
<bean name="factoryDao" factory-bean="daoFactory" factory-method="getFactoryDaoImpl"></bean> 
```

5. 特殊的注入 lookup-method和replaced-method  
a. lookup-method  
lookup method注入是spring动态改变bean里方法的实现。方法执行返回的对象，使用spring内原有的这类对象替换，通过改变方法返回值来动态改变方法。原理:使用cglib重新生成子类，重写方法返回对象。  
```java
public abstract class CommandManager {

   public Object process(Object commandState) {
       // grab a new instance of the appropriate Command interface
       Command command = createCommand();
       // set the state on the (hopefully brand new) Command instance
       command.setState(commandState);
       return command.execute();
   }

   // okay... but where is the implementation of this method?
   protected abstract Command createCommand();
}
```
```xml
<bean id="command" class="fiona.apple.AsyncCommand" scope="prototype">
</bean>

<bean id="commandManager" class="fiona.apple.CommandManager">
 <lookup-method name="createCommand" bean="command"/>
</bean>
```
b. replaced-method   
replaced method注入是spring提供了一种替换bean方法的机制。新的逻辑需要实现org.springframework.beans.factory.support.MethodReplacer接口的方法。原理:cglib生成子类重写方法。   
```java
public class ReplacementComputeValue implements MethodReplacer {
    //新的逻辑
    public Object reimplement(Object o, Method m, Object[] args) throws Throwable {
        String input = (String) args[0];
        ...
        return ...;
    }
}
```
```xml
<bean id="myValueCalculator" class="x.y.z.MyValueCalculator">
    <!-- arbitrary method replacement -->
    <replaced-method name="computeValue" replacer="replacementComputeValue">
        <arg-type>String</arg-type>
    </replaced-method>
</bean>
 
<bean id="replacementComputeValue" class="a.b.c.ReplacementComputeValue"/>

```
c. 应用场景   
1. 提高编程的灵活性  
2. 可实现可拔插、耦合性小的有点   

参考作者：小陈阿飞
链接：https://www.jianshu.com/p/06f71d241866

### ApplicationContext和BeanFactory的区别
ApplicationContext和BeanFactory都是Spring容器的接口，负责bean的配置和实例化。   
1. ApplicationContext继承自BeanFactory，在BeanFactory的基础上扩展了更多的功能，例如：国际化的消息访问、资源访问、事件传递等，更能适应功能扩展的需求。   
2. 加载bean的时机不同，使用ApplicationContext会预加载bean，好处是速度快，缺点是浪费内存，而BeanFactory采用懒加载，好处是节约空间，缺点是使用时加载慢，多用于内存紧张的移动设备。   
3. BeanFactory和ApplicationContext都支持BeanPostProcessor、BeanFactoryPostProcessor，BeanFactory需要手动注册，而ApplicationContext则是自动注册   

### Aware接口的作用   
Spring的依赖注入最大的好处就是Bean并不依赖Spring容易，也就是说可以随意去更改Spring容易，耦合性很低，但是很多情况下Bean需要去感知Spring容器去获取一些必要的功能，所以Aware接口可以通过回调方法的方式实现这种需求。  
```java
BeanNameAware //获取beanName
BeanFactoryAware //获取BeanFactory对象
EnvironmentAware //获取Environment对象
ApplicationContextAware //获取ApplicationContext对象
```

### bean生命周期   
1. 主流程：类被加载成BeanDefinition存放->实例化->属性设置->初始化->销毁   
![2.jpg](.\image\2.jpg)
2. 实例化前后的扩展  
```java
public interface InstantiationAwareBeanPostProcessor extends BeanPostProcessor {
    @Nullable
    default Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
        return null;
    }

    default boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
        return true;
    }
}
```

2. 属性设置后各种Aware   
3. 初始化之前之后   
```java
public interface BeanPostProcessor {
    @Nullable
    default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }

    @Nullable
    default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }
}
```
4. 属性设置后调用afterPropertiesSet   
```java
public interface InitializingBean {
    void afterPropertiesSet() throws Exception;
}
```
5. 自定义初始化 init-method   
6. 注册销毁接口Destruction接口  
```java
public interface DestructionAwareBeanPostProcessor extends BeanPostProcessor {
    //这里实现销毁对象的逻辑
    void postProcessBeforeDestruction(Object bean, String beanName) throws BeansException;
    //判断是否需要处理这个对象的销毁
    default boolean requiresDestruction(Object bean) {
        return true;
    }
}
```
7. 销毁前调用 DisposableBean 接口   

8. 自定义销毁方法 destroy   

### AOP是在哪里处理的   
在bean初始化完成后，会调用BeanProcessor接口的postProcessAfterInitialization方法，其中有一个InfrastructureAdvisorAutoProxyCreator实现类是处理aop的，会使用jdk或者cglib生成代理类返回  
```java
public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
            throws BeansException {

        Object result = existingBean;
        for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {
            Object current = beanProcessor.postProcessAfterInitialization(result, beanName);
            if (current == null) {
                return result;
            }
            result = current;
        }
        return result;
    }
```

参考自 https://www.cnblogs.com/zcmzex/p/8822509.html

### Spring实例化bean的方式
1. 反射(在不需要动态操作字节码的条件下)   
2. Cglib生成(在需要动态生成类，重写方法等操作字节码的情况下)


### BeanDefinition属性   
```java
    AbstractBeanDefinition:
    private volatile Object beanClass;   //类类型
    private String scope;           //类的范围  单例  多例
    private boolean lazyInit;   //是否懒加载

    private int autowireMode;   //属性注入的模型
    public static final int AUTOWIRE_NO = 0;  //不注入
    public static final int AUTOWIRE_BY_NAME = 1;  //bean属性名
    public static final int AUTOWIRE_BY_TYPE = 2;  //bean类型
    public static final int AUTOWIRE_CONSTRUCTOR = 3;  //构造器

    private String[] dependsOn;       //对象需要依赖哪些bean，构造之前先实例化

    private boolean primary;    //实现的接口优先使用这个实现
    @Nullable
    private String factoryBeanName;  //工厂名字
    @Nullable
    private String factoryMethodName;   //工厂方法名
    @Nullable
    private ConstructorArgumentValues constructorArgumentValues;   //指定bean实例化调用哪一个构造器
    @Nullable
    private String initMethodName;    //初始化方法名
    @Nullable
    private String destroyMethodName;  //销毁方法名

```
### Springmvc的流程   
![4.jpg](.\image\4.jpg)


1. 用户请求被Spring 前端控制Servelt DispatcherServlet捕获；

2. DispatcherServlet对请求URL进行解析，根据URI，调用HandlerMapping获得Handler对象以及Handler对象对应的拦截器，最后以HandlerExecutionChain引用链的形式返回；

3. DispatcherServlet 根据获得的Handler，选择一个合适的HandlerAdapter。

4. 提取Request中的模型数据，填充Handler入参，开始执行Handler（Controller)。 在填充Handler的入参过程中，根据你的配置，Spring将帮你做一些额外的工作：   
    a. HttpMessageConveter： 将请求消息（如Json、xml等数据）转换成一个对象，将对象转换为指定的响应信息；   
    b. 数据转换：对请求消息进行数据转换。如String转换成Integer、Double等；   
    c. 数据根式化：对请求消息进行数据格式化。 如将字符串转换成格式化数字或格式化日期等；   

5. Handler执行完成后，向DispatcherServlet 返回一个ModelAndView对象；
根据返回的ModelAndView，选择一个适合的ViewResolver；

6. ViewResolver 结合Model和View，来渲染视图；

7. 将渲染结果返回给客户端。

转载自：https://blog.csdn.net/weixin_43258908/article/details/88595159


### Spring用到哪些设计模式
1. 单例->ioc容器   
2. 代理模式->aop   
3. 适配者模式
* aop  
* spring MVC中的适配器模式

在Spring MVC中，DispatcherServlet 根据请求信息调用 HandlerMapping，解析请求对应的 Handler。解析到对应的 Handler（也就是我们平常说的 Controller 控制器）后，开始由HandlerAdapter 适配器处理。HandlerAdapter 作为期望接口，具体的适配器实现类用于对目标类进行适配，Controller 作为需要适配的类。   
因为Controller的种类有多种，如果不适用适配者模式去处理，需要对每一个种类的Controller都作判断，当需要新增新的Controller时需要去修改以前的代码，违反了开闭原则。
```java
if(mappedHandler.getHandler() instanceof MultiActionController){  
   ((MultiActionController)mappedHandler.getHandler()).xxx  
}else if(mappedHandler.getHandler() instanceof XXX){  
    ...  
}else if(...){  
   ...  
}  
```
4. 装饰者模式
5. 工厂模式
6. 模板模式
7. 策略模式 
转载自：https://www.cnblogs.com/kyoner/p/10949246.html


### SpringAop和AspectJ的区别
  Aop             Aspectj   
1. 运行时增强  编译时增强   
2. 基于代理实现  基于字节码操作   
3. 直接使用  导入jar包
4. 速度慢、功能少  速度快、功能齐全  
5. 只能基于方法织入   对字段、方法、构造函数等 


### bean是线程安全的吗？
* Springioc并没有对bean的线程安全做什么处理，所以默认条件下，bean是单例的，是线程不安全的，例如Controller、Server这些bean若没有涉及到数据的存储共享，只是方法的调用是没有线程问题的，因为方法的调用形成的栈帧存放在虚拟机栈上是线程私有的；若涉及到数据的存储共享，则有线程安全问题。

* 解决方法
1. 使用多例，每次请求重新生成一个bean(不是100%的线程安全，对static变量除外)
2. 使用ThreadLoccal去处理



### Spring事务内部类调用不起效 
* 原因是使用jdk代理模式，jdk代理模式是通过生成一个代理对象，类内部带有被代理对象的引用，在被代理方法前后加入逻辑后利用引用调用方法，所以调用的是没有加入事务逻辑的方法，所以不起作用。
* 使用cglib代理模式不会出现这样的问题，因为cglib是通过继承被代理类，重写方法去实现的，所以调用的是子类的已经加入了事务逻辑的方法。
* 解决方法：
1. 重新写一个方法，从外部调用  
2. 使用cglib代理模式 

### Springboot相对于Spring的优劣势
优势：
1. 简化配置，去除了许多xml配置文件，配置简单  
2. 简化依赖，starter  
3. 简化部署，内置了tomcat等容器
4. 扩展了很多第三方的插件，例如redis、rabbitmq等

劣势：
1. 排查问题困难  
2. 版本迭代变化大

### Springboot自动配置
自动配置是指根据项目中用到的组件去做默认的配置(redis、aop、rabbitmq等)。这个过程是Spring框架自动进行的。   

* 原理  
Springboot的主函数注解@SpringbootApplication包含了@EnableAutoConfiguration注解，@EnableAutoConfiguration引入了一个AutoConfigurationImportSelector类去做自动配置的工作，改类会去扫描含有 META-INF/spring.facotories文件的jar包，文件采用键值对的方式记录了自动配置的相关类。会实例化这些类去做一些配置的工作。

```java
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\
```
* 自动配置体现在哪里？ 
Springboot定义了一组基于@Conditional的注解，可以根据不同的条件判断bean是否需要被实例化去做对应的配置。
```java
@Configuration：这个配置就不用多做解释了，我们一直在使用
@EnableConfigurationProperties：这是一个开启使用配置参数的注解，value值就是我们配置实体参数映射的ClassType，将配置实体作为配置来源。
以下为SpringBoot内置条件注解：
@ConditionalOnBean：当SpringIoc容器内存在指定Bean的条件
@ConditionalOnClass：当SpringIoc容器内存在指定Class的条件
@ConditionalOnExpression：基于SpEL表达式作为判断条件
@ConditionalOnJava：基于JVM版本作为判断条件
@ConditionalOnJndi：在JNDI存在时查找指定的位置
@ConditionalOnMissingBean：当SpringIoc容器内不存在指定Bean的条件
@ConditionalOnMissingClass：当SpringIoc容器内不存在指定Class的条件
@ConditionalOnNotWebApplication：当前项目不是Web项目的条件
@ConditionalOnProperty：指定的属性是否有指定的值
@ConditionalOnResource：类路径是否有指定的值
@ConditionalOnSingleCandidate：当指定Bean在SpringIoc容器内只有一个，或者虽然有多个但是指定首选的Bean
@ConditionalOnWebApplication：当前项目是Web项目的条件

```
原文转自：
https://www.jianshu.com/p/5901da52ca09








· Java 的IO

· 三种IO 的特点

· 最了解那一种IO？讲一下FileInputStream/FileOutputStream

· 怎么文件的读写？具体过程

· 序列化和反序列化

· 如何优化可以提高文件的读写速度

· 封装成Buffer 可以提升速度的原因

· 文件IO 的时候有遇到过爆内存的情况吗？怎么监控？

    2.2 Mybatis防止sql注入
    2.3 Mybatis特性 分页 有没有用过插件 