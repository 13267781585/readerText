## HashMap 源码解读(JDK 1.8)

```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable 
```

1. 存放数据的容器用的是内部自定义的类

```java
static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;    //哈希值
        final K key;     //键
        V value;        //值
        Node<K,V> next;    //指向下一个哈希值相同的节点

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        public final K getKey()        { return key; }
        public final V getValue()      { return value; }
        public final String toString() { return key + "=" + value; }

        //需要重写 hashCode() 和 equals() 方法，很重要，因为在使用的过程中会有很多判断两个变量是否相同的情况
        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }

        public final boolean equals(Object o) {
            if (o == this)
                return true;
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>)o;
                if (Objects.equals(key, e.getKey()) &&
                    Objects.equals(value, e.getValue()))
                    return true;
            }
            return false;
        }
    }

//用的是 Node数组，默认为null
transient Node<K,V>[] table;
```

2. 初始化

```java
//系统参数
//最小的红黑树容量
static final int MIN_TREEIFY_CAPACITY = 64;

//负载因子，当数组 使用的空间/总空间 = 负载因子 时需要扩容
static final float DEFAULT_LOAD_FACTOR = 0.75f;

//存放Node节点数组最大长度
static final int MAXIMUM_CAPACITY = 1 << 30;
//第一次初始化数组长度
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; 

//数组容量达到该值扩容
int threshold;

//不传参数
 public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }

//传入HashMap 的初始化长度
public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

//传入 初始化长度 和 负载因子
 public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }

//这里 tableSizeFor 函数 处理用户传入的初始化长度，返回大于传入值的2倍整数
//补充说明：算法就是让初始二进制分别右移1，2，4，8，16位，与自己异或，把高位第一个为1的数通过不断右移，把高位为1的后面全变为1
 static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }

```

3. 数组扩容(为原数组大小的2倍)
threshold变量表示数组已用空间达到该值时需要扩容，创建 HashMap 对象的时候若没有传入参数，threshold默认为16，若传入初始值，则threshold为大于初始值的2倍整数，第一次扩容大小为 thrshold，然后 threshold = 数组总容量 * 负载因子。

```java
    //数组容量达到该值扩容
    int threshold;
    static final int MAXIMUM_CAPACITY = 1 << 30;
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    //检查数组是否需要扩容
    final HashMap.Node<K,V>[] resize() {
        HashMap.Node<K,V>[] oldTab = table;
        //原有数组长度
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        //扩容后的长度
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            //原来数组长度是否大于系统设定最大长度(int可表示范围一半)说明不能在扩容，会超出数值表述范围
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            //说明原来的容量小于 int 表示范围一半，可扩大为原来的2倍也不会超出范围
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                    oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) //说明是第一次扩容 数组的长度为 0
            newCap = oldThr;
        else {              
            //说明是第一次扩容
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                    (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
        HashMap.Node<K,V>[] newTab = (HashMap.Node<K,V>[])new HashMap.Node[newCap];
        table = newTab;
        //将原来数组中的元素复制到新的数组中
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                HashMap.Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;          //根据元素hash 和新的数组长度 计算 新的索引
                    else if (e instanceof HashMap.TreeNode)       //如果采用的存储结构是红黑树
                        ((HashMap.TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        //说明采用的存储结构是链表，在位置上可能存放一个或者多个元素 
                        HashMap.Node<K,V> loHead = null, loTail = null;
                        HashMap.Node<K,V> hiHead = null, hiTail = null;
                        HashMap.Node<K,V> next;
                        do {
                            next = e.next;
                            //判断高位是否为0
                            //采用高低位原理，将链表分为高位和低位链表，一次性迁移
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```

4. 存取方式
HashMap 的存取数据是通过 算法计算数据的 hash 映射到数组的某一位置上，不需要遍历整个数组进行操作，所以效率比较高。

```java
//结合 key 和 value 的值计算 hash
//扰动函数
  static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

* 为什么用哈希值的高16位异或低16位？
 因为 key.hashCode() 函数调用的是key键值类型自带的哈希函数，返回int型散列值。int值范围为**-2147483648~2147483647**，前后加起来大概40亿的映射空间。只要哈希函数映射得比较均匀松散，一般应用是很难出现碰撞的。但问题是一个40亿长度的数组，内存是放不下的。你想，如果HashMap数组的初始大小才16，用之前需要对数组的长度取模运算，得到的余数才能用来访问数组下标。

 ```java
