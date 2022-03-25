## ArrayBlockingQueue 源码   

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
    //不可变数组  在创建需要指定大小  不扩容
    final Object[] items;
    ArrayBlockingQueue<Integer> queue = new ArrayBlockingQueue<>(10);

    //数组在进行删除和增加元素并不需要进行数据迁移，维护了 put 和 get 指针
    /** items index for next take, poll, peek or remove */
    int takeIndex;

    /** items index for next put, offer, or add */
    int putIndex;

    /** Number of elements in the queue */
    int count;

```

3. 提供两种api，阻塞和非阻塞
* 元素入列和出列
```java
    //入列
    private void enqueue(E x) {
        final Object[] items = this.items;
        items[putIndex] = x;
        if (++putIndex == items.length)
            putIndex = 0;
        count++;
        //唤醒take线程
        notEmpty.signal();
    }

    //出列
    private E dequeue() {
        final Object[] items = this.items;
        @SuppressWarnings("unchecked")
        E x = (E) items[takeIndex];
        items[takeIndex] = null;
        if (++takeIndex == items.length)
            takeIndex = 0;
        count--;
        if (itrs != null)
            itrs.elementDequeued();
        //唤醒put线程
        notFull.signal();
        return x;
    }

```
* 非阻塞  
```java
    public boolean offer(E e) {
        checkNotNull(e);
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            if (count == items.length)
                return false;
            else {
                enqueue(e);
                return true;
            }
        } finally {
            lock.unlock();
        }
    }

        public E poll() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return (count == 0) ? null : dequeue();
        } finally {
            lock.unlock();
        }
    }
```

* 阻塞
```java
    public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            //如果集合中没有元素  阻塞线程到条件队列
            while (count == 0)
                notEmpty.await();
            //出列一个元素 并 唤醒put阻塞条件队列的一个线程
            return dequeue();
        } finally {
            lock.unlock();
        }
    }


    public void put(E e) throws InterruptedException {
        checkNotNull(e);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            //如果数组没有空间存放  阻塞到条件队列
            while (count == items.length)
                notFull.await();
            //放入一个元素 并 唤醒get阻塞队列上的一个线程
            enqueue(e);
        } finally {
            lock.unlock();
        }
    }
```

4. 条件队列唤醒策略   
在创建集合可以确定是否使用公平策略唤醒条件队列中的线程
```java
    public ArrayBlockingQueue(int capacity, boolean fair) {
        if (capacity <= 0)
            throw new IllegalArgumentException();
        this.items = new Object[capacity];
        lock = new ReentrantLock(fair);
        notEmpty = lock.newCondition();
        notFull =  lock.newCondition();
    }
```
5. 初始化需要指定长度，没有提供默认构造函数，因为需要初始化数组，需要显示指定大小