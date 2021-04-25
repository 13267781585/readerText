## Hashtable 源码解读

```java
public class HashSet<E>
    extends AbstractSet<E>
    implements Set<E>, Cloneable, java.io.Serializable
```
1. 底层数据结构
Hashtable 内部是用 数组 + 链表 的形式实现的
```java
//存放节点的数组
private transient Entry<?,?>[] table;

//自定义数据结构
 private static class Entry<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Entry<K,V> next;
 }
```

2. 初始化
```java
//传入参数，按参数申请数组大小，当传入为0时，申请空间为1
public Hashtable(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal Load: "+loadFactor);

        if (initialCapacity==0)
            initialCapacity = 1;
        this.loadFactor = loadFactor;
        table = new Entry<?,?>[initialCapacity];
        threshold = (int)Math.min(initialCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
    }

public Hashtable(int initialCapacity) {
        this(initialCapacity, 0.75f);
    }

//不传入任何参数，默认申请数组大小为11，负载因子为0.75
public Hashtable() {
        this(11, 0.75f);
    }

```

3. 扩容
Hashtable 若创建对象没有传入初始化长度，则默认为11，当使用的空间超过 原有数组长度 * 负载因子，则扩容到原来数组长度的 2倍 + 1
```java
protected void rehash() {
        int oldCapacity = table.length;
        Entry<?,?>[] oldMap = table;

        // 新的数组长度 为 原有的2倍 + 1
        int newCapacity = (oldCapacity << 1) + 1;
        if (newCapacity - MAX_ARRAY_SIZE > 0) {
            if (oldCapacity == MAX_ARRAY_SIZE)
                // Keep running with MAX_ARRAY_SIZE buckets
                return;
            newCapacity = MAX_ARRAY_SIZE;
        }
        Entry<?,?>[] newMap = new Entry<?,?>[newCapacity];

        modCount++;
        threshold = (int)Math.min(newCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
        table = newMap;

        //将原来数组中的元素复制到新的数组中
        for (int i = oldCapacity ; i-- > 0 ;) {
            for (Entry<K,V> old = (Entry<K,V>)oldMap[i] ; old != null ; ) {
                Entry<K,V> e = old;
                old = old.next;
                //重新计算元素的下标
                int index = (e.hash & 0x7FFFFFFF) % newCapacity;
                e.next = (Entry<K,V>)newMap[index];
                newMap[index] = e;
            }
        }
    }
```

4. Hashtable 和 HashMap 的区别   
i.  hash函数 不同
```java
//HashMap 用 key 的hasCode() 函数返回值 高16位 异或 低16位 作为hash值
 static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }

//Hashtable 用 key 的 hasCode()函数返回值 作为hash值
int hash = key.hashCode();
```

ii. Hashtable 不允许放入 null 的数据，HashMap则允许

iii. Hashtable是线程安全的，在方法前加了 synchronized 限制，HashMap 则是线程不安全的

iv. 计算 index 的方式不同
```java
//Hashtable 用的是模运算，(hash & 0x7FFFFFFF)是为了让返回值为正的
int index = (hash & 0x7FFFFFFF) % tab.length;

//HashMap 用的是位运算，因为数组长度刚好是2的整数倍，-1后可以作为掩码
static int indexFor(int h, int length) {
     return h & (length-1);
}
```

vi. Hashtable 中节点插入数组中是 头插法， 而 HashMap 是 尾插法。