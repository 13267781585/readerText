
## LinkedBlockingDeque 源码   


1. 保证线程安全的策略  
```java
    /** Main lock guarding all access */
    final ReentrantLock lock;

    /** Condition for waiting takes */
    private final Condition notEmpty;

    /** Condition for waiting puts */
    private final Condition notFull;
```

2. 底层数据结构  
```java
    //双向链表  在创建时不指定大小，默认Integer.MAX_VALUE，有长度限制
    static final class Node<E> {
        E item;

        Node<E> prev;
        Node<E> next;

        Node(E x) {
            item = x;
        }
    }

    transient Node<E> first;

    transient Node<E> last;

```

3. 提供两种api，阻塞和非阻塞
* 入列和出列
```java
    private boolean linkLast(Node<E> node) {
        // assert lock.isHeldByCurrentThread();
        if (count >= capacity)
            return false;
        Node<E> l = last;
        node.prev = l;
        last = node;
        if (first == null)
            first = node;
        else
            l.next = node;
        ++count;
        notEmpty.signal();
        return true;
    }

    private E unlinkFirst() {
        // assert lock.isHeldByCurrentThread();
        Node<E> f = first;
        if (f == null)
            return null;
        Node<E> n = f.next;
        E item = f.item;
        f.item = null;
        f.next = f; // help GC
        first = n;
        if (n == null)
            last = null;
        else
            n.prev = null;
        --count;
        notFull.signal();
        return item;
    }
```
* 非阻塞  
```java
    public boolean offer(E e) {
        return offerLast(e);
    }

    public boolean offerLast(E e) {
        if (e == null) throw new NullPointerException();
        Node<E> node = new Node<E>(e);
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //连接最后  唤醒take线程
            return linkLast(node);
        } finally {
            lock.unlock();
        }
    }

    public E poll() {
        return pollFirst();
    }

    public E pollFirst() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //删除头结点 唤醒 put 线程
            return unlinkFirst();
        } finally {
            lock.unlock();
        }
    }
    
```

* 阻塞
```java
    public E take() throws InterruptedException {
        return takeFirst();
    }

    public E takeFirst() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            E x;
            while ( (x = unlinkFirst()) == null)
                notEmpty.await();
            return x;
        } finally {
            lock.unlock();
        }
    }


public void put(E e) throws InterruptedException {
        putLast(e);
    }

        public void putLast(E e) throws InterruptedException {
        if (e == null) throw new NullPointerException();
        Node<E> node = new Node<E>(e);
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //
            while (!linkLast(node))
                notFull.await();
        } finally {
            lock.unlock();
        }
    }
```

4. 初始化可以不指定队列长度，默认 Integer.MAX_VALUE,可能发生内层溢出