# Disruptor源码解读
## 高性能的原因
* 无锁设计：使用写覆盖替代加锁+删数据
* 并发消费：多个消费者可以同时处理一个数据
* 避免伪共：根据cpu缓存特性，提高缓存利用率

## 知识点
### CPU自旋优化
JDK9以上增加了onSpinWait方法，在线程在自旋等待某些条件时，可以提示CPU将资源更多分配给其他线程，防止长时间空旋占用过多的资源(当进程采用自旋时，建议将进程和特定的CPU进行绑定，防止该进程占用过多的cpu资源去等待条件的完成)。
```java
    // ThreadHints.onSpinWait();
    public final class ThreadHints
{
    private static final MethodHandle ON_SPIN_WAIT_METHOD_HANDLE;

    static
    {
        // 使用onSpinWait方法必须在JDK9以上 否则MethodHandle为nil
        final MethodHandles.Lookup lookup = MethodHandles.lookup();

        MethodHandle methodHandle = null;
        try
        {
            methodHandle = lookup.findStatic(Thread.class, "onSpinWait", methodType(void.class));
        }
        catch (final Exception ignore)
        {
        }

        ON_SPIN_WAIT_METHOD_HANDLE = methodHandle;
    }

    private ThreadHints()
    {
    }

    public static void onSpinWait()
    {
        // 如果使用的JDK版本在9以下 ON_SPIN_WAIT_METHOD_HANDLE为nil
        if (null != ON_SPIN_WAIT_METHOD_HANDLE)
        {
            try
            {
                // 提醒CPU该线程处于自旋状态，可以将资源更多分配给其他线程
                ON_SPIN_WAIT_METHOD_HANDLE.invokeExact();
            }
            catch (final Throwable ignore)
            {
            }
        }
    }
}
```

