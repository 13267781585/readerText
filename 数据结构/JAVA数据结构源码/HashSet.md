## HashSet 源码解读

```java
public class HashSet<E>
    extends AbstractSet<E>
    implements Set<E>, Cloneable, java.io.Serializable
```

1. HashSet 是基于 HashMap 实现的，因为HashMap 是 键值对 集合，而HashSet 是 值集合，所以采用一个内部 Object变量 填充HashMap 元素的值。
```java
//内部用一个 HashMap 实现
private transient HashMap<E,Object> map;
//用 Object变量 填充 值
private static final Object PRESENT = new Object();

//加入的元素转化为键值对
public boolean add(E e) {
        return map.put(e, PRESENT)==null;
    }
```