bucketIndex = indexFor(hash, table.length);

static int indexFor(int h, int length) {
     return h & (length-1);
}
```

顺便说一下，这也正好解释了为什么HashMap的数组长度要取2的整数幂。因为这样（数组长度-1）正好相当于一个“低位掩码”。“与”操作的结果就是散列值的高位全部归零，只保留低位值，用来做数组下标访问。以初始长度16为例，16-1=15。2进制表示是00000000 00000000 00001111。和某散列值做“与”操作如下，结果就是截取了最低的四位值。

```java
10100101 11000100 00100101
& 00000000 00000000 00001111
----------------------------------
  00000000 00000000 00000101    //高位全部归零，只保留末四位
```

但这时候问题就来了，这样就算我的散列值分布再松散，要是只取最后几位的话，碰撞也会很严重。更要命的是如果散列本身做得不好，分布上成等差数列的漏洞，如果正好让最后几个低位呈现规律性重复，就无比蛋疼。

这时候 hash 函数（“扰动函数”）的价值就体现出来了，说到这里大家应该猜出来了。看下面这个图，

<img src="..\image\HashMap-4-1.jpg" alt="4-1" />
右位移16位，正好是32bit的一半，自己的高半区和低半区做异或，就是为了混合原始哈希码的高位和低位，以此来加大低位的随机性。而且混合后的低位掺杂了高位的部分特征，这样高位的信息也被变相保留下来。
摘取自https://zhuanlan.zhihu.com/p/133618417

源码中模运算就是把散列值和数组长度-1做一个"与"操作，位运算比%运算要快。

5. 存值
存值时首先会通过hash计算数组索引添加到对应位置上，若该位置上已经有节点，则用链表的方式添加到尾部，若链表长度超过系统设置范围，则将链表转化为红黑树。  
大致过程: 判断数组是否为空 -> 数组该位置上是否有值 -> 第一个元素是否和 插入元素 key 相同 -> 判断存储节点的结构是否为红黑树 -> 循环链表找到相同的元素替换旧值或者在尾部插入节点

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        HashMap.Node<K,V>[] tab; HashMap.Node<K,V> p; int n, i;
        //判断数组是否为空，是则初始化数组
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        //通过 hash 算法计算数组的位置索引，判断该位置若为null，直接插入
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            //数组对应位置上不为空，说明该位置上有一个或多个节点，采用的结构可能是链表或红黑树
            //关键是我们放入的节点可能该位置上不存在-> 连接新节点  已经存在 -> 替换旧的值
            //p 指向 该位置上第一个 元素  e 指向是否有节点 key 和 添加 key 相同，没有则为 null
            HashMap.Node<K,V> e; K k;
            //判断第一个和我们添加的节点 key 是否相同
            if (p.hash == hash &&
                    ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            //判断该位置采用的是红黑树的存储结构，是则按红黑树存值添加节点
            else if (p instanceof HashMap.TreeNode)
                e = ((HashMap.TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                //说明采用的是 链表 存储结构，循环遍历节点
                for (int binCount = 0; ; ++binCount) {
                    //遍历到链表尾部，说明添加的 节点是新的
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        //插入后判断链表长度是否大于系统规定范围，超过则将链表转化为红黑树，提高效率
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    //判断key是否相同
                    if (e.hash == hash &&
                            ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            //说明添加的key 已经存在，替换原来的数值
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                //onlyIfAbsent 是用户传入的参数，表示若添加的key已经存在，是否替换原来的值
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        //更新内部计数器和检查是否需要扩容
        ++modCount;
        //判断数组是否需要扩容
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```

6. 取值
大致过程: 判断数组是否为空且在位置上是否有节点 -> 第一个节点是否是我们需要的 -> 数组该位置上采用的是链表还是红黑树存放节点 -> 循环所有节点查找目标节点

```java
  public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }


 final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        //检查数组不为空且通过hash计算的到的对应位置上不为null
        //对应位置上不为null说明该位置上有一个或多个节点
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            //判断第一个是否是我们需要的
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            //第一个不是那就循环链表
            if ((e = first.next) != null) {
                //判断节点是否为 TreeNode 类型，是则说明该位置节点超过系统设置的范围，已经被转化为红黑树了，所以需要调用红黑树的查找方法找到该节点
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        //数据为空返回 null
        return null;
    }
```

