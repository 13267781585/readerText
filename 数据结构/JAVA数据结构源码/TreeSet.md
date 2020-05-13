## TreeSet 源码解读
```java
public class TreeSet<E> extends AbstractSet<E>
    implements NavigableSet<E>, Cloneable, java.io.Serializable
```
1. TreeSet 是基于 TreeMap 实现的，因为 TreeMap 是 键值对 集合，而 TreeSet 是 值集合，所以采用一个内部 Object变量 填充 TreeMap 元素的值。

```java
private transient NavigableMap<E,Object> m;

private static final Object PRESENT = new Object();

 public TreeSet() {
        this(new TreeMap<E,Object>());
    }

```
其中 NavigableMap 是一个接口，TreeMap 也实现了这个接口
```java
public interface NavigableMap<K,V> extends SortedMap<K,V>
```

2. TreeSet 是线程不安全的