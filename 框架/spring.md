# Spring 笔记

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