7. 当数组中一个位置节点数大于8且数组的长度大于64，节点存储在该位置上的结构会由链表变为红黑树，如果红黑树的节点小于6，则会由红黑树转化为链表。  
两个阙值不同是因为如果hash碰撞次数在8附近徘徊，会一直发生链表和红黑树的转化，为了预防这种情况的发生。。

8. HashMap 是线程不安全的

9. 1.7和1.8的区别
i. JDK1.7用的是头插法，而JDK1.8及之后使用的都是尾插法.因为JDK1.7是用单链表进行的纵向延伸，当采用头插法就是能够提高插入的效率，但是也会容易出现逆序且环形链表死循环问题。但是在JDK1.8之后是因为加入了红黑树使用尾插法，能够避免出现逆序且链表死循环的问题。

ii. 扩容后数据存储位置的计算方式也不一样：1. 在JDK1.7的时候是直接用hash值和扩容后长度-1去做与运算，而在JDK1.8只需要判断Hash值的新增参与运算的位是0还是1就直接迅速计算出了扩容后的储存方式(用hash&oldheight?=0)。

iii. JDK1.7的时候使用的是数组+ 单链表的数据结构。但是在JDK1.8及之后时，使用的是数组+链表+红黑树的数据结构（当链表的深度达到8的时候，也就是默认阈值，就会自动扩容把链表转成红黑树的数据结构来把时间复杂度从O（n）变成O（nlogN）提高了效率）  

iv. JDK1.7的时候是先进行扩容后进行插入，而在JDK1.8的时候则是先插入后进行扩容的,因为当插入的节点已经存在时，只是覆盖原来的值，节点的个数没有增加，可以省去一次不必要的扩容。

10. 为什么要用异或运算符？  
保证了对象的 hashCode 的 32 位值只要有一位发生改变，整个 hash() 返回值就会改变。尽可能的减少碰撞。

11. HashMap，LinkedHashMap，TreeMap 有什么区别？
LinkedHashMap 保存了记录的插入顺序，在用 Iterator 遍历时，先取到的记录肯定是先插入的；遍历比 HashMap 慢；
TreeMap 实现 SortMap 接口，能够把它保存的记录根据键排序（默认按键值升序排序，也可以指定排序的比较器）
13. HashMap & TreeMap & LinkedHashMap 使用场景？
一般情况下，使用最多的是 HashMap。
HashMap：在 Map 中插入、删除和定位元素时；
TreeMap：在需要按自然顺序或自定义顺序遍历键的情况下；
LinkedHashMap：在需要输出的顺序和输入的顺序相同的情况下。

13. 为什么每次要扩容2倍，而不是3倍?

* 索引的计算，长度-1可以作为掩码计算位置，扩容后只需要判断高位是否为0就可以计算出扩容后的位置
* 哈希冲突，扩容两倍后，对于每一个节点来说，要么在原位置，要么迁移到新的开辟出来的空间，所以最坏情况下该位置的所有节点不变，哈希冲突保持不变，如果使用的是其他倍数，节点的分布不规则，可能会出现更严重的哈希冲突；

14. 为什么hashmap的负载因子默认是0.75？

* 空间利用率和哈希冲突的综合考虑
* 0.75 可以做到良好的空间利用率和尽可能减小哈希冲突

15. 为什么当节点个数达到8后转化为红黑树？
当hash值是随机的情况下，负载因子是0.75时，每个桶出现节点数量的频次符合松柏分布，每个桶的节点数量为8的几率非常小，若出现桶的节点数量为8，说明哈希冲突严重，为了效率，转化为红黑树

16. hashmap解决哈希冲突的方法？

* 扩容
* 红黑树
* hash函数
* 扩容2倍数

17. 如何解决1.7hashmap并发环问题？

* 尾插法
* 设置一个标志位，扩容时直接返回？？？

18. 哈希冲突解决算法
a.拉链法
将哈希冲突的元素以链表的形式存放

* 优点：实现简单，增删的效率高，在大数据集时使用
* 缺点：需要额外的指针指向哈希冲突的元素
b.线性嗅探法
发生哈希冲突时，向后寻找没有节点的空槽放入
c.二次线性嗅探法
和线性嗅探法原理一致，只是不是按顺序寻找下一个，而是以+-2^n次方的增量去寻找
* 优点：小数据集时节省空间，不需要额外的指针
* 增删处理复杂，会有堆积的问题(由于哈希冲突元素集中在一起，二次线性嗅探有效解决)
d.双重散列
和线性嗅探法原理一致，使用另一个哈希函数计算下一个位置。
e.扩容
