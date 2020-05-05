## LinkedList 源码解读

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
```

1. LinkedList 底层是用双向链表实现的  
```java
private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
```

2. 发现一个内部方法细节处理很好
```java
//LinkedList的内部私有方法，查找并返回第index个节点
Node<E> node(int index) {
        // assert isElementIndex(index);
        //因为链表没有索引，需要利用指针遍历，这里做了一个优化，判断需要放回的在链表的前半段还是后半段决定从头向后遍历还是从后向前遍历，在数据量很大的时候可以提高效率
        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
```

3. 因为这里实现了队列接口，所以这么实现了很多关于头结点尾节点的操作，使用起来很方便。
```java
//返回头结点
public E peek();

//返回并弹出头结点
public E poll();

public E peekFirst();
public E peekLast();
public E pollFirst();
public E pollLast();
public void push(E e);
public E pop();
···
```

4. LinkedList是线程不安全的

5. ArrayList 和 LinkedList  的区别   
i. ArrayList 是基于数组实现的，所以在查改有优势，但增删有缺陷，还有在空间不足时需要考虑扩容的问题；而LinkedList 是基于链表实现的，增删比较方便，没有扩容问题，但查改效率低。   
ii. 两者都时线程不安全的，但都提供了 spliterator 用于并行遍历   
iii. 查改频繁用ArrayList，增删频繁用LinkedList。