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

2. 