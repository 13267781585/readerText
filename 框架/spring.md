# Spring 笔记

1. 监听器接口ServletContextListener,在Web项目启动关闭执行方法
```java
public interface ServletContextListener extends EventListener {
    default void contextInitialized(ServletContextEvent sce) {
    }

    default void contextDestroyed(ServletContextEvent sce) {
    }
}

```

2. 三级缓存   
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
因为一级缓存需要和类外部交互，涉及到多线程，需要考虑线程安全，而二三级只是spring内部生成bean的中介缓存，不需要考虑多线程的因素。

* 单例bean生成过程在三级缓存中位置的变化?
1. 创建bean对象，调用bean的构造器，完成后放入三级缓存   
2. 进行属性依赖注入，获取对应依赖的bean，寻找的顺序:一级->二级->三级(allowEarlyReference=true)，若bean是从三级缓存获取的，从三级缓存删除，放入二级缓存(若需要被属性注入的bean还在三级缓存的话，说明出现了依赖注入，将三级缓存中的bean返回，并删除放入二级缓存)   
3. 依赖注入完成后从三级缓存放入一级缓存   

* 为什么三级缓存无法解决构造器循环注入依赖的问题?   
因为放入三级缓存的bean是在运行完类构造器后才放入的，若在执行构造器的时候就已经出现循环依赖，则无法解决。   




原博客地址:https://blog.csdn.net/m0_43448868/article/details/113578628