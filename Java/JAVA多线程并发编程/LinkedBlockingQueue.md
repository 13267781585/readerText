## LinkedBlockingQueue 源码   



1. 保证线程安全的策略  
ReentrantLock 双锁机制，put锁和take锁分离，提高效率，用 AtomicInteger 保证count 线程安全，因为put和take锁分离，head和last指针同时会被take和put操作，采用中间增加一个空节点隔离前take后put操作，保证线程安全
```java
    /** Current number of elements */
    private final AtomicInteger count = new AtomicInteger();

    /** Main lock guarding all access */
    private final ReentrantLock takeLock = new ReentrantLock();

    /** Wait queue for waiting takes */
    private final Condition notEmpty = takeLock.newCondition();

    /** Lock held by put, offer, etc */
    private final ReentrantLock putLock = new ReentrantLock();

    /** Wait queue for waiting puts */
    private final Condition notFull = putLock.newCondition();

    public LinkedBlockingQueue(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException();
        this.capacity = capacity;
        //隔离 put 和 take 操作的节点
        last = head = new Node<E>(null);
    }
```

2. 底层数据结构  
```java
    //单向链表 减小生成 Node 对象的消耗  在创建时不指定大小，默认Integer.MAX_VALUE，有长度限制
    static class Node<E> {
        E item;
        Node<E> next;

        Node(E x) { item = x; }
    }

    transient Node<E> first;

    transient Node<E> last;

```

3. 提供两种api，阻塞和非阻塞
* 入列和出列
```java
    //出列只操作head
    private E dequeue() {
        Node<E> h = head;
        Node<E> first = h.next;
        //head指向空节点 把空节点出列，把第一个元素当作空节点使用 不涉及last的操作
        h.next = h; // help GC
        head = first;
        E x = first.item;
        first.item = null;
        return x;
    }

    //入列 只涉及 last 的操作
    private void enqueue(Node<E> node) {
        last = last.next = node;
    }
```
* 非阻塞  
```java
    public boolean offer(E e) {
        if (e == null) throw new NullPointerException();
        final AtomicInteger count = this.count;
        if (count.get() == capacity)
            return false;
        int c = -1;
        Node<E> node = new Node<E>(e);
        final ReentrantLock putLock = this.putLock;
        putLock.lock();
        try {
            if (count.get() < capacity) {
                enqueue(node);
                c = count.getAndIncrement();
                if (c + 1 < capacity)
                    //判断剩余空间 唤醒 put 操作，提高效率
                    notFull.signal();
            }
        } finally {
            putLock.unlock();
        }
        if (c == 0)
            //
            signalNotEmpty();
        return c >= 0;
    }

    private void signalNotEmpty() {
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lock();
        try {
            notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
    }

    public E poll() {
        final AtomicInteger count = this.count;
        if (count.get() == 0)
            return null;
        E x = null;
        int c = -1;
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lock();
        try {
            if (count.get() > 0) {
                x = dequeue();
                c = count.getAndDecrement();
                if (c > 1)
                    notEmpty.signal();
            }
        } finally {
            takeLock.unlock();
        }
        if (c == capacity)
            signalNotFull();
        return x;
    }

    private void signalNotFull() {
        final ReentrantLock putLock = this.putLock;
        putLock.lock();
        try {
            notFull.signal();
        } finally {
            putLock.unlock();
        }
    }
    
```

* 阻塞
```java
    public void put(E e) throws InterruptedException {
        if (e == null) throw new NullPointerException();
        int c = -1;
        Node<E> node = new Node<E>(e);
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        putLock.lockInterruptibly();
        try {
            //阻塞
            while (count.get() == capacity) {
                notFull.await();
            }
            enqueue(node);
            //队列元素个数自增，注意，由于这里调用的是getAndIncrement()方法，
            // 不是incrementAndGet()方法，所以返回的是自增之前的值。
            c = count.getAndIncrement();
            if (c + 1 < capacity)
                //还有剩余空间 唤醒put操作
                notFull.signal();
        } finally {
            putLock.unlock();
        }
        // 如果阻塞队列在没有添加元素之前，阻塞队列的元素个数为0，那么可能有线程处于notEmpty的等待队列中
        // 因此这里会唤醒处于notEmpty的等待队列中的线程
        if (c == 0)
            signalNotEmpty();
    }

    public E take() throws InterruptedException {
        E x;
        int c = -1;
        final AtomicInteger count = this.count;
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lockInterruptibly();
        try {
            while (count.get() == 0) {
                notEmpty.await();
            }
            x = dequeue();
            c = count.getAndDecrement();
            if (c > 1)
                notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
        if (c == capacity)
            signalNotFull();
        return x;
    }
```

#### ArrayBlockingQueue 为什么不使用双锁机制？？(不理解)
这个主要是循环队列的原因，主要是数组和链表不同，链表队列的添加和头部的删除，都是只和一个节点相关，添加只往后加就可以，删除只从头部去掉就好。为了防止head和tail相互影响出现问题，这里就需要原子性的计数器，头部要移除，首先得看计数器是否大于0，每个添加操作，都是先加入队列，然后计数器加1，这样保证了，队列在移除的时候，长度是大于等于计数器的，通过原子性的计数器，双锁才能互不干扰。数组的一个问题就是位置的选择没有办法原子化，因为位置会循环，走到最后一个位置后就返回到第一个位置，这样的操作无法原子化，所以只能是加锁来解决。
https://www.imooc.com/article/18726






#### LinkedBlockingQueue 和 LinkedBlockingDeque 的区别 
* 锁机制：LinkedBlockingQueue是双锁，效率高，LinkedBlockingDeque是单锁  
* 底层数据结构：LinkedBlockingQueue是基于单链表实现的，提供的api只能按照队列严格的先进先出顺序，LinkedBlockingDeque基于双向链表实现，提供了逆向遍历的迭代器，提供了丰富的api，可以在头尾进行增加删除，可以实现更加复杂的功能
* 初始化：LinkedBlockingQueue是双锁，put和take可能同时操作head，last指针，会出现线程安全问题，初始化是生成一个空的Node节点隔离前take后put的操作，LinkedBlockingDeque是单锁，head和last被put和take一个操作，不存在问题   
* 唤醒模型：LinkedBlockingQueue是在有剩余空间时put唤醒put操作，只有在元素数量为0是才会唤醒get操作，LinkedBlockingDeque是交叉唤醒，put完唤醒take，take完唤醒put(由于LinkedBlockingQueue是双锁机制，交叉唤醒需要获取对应的锁，效率很低，且大部门情况下条件队列为空，唤醒操作是多余的)  



#### 并发容器优缺点  
* ArrayBlockingQueue 
* LinkedBlockingDeque 双向链表，提供丰富api，可以实现多种功能   
* LinkedBlockingQueue 单向链表，双锁机制，效率高，只能前进先出，元素被转化为Node，当大量数据增删给gc带来负担 


#### head、last不需要保证可见性吗？


#### 初始化可以不指定队列长度，默认 Integer.MAX_VALUE,可能发生内层溢出