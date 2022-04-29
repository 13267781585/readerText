## CyclicBarrier
#### 实现原理
* ReentrantLock + condition



#### 应用场景
* 计数器(CountDownLatch和CyclicBarrier实现计算器的区别，CyclicBarrier去减少计数器时线程被阻塞住，而CountdownLatch减少计数器后线程可以返回，相比之下CountDownLatch更好)
* 需要同时启动的任务



#### 属性
```java
    //锁
    private final ReentrantLock lock = new ReentrantLock();
    //阻塞队列
    private final Condition trip = lock.newCondition();
    //初始化设置的资源个数
    private final int parties;
    //当资源为0 唤醒阻塞线程前执行的任务
    private final Runnable barrierCommand;
    //当前的版本
    private Generation generation = new Generation();

    //等待的任务个数
    private int count;
```

#### await方法
* 提供了两种阻塞模式：无限期阻塞和有时间限制阻塞(在固定时间后线程没有被唤醒会抛出异常并且唤醒其他线程)
```java
    public int await() throws InterruptedException, BrokenBarrierException {
        try {
            return dowait(false, 0L);
        } catch (TimeoutException toe) {
            throw new Error(toe); // cannot happen
        }
    }


    private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
        final ReentrantLock lock = this.lock;
        //加锁
        lock.lock();
        try {
            final Generation g = generation;
            //是否在调用过程中被重置
            if (g.broken)
                throw new BrokenBarrierException();
            //是否被中断
            if (Thread.interrupted()) {
                breakBarrier();
                throw new InterruptedException();
            }

            int index = --count;
            //如果计数器为0 唤醒阻塞的线程
            if (index == 0) {  // tripped
                boolean ranAction = false;
                try {
                    final Runnable command = barrierCommand;
                    if (command != null)
                        command.run();
                    ranAction = true;
                    //唤醒线程
                    nextGeneration();
                    return 0;
                } finally {
                    if (!ranAction)
                        breakBarrier();
                }
            }

            for (;;) {
                try {

                    //阻塞线程到条件队列上
                    if (!timed)
                        trip.await();
                    else if (nanos > 0L)
                        nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) {
                    if (g == generation && ! g.broken) {
                        breakBarrier();
                        throw ie;
                    } else {
                        // We're about to finish waiting even if we had not
                        // been interrupted, so this interrupt is deemed to
                        // "belong" to subsequent execution.
                        Thread.currentThread().interrupt();
                    }
                }

                if (g.broken)
                    throw new BrokenBarrierException();

                if (g != generation)
                    return index;

                if (timed && nanos <= 0L) {
                    breakBarrier();
                    throw new TimeoutException();
                }
            }
        } finally {
            lock.unlock();
        }
    }

    private void nextGeneration() {
        // signal completion of last generation
        trip.signalAll();
        // set up next generation
        count = parties;
        generation = new Generation();
    }
```

#### reset 重置计数器
```java
    public void reset() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //唤醒阻塞线程和重置计数器
            breakBarrier();   // break the current generation
            nextGeneration(); // start a new generation
        } finally {
            lock.unlock();
        }
    }

    private void breakBarrier() {
        generation.broken = true;
        count = parties;
        trip.signalAll();
    }

    private void nextGeneration() {
        // signal completion of last generation
        trip.signalAll();
        // set up next generation
        count = parties;
        generation = new Generation();
    }
```