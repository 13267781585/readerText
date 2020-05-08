## LinkedHashMap 源码解读
```java
public class LinkedHashMap<K,V>
    extends HashMap<K,V>
    implements Map<K,V>
```

1. LinkedHashMap 是基于 HashMap 实现的，只是LinkedHashMap 维护了一个节点列表，从头到尾按顺序连接了加入的节点。也就是说LinkedHashMap同时维护者 HashMap 中的节点数组和一个节点链表。

2. 迭代器遍历区别
LinkedHashMap 是有序的，按照元素加入的先后顺序遍历(链表); HashMap 是无序的，遍历是按照数组的下标从小到大，因为元素加入是根据 hash 值计算下标，所以是无序的。