### 伪共享
缓存系统中是以缓存行（cache line）为单位存储的，当多线程修改互相独立的变量时，如果这些变量共享同一个缓存行，只有一个线程可以操作改缓存行且在修改后其他的缓存副本会被失效。
#### 解决方式
* 数据填充   
[伪共享（false sharing），并发编程无声的性能杀手](https://www.cnblogs.com/cyfonly/p/5800758.html)

## SingleProducerSequencer
### 发布申请
```java
/**
     * @see Sequencer#next(int)
     */
    @Override
    public long next(int n)
    {
        if (n < 1)
        {
            throw new IllegalArgumentException("n must be > 0");
        }
        //下一个被分配的位置(在这之前的位置已经被分配出去但不一定被发布)
        long nextValue = this.nextValue;

        //申请的位置
        long nextSequence = nextValue + n;
        //申请的位置的上一轮的位置 用于判断申请位置中间是否还有没被消费的事件
        long wrapPoint = nextSequence - bufferSize;
        //上一次申请缓存的最慢消费者位置
        long cachedGatingSequence = this.cachedValue;

        //这里是一个优化点 
        //申请成功的限制：申请的位置中不能还有事件没有被消费
        //解决方案：遍历消费者拿到最慢的消费位置(耗时操作)
        //优化方案：缓存上一次计算的最小消费者位置，如果申请的位置<缓存的位置，不需要计算
        if (wrapPoint > cachedGatingSequence || cachedGatingSequence > nextValue)
        {
            //cursor 代表已经被分配出去且发布的最大位置
            cursor.setVolatile(nextValue);  // StoreLoad fence

            //计算最慢消费者位置，最慢位置<申请位置 阻塞线程
            long minSequence;
            while (wrapPoint > (minSequence = Util.getMinimumSequence(gatingSequences, nextValue)))
            {
                LockSupport.parkNanos(1L); // TODO: Use waitStrategy to spin?
            }
            //缓存最慢消费者位置
            this.cachedValue = minSequence;
        }

        this.nextValue = nextSequence;

        return nextSequence;
    }
```

### 发布提交
```java
   @Override
    public void publish(long sequence)
    {
        // 设置最大已发布位置
        cursor.set(sequence);
        // 唤醒所有等待的消费者
        waitStrategy.signalAllWhenBlocking();
    }
```

## MultiProducerSequencer
在多消费者中，cursor表示已分配的最大位置
### 发布申请
```java
    public long next(int n)
    {
        if (n < 1)
        {
            throw new IllegalArgumentException("n must be > 0");
        }

        long current;
        long next;

        do
        {
            // 已分配的最大位置 因为有多个生产者 所以每次都要重新获取
            current = cursor.get();
            // 需要申请的最大位置
            next = current + n;

            long wrapPoint = next - bufferSize;
            // 上一次查询的消费者最慢进度(减少调用getMinimumSequence耗时操作)
            long cachedGatingSequence = gatingSequenceCache.get();

            if (wrapPoint > cachedGatingSequence || cachedGatingSequence > current)
            {
                long gatingSequence = Util.getMinimumSequence(gatingSequences, current);

                if (wrapPoint > gatingSequence)
                {
                    LockSupport.parkNanos(1); // TODO, should we spin based on the wait strategy?
                    continue;
                }

                gatingSequenceCache.set(gatingSequence);
            }
            // 如果满足申请的条件 cas设置cursor的值
            else if (cursor.compareAndSet(current, next))
            {
                break;
            }
        }
        while (true);

        return next;
    }
```

### 发布提交
```java
    @Override
    public void publish(final long sequence)
    {
        setAvailable(sequence);
        waitStrategy.signalAllWhenBlocking();
    }

    private void setAvailable(final long sequence)
    {
        setAvailableBufferValue(calculateIndex(sequence), calculateAvailabilityFlag(sequence));
    }

    //标识对应 ringbuffer 位置上数据是否被发布
    private final int[] availableBuffer;
    private void setAvailableBufferValue(int index, int flag)
    {
        long bufferAddress = (index * SCALE) + BASE;
        UNSAFE.putOrderedInt(availableBuffer, bufferAddress, flag);
    }
```

## Disruptor
## 创建消费者
```java
    /*
        disruptor.handleEventsWith(new OrderEventHandler(1),new OrderEventHandler(2)).then(new OrderEventHandler(3));
    */
      public final EventHandlerGroup<T> handleEventsWith(final EventHandler<? super T>... handlers)
    {
        return createEventProcessors(new Sequence[0], handlers);
    }

    /*
        barrierSequences 依赖消费者的消费进度(A->B，本次B的创建，传入A的消费进度)
        eventHandlers 消费者
    */
    EventHandlerGroup<T> createEventProcessors(
        final Sequence[] barrierSequences,
        final EventHandler<? super T>[] eventHandlers)
    {
        checkNotStarted();
        // 消费者的消费进度
        final Sequence[] processorSequences = new Sequence[eventHandlers.length];
        // 1. 根据依赖的消费者进度数组生产本次消费者屏障
        final SequenceBarrier barrier = ringBuffer.newBarrier(barrierSequences);

        for (int i = 0, eventHandlersLength = eventHandlers.length; i < eventHandlersLength; i++)
        {
            final EventHandler<? super T> eventHandler = eventHandlers[i];
            // 封装消费者(消费者屏障，消费事件处理逻辑)
            final BatchEventProcessor<T> batchEventProcessor =
                new BatchEventProcessor<>(ringBuffer, barrier, eventHandler);
            // 异常处理
            if (exceptionHandler != null)
            {
                batchEventProcessor.setExceptionHandler(exceptionHandler);
            }
            // 2. consumerRepository 消费者容器
            consumerRepository.add(batchEventProcessor, eventHandler, barrier);
            processorSequences[i] = batchEventProcessor.getSequence();
        }

        //3. ringbuffer保存着消费者的最慢消费进度 需要更新
        updateGatingSequencesForNextInChain(barrierSequences, processorSequences);

        return new EventHandlerGroup<>(this, consumerRepository, processorSequences);
    }

    //1.
    //AbstractSequencer
    public SequenceBarrier newBarrier(Sequence... sequencesToTrack)
    {
        return new ProcessingSequenceBarrier(this, waitStrategy, cursor, sequencesToTrack);
    }

    ProcessingSequenceBarrier(
        final Sequencer sequencer,
        final WaitStrategy waitStrategy,
        final Sequence cursorSequence,
        final Sequence[] dependentSequences)
    {
        //生产者
        this.sequencer = sequencer;
        //阻塞策略
        this.waitStrategy = waitStrategy;
        // SingleProducerSequencer-> 已被分配且发布的最大位置
        // MultiProducerSequencer-> 
        this.cursorSequence = cursorSequence;
        if (0 == dependentSequences.length)
        {
            dependentSequence = cursorSequence;
        }
        else
        {
            dependentSequence = new FixedSequenceGroup(dependentSequences);
        }
    }

    //2.
    // ConsumerRepository
    // 三个维度存放消费者 a.处理逻辑->消费者 b.消费者进度->消费者 c.消费者队列 (EventProcessorInfo是ConsumerInfo接口的实现)
    private final Map<EventHandler<?>, EventProcessorInfo<T>> eventProcessorInfoByEventHandler =
        new IdentityHashMap<>();
    private final Map<Sequence, ConsumerInfo> eventProcessorInfoBySequence =
        new IdentityHashMap<>();
    private final Collection<ConsumerInfo> consumerInfos = new ArrayList<>();

    public void add(
        final EventProcessor eventprocessor,
        final EventHandler<? super T> handler,
        final SequenceBarrier barrier)
    {
        final EventProcessorInfo<T> consumerInfo = new EventProcessorInfo<>(eventprocessor, handler, barrier);
        eventProcessorInfoByEventHandler.put(handler, consumerInfo);
        eventProcessorInfoBySequence.put(eventprocessor.getSequence(), consumerInfo);
        consumerInfos.add(consumerInfo);
    }

    //3. Disruptor
    private void updateGatingSequencesForNextInChain(final Sequence[] barrierSequences, final Sequence[] processorSequences)
    {
        if (processorSequences.length > 0)
        {
            // 将新增的消费者进度全部加入
            ringBuffer.addGatingSequences(processorSequences);
            // 如果消费者有依赖关系 将被依赖的消费者从最慢数组中删除，因为被依赖的消费者进度始终 > 需要依赖的消费者
            for (final Sequence barrierSequence : barrierSequences)
            {
                ringBuffer.removeGatingSequence(barrierSequence);
            }
            // 标记被依赖的消费者不为依赖链路上终点
            consumerRepository.unMarkEventProcessorsAsEndOfChain(barrierSequences);
        }
    }

    // RingBuffer.addGatingSequences->AbstractSequencer.addGatingSequences->
    // SequenceGroups.addSequences
    // AtomicReferenceFieldUpdater - 原子更新字段
        static <T> void addSequences(
        final T holder,
        final AtomicReferenceFieldUpdater<T, Sequence[]> updater,
        final Cursored cursor,
        final Sequence... sequencesToAdd)
    {
        long cursorSequence;
        Sequence[] updatedSequences;
        Sequence[] currentSequences;

        do
        {
            currentSequences = updater.get(holder);
            updatedSequences = copyOf(currentSequences, currentSequences.length + sequencesToAdd.length);
            cursorSequence = cursor.getCursor();

            int index = currentSequences.length;
            for (Sequence sequence : sequencesToAdd)
            {
                sequence.set(cursorSequence);
                updatedSequences[index++] = sequence;
            }
        }
        while (!updater.compareAndSet(holder, currentSequences, updatedSequences));
        // todo 为什么更新两次
        cursorSequence = cursor.getCursor();
        for (Sequence sequence : sequencesToAdd)
        {
            sequence.set(cursorSequence);
        }
    }
```

## 启动容器
```java
    public RingBuffer<T> start()
    {
        checkOnlyStartedOnce();
        // 遍历消费者 每个消费者新建新的线程执行事件处理方法
        for (final ConsumerInfo consumerInfo : consumerRepository)
        {
            consumerInfo.start(executor);
        }

        return ringBuffer;
    }

    // BatchEventProcessor.run->BatchEventProcessor.processEvents
    private void processEvents()
    {
        T event = null;
        // 下一个消费位置 nextSequence初始化 -1
        long nextSequence = sequence.get() + 1L;

        while (true)
        {
            try
            {
                // 查询可消费最大位置
                final long availableSequence = sequenceBarrier.waitFor(nextSequence);
                if (batchStartAware != null)
                {
                    batchStartAware.onBatchStart(availableSequence - nextSequence + 1);
                }

                while (nextSequence <= availableSequence)
                {
                    // 范围对应位置的事件
                    event = dataProvider.get(nextSequence);
                    // 处理事件
                    eventHandler.onEvent(event, nextSequence, nextSequence == availableSequence);
                    nextSequence++;
                }

                sequence.set(availableSequence);
            }
            // 除了TimeoutException和AlertException异常，其他异常默认跳过事件
            catch (final TimeoutException e)
            {
                notifyTimeout(sequence.get());
            }
            catch (final AlertException ex)
            {
                if (running.get() != RUNNING)
                {
                    break;
                }
            }
            catch (final Throwable ex)
            {
                // 消费抛出异常 可自定义异常处理逻辑
                exceptionHandler.handleEventException(ex, nextSequence, event);
                sequence.set(nextSequence);
                nextSequence++;
            }
        }
    }

    // ProcessingSequenceBarrier
    public long waitFor(final long sequence)
        throws AlertException, InterruptedException, TimeoutException
    {
        checkAlert();

        long availableSequence = waitStrategy.waitFor(sequence, cursorSequence, dependentSequence, this);

        if (availableSequence < sequence)
        {
            return availableSequence;
        }

        return sequencer.getHighestPublishedSequence(sequence, availableSequence);
    }
```

## 等待策略
```java
public interface WaitStrategy
{
    // sequence 申请的位置
    // cursor 当前发布最大位置
    // SequenceBarrier 依赖消费者消费进度
    long waitFor(long sequence, Sequence cursor, Sequence dependentSequence, SequenceBarrier barrier)
        throws AlertException, InterruptedException, TimeoutException;

    void signalAllWhenBlocking();
}
```
### BlockingWaitStrategy
当容器中没有事件时阻塞线程，有事件产生后唤醒线程，若要目标位置依赖的消费者还没消费，自旋等待。
```java
public final class BlockingWaitStrategy implements WaitStrategy
{
    private final Lock lock = new ReentrantLock();
    private final Condition processorNotifyCondition = lock.newCondition();

    @Override
    public long waitFor(long sequence, Sequence cursorSequence, Sequence dependentSequence, SequenceBarrier barrier)
        throws AlertException, InterruptedException
    {
        long availableSequence;
        if (cursorSequence.get() < sequence)
        {
            lock.lock();
            try
            {
                // 申请的位置不能大于已发布的最大位置
                while (cursorSequence.get() < sequence)
                {
                    barrier.checkAlert();
                    // 阻塞线程
                    processorNotifyCondition.await();
                }
            }
            finally
            {
                lock.unlock();
            }
        }
        // 申请的位置不能大于依赖的消费者最慢进度
        while ((availableSequence = dependentSequence.get()) < sequence)
        {
            barrier.checkAlert();
            // 自旋等待
            // JDK9以上->会优化
            // JDK9以下->无差别循环，很耗费CPU
            ThreadHints.onSpinWait();
        }

        return availableSequence;
    }

    @Override
    public void signalAllWhenBlocking()
    {
        lock.lock();
        try
        {
            processorNotifyCondition.signalAll();
        }
        finally
        {
            lock.unlock();
        }
    }
}
```

### TimeoutBlockingWaitStrategy
在BlockingWaitStrategy策略的阻塞阶段加入超时机制。

### BusySpinWaitStrategy
不阻塞，若要目标位置依赖的消费者还没消费，自旋等待。
```java
public final class BusySpinWaitStrategy implements WaitStrategy
{
    @Override
    public long waitFor(
        final long sequence, Sequence cursor, final Sequence dependentSequence, final SequenceBarrier barrier)
        throws AlertException, InterruptedException
    {
        long availableSequence;

        while ((availableSequence = dependentSequence.get()) < sequence)
        {
            barrier.checkAlert();
            ThreadHints.onSpinWait();
        }

        return availableSequence;
    }

    @Override
    public void signalAllWhenBlocking()
    {
    }
}
```

### LiteBlockingWaitStrategy
单消费者模式下阻塞模式的优化，在产生事件后判断消费者是否在等待，若没有则不需要执行唤醒操作。
```java
public final class LiteBlockingWaitStrategy implements WaitStrategy
{
    private final Lock lock = new ReentrantLock();
    private final Condition processorNotifyCondition = lock.newCondition();
    // 记录消费者是否被阻塞
    private final AtomicBoolean signalNeeded = new AtomicBoolean(false);

    @Override
    public long waitFor(long sequence, Sequence cursorSequence, Sequence dependentSequence, SequenceBarrier barrier)
        throws AlertException, InterruptedException
    {
        long availableSequence;
        if (cursorSequence.get() < sequence)
        {
            lock.lock();

            try
            {
                do
                {
                    // 阻塞时设置标识
                    signalNeeded.getAndSet(true);

                    if (cursorSequence.get() >= sequence)
                    {
                        break;
                    }

                    barrier.checkAlert();
                    processorNotifyCondition.await();
                }
                while (cursorSequence.get() < sequence);
            }
            finally
            {
                lock.unlock();
            }
        }

        while ((availableSequence = dependentSequence.get()) < sequence)
        {
            barrier.checkAlert();
            ThreadHints.onSpinWait();
        }

        return availableSequence;
    }

    @Override
    public void signalAllWhenBlocking()
    {
        // 通过标志判断是否需要执行唤醒操作
        if (signalNeeded.getAndSet(false))
        {
            lock.lock();
            try
            {
                processorNotifyCondition.signalAll();
            }
            finally
            {
                lock.unlock();
            }
        }
    }
}
```

### LiteTimeoutBlockingWaitStrategy
在LiteBlockingWaitStrategy策略的阻塞阶段加入超时。

### YieldingWaitStrategy
检查依赖的消费者进度，若要目标位置依赖的消费者还没消费，自旋等待，超过重试次数，让出cpu资源。
```java
public final class YieldingWaitStrategy implements WaitStrategy
{
    private static final int SPIN_TRIES = 100;

    @Override
    public long waitFor(
        final long sequence, Sequence cursor, final Sequence dependentSequence, final SequenceBarrier barrier)
        throws AlertException, InterruptedException
    {
        long availableSequence;
        int counter = SPIN_TRIES;

        while ((availableSequence = dependentSequence.get()) < sequence)
        {
            counter = applyWaitMethod(barrier, counter);
        }

        return availableSequence;
    }

    @Override
    public void signalAllWhenBlocking()
    {
    }

    private int applyWaitMethod(final SequenceBarrier barrier, int counter)
        throws AlertException
    {
        barrier.checkAlert();

        if (0 == counter)
        {
            Thread.yield();
        }
        else
        {
            --counter;
        }

        return counter;
    }
}
```

### SleepingWaitStrategy
检查依赖的消费者进度，若要目标位置依赖的消费者还没消费，重试次数超过100，自旋等待，大于0小于100，让出cpu资源，小于等于0，睡眠一定时间，是一个响应时间和资源消耗的平衡(重试->让出cpu->睡眠)。
```java
public final class SleepingWaitStrategy implements WaitStrategy
{
    // 默认重试次数200 睡眠时间100纳秒
    private static final int DEFAULT_RETRIES = 200;
    private static final long DEFAULT_SLEEP = 100;

    private final int retries;
    private final long sleepTimeNs;

    public SleepingWaitStrategy()
    {
        this(DEFAULT_RETRIES, DEFAULT_SLEEP);
    }

    public SleepingWaitStrategy(int retries)
    {
        this(retries, DEFAULT_SLEEP);
    }

    public SleepingWaitStrategy(int retries, long sleepTimeNs)
    {
        this.retries = retries;
        this.sleepTimeNs = sleepTimeNs;
    }

    @Override
    public long waitFor(
        final long sequence, Sequence cursor, final Sequence dependentSequence, final SequenceBarrier barrier)
        throws AlertException
    {
        long availableSequence;
        int counter = retries;

        while ((availableSequence = dependentSequence.get()) < sequence)
        {
            counter = applyWaitMethod(barrier, counter);
        }

        return availableSequence;
    }

    @Override
    public void signalAllWhenBlocking()
    {
    }

    private int applyWaitMethod(final SequenceBarrier barrier, int counter)
        throws AlertException
    {
        barrier.checkAlert();

        // 重试次数 >100 自旋
        //         >0 <100 让出cpu资源
        //         <=0 睡眠
        if (counter > 100)
        {
            --counter;
        }
        else if (counter > 0)
        {
            --counter;
            Thread.yield();
        }
        else
        {
            LockSupport.parkNanos(sleepTimeNs);
        }

        return counter;
    }
}
```
### PhasedBackoffWaitStrategy(yieldTimeoutNanos>spinTimeoutNanos)
该策略分为四个阶段(yieldTimeoutNanos=3000，spinTimeoutNanos=1000)
1. 自旋10000次
2. 带超时自旋spinTimeoutNanos秒
3. 让出cpu资源 yieldTimeoutNanos-spinTimeoutNanos 秒
4. 计时超过yieldTimeoutNanos后采用备用策略fallbackStrategy(BlockingWaitStrategy、LiteBlockingWaitStrategy、SleepingWaitStrategy)
```java
public final class PhasedBackoffWaitStrategy implements WaitStrategy
{
    //
    private static final int SPIN_TRIES = 10000;
    private final long spinTimeoutNanos;
    private final long yieldTimeoutNanos;
    // 备用策略
    private final WaitStrategy fallbackStrategy;

    //....

    @Override
    public long waitFor(long sequence, Sequence cursor, Sequence dependentSequence, SequenceBarrier barrier)
        throws AlertException, InterruptedException, TimeoutException
    {
        long availableSequence;
        long startTime = 0;
        int counter = SPIN_TRIES;

        do
        {
            if ((availableSequence = dependentSequence.get()) >= sequence)
            {
                return availableSequence;
            }
            // 重试10000
            if (0 == --counter)
            {
                // 计时
                if (0 == startTime)
                {
                    startTime = System.nanoTime();
                }
                else
                {
                    long timeDelta = System.nanoTime() - startTime;
                    // 超过yieldTimeoutNanos采用备用策略
                    if (timeDelta > yieldTimeoutNanos)
                    {
                        return fallbackStrategy.waitFor(sequence, cursor, dependentSequence, barrier);
                    }
                    // 超过spinTimeoutNanos让出cpu资源
                    else if (timeDelta > spinTimeoutNanos)
                    {
                        Thread.yield();
                    }
                }
                counter = SPIN_TRIES;
            }
        }
        while (true);
    }

    @Override
    public void signalAllWhenBlocking()
    {
        fallbackStrategy.signalAllWhenBlocking();
    }
}

```
## Questions
### cursor和nextValue的区别?
cursor在单生产者和多生产者含义不同，nextValue是单生产者独有的变量
* 在单生产者中，cursor表示已分配且已发布的最大位置，nextValue表示已分配的最大位置
* 在多生产者中，cursor表示已分配的最大位置

[disruptor笔记之七：等待策略](https://blog.csdn.net/boling_cavalry/article/details/117608051)   
[Java多线程内存读写（一） —— 从AtomicReferenceFieldUpdater开始理解volatile与unsafe](https://juejin.cn/post/7145049623042736142)    
[高性能队列——Disruptor](https://tech.meituan.com/2016/11/18/disruptor.html)