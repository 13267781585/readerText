## LinkedHashMap 源码解读
```java
public class LinkedHashMap<K,V>
    extends HashMap<K,V>
    implements Map<K,V>
```

1. LinkedHashMap 是基于 HashMap 实现的，只是LinkedHashMap 维护了一个节点列表，从头到尾按顺序连接了加入的节点。也就是说LinkedHashMap同时维护者 HashMap 中的节点数组和一个节点链表。

2. 迭代器遍历区别
LinkedHashMap 是有序的，按照元素加入的先后顺序遍历(链表); HashMap 是无序的，遍历是按照数组的下标从小到大，因为元素加入是根据 hash 值计算下标，所以是无序的。

3. LinkedHashMap 有插入顺序和使用顺序。
插入顺序：类内部索引会按插入时的顺序连接
使用顺序：类内成员变量accessOrder 默认为 false，当我们在创建LinkedHashMap 实例传入 true是，调用 get() 函数后会将返回的节点插入到最后，例如原来的序列为 1234 ，使用了 2 之后序列变为 1342.

```java
final boolean accessOrder;         //是否开启使用顺序规则

public V get(Object key) {
        Node<K,V> e;
        if ((e = getNode(hash(key), key)) == null)         //返回需要的节点
            return null;
        if (accessOrder)
            afterNodeAccess(e);           //判断是否把节点移动到结尾
        return e.value;
    }
```

```java
//HashMap 节点打印是随机的

  @Test
    public void test8(){
        HashMap<String,String> map = new HashMap<>();
        map.put("a1","a");
        map.put("a2","b");
        map.put("b1","c");
        map.put("b2","d");
        map.put("c1","e");
        for (String s : map.values()) System.out.println(s);

    }

a
d
b
e
c

//LinkedHashMap 的节点输出是有序的
   @Test
    public void test8(){
        LinkedHashMap<String,String> map = new LinkedHashMap<>();
        map.put("a1","a");
        map.put("a2","b");
        map.put("b1","c");
        map.put("b2","d");
        map.put("c1","e");
        for (String s : map.values()) System.out.println(s);

    }

a
b
c
d
e

//开启 LinkedHashMap 的使用顺序
    @Test
    public void test8(){
        LinkedHashMap<String,String> map = new LinkedHashMap<String,String>(16,0.75f,true);
        map.put("a1","a");
        map.put("a2","b");
        map.put("b1","c");
        map.put("b2","d");
        map.put("c1","e");
        String a = map.get("b1");
        for (String s : map.values()) System.out.println(s);
    }

a
b
d
e
c
```