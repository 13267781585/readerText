## ThreadPoolExecutor 源码

#### 保证线程安全方式
* CAS -> 用于线程数量的设置
* ReentrantLock -> 用于创建Worker线程
* Worker 实现aqs，保证每次只能执行一个任务(独占锁)
* mainLock 全局锁，在操作线程池一些变量时候需要加锁，例如统计未完成任务数，已完成任务数，活跃的任务(判断Worker的锁是否被获取，被获取说明正在执行任务)
#### 线程池记录变量
* 线程池个数和状态
```java
//用int变量高3位记录状态 int位数-3 表示线程个数
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));

//线程池状态
    private static final int COUNT_BITS = Integer.SIZE - 3;

    /*
     *   RUNNING:  接受新的任务和处理队列中的任务
     *   SHUTDOWN: 不接受新的任务但是处理队列中的任务
     *   STOP:     不接受新的任务 不再继续处理队列中的任务 中断正在进行的任务
     *   TIDYING:  全部任务被中断 工作线程为0 将要调用 terminated 方法
     *   TERMINATED: terminated() 调用完成

     * RUNNING -> SHUTDOWN
     *   显式调用 shutdown() 方法，或者隐式调用了 finalize()，它里面调用了 shutdown() 方法
     * (RUNNING or SHUTDOWN) -> STOP
     *    显示调用 shutdownNow()
     * SHUTDOWN -> TIDYING
     *    队列和线程池为空
     * STOP -> TIDYING
     *    线程池为空(线程释放)
     * TIDYING -> TERMINATED
     *     terminated() 执行完成
    */
    private static final int RUNNING    = -1 << COUNT_BITS;
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    private static final int STOP       =  1 << COUNT_BITS;
    private static final int TIDYING    =  2 << COUNT_BITS;
    private static final int TERMINATED =  3 << COUNT_BITS;
```


#### 运行流程
```java
public void execute(Runnable command) {

    if (command == null)
        throw new NullPointerException();

    //获取当前线程池的状态+线程个数变量的组合值
    int c = ctl.get();

    //当前线程池线程个数是否小于corePoolSize,小于则开启新线程运行
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }

    //添加任务到阻塞队列
    if (isRunning(c) && workQueue.offer(command)) {

        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);

        //如果当前线程池线程空，则添加一个线程
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    //如果队列满了，则新增线程，新增失败则执行拒绝策略
    else if (!addWorker(command, false))
        reject(command);
}
```

#### 新增线程
```java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        //循环cas增加线程个数
        for (;;) {
            int wc = workerCountOf(c);
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
        }
    }

    //到这里说明cas成功了
    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        //创建worker
        final ReentrantLock mainLock = this.mainLock;
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {

            //加独占锁，为了workers同步，因为可能多个线程调用了线程池的execute方法。
            mainLock.lock();
            try {

                //重新检查线程池状态，为了避免在获取锁前调用了shutdown接口
                int c = ctl.get();
                int rs = runStateOf(c);

                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) 
                        throw new IllegalThreadStateException();
                    //添加任务
                    workers.add(w);
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            //添加成功则启动任务
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
```

#### 运行线程
```java
    //新增线程run方法
    public void run() {
        runWorker(this);
    }



    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        /**
            private final class Worker
                extends AbstractQueuedSynchronizer
                    implements Runnable
            Worker 实现了aqs保证一个worker一次只能执行一个任务
             //status设置为0，允许中断
        **/
        w.unlock();
        boolean completedAbruptly = true;
        try {
           //Worker线程循环拿任务执行
            while (task != null || (task = getTask()) != null) {

                 
                w.lock();
               ...
                try {
                    //任务执行前干一些事情
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        task.run();//执行任务
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        //任务执行完毕后干一些事情
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    //统计当前worker完成了多少个任务
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {

            //执行清工作
            processWorkerExit(w, completedAbruptly);
        }
    }
```

#### shutdown() -> 不接受新的任务，队列现有的任务继续执行
```java
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        //(12)权限检查
        checkShutdownAccess();

        //(13)设置当前线程池状态为SHUTDOWN，如果已经是SHUTDOWN则直接返回
        advanceRunState(SHUTDOWN);

        //(14)设置中断标志
        interruptIdleWorkers();
        onShutdown(); 
    } finally {
        mainLock.unlock();
    }
    //(15)尝试状态变为TERMINATED
    tryTerminate();
}



private void interruptIdleWorkers(boolean onlyOne) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        for (Worker w : workers) {
            Thread t = w.thread;
            //如果工作线程没有被中断，并且没有正在运行则设置中断
            //获取锁来判断是否现在运行任务
            if (!t.isInterrupted() && w.tryLock()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                } finally {
                    w.unlock();
                }
            }
            if (onlyOne)
                break;
        }
    } finally {
        mainLock.unlock();
    }
}
```

#### shutdownNow() -> 丢弃任务 终止正在运行任务
```java
public List<Runnable> shutdownNow() {
    List<Runnable> tasks;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();//（16)权限检查
        advanceRunState(STOP);//(17) 设置线程池状态为stop
        interruptWorkers();//(18)中断所有线程
        tasks = drainQueue();//（19）移动队列任务到tasks
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
    return tasks;
}

private void interruptWorkers() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers)
                w.interruptIfStarted();
        } finally {
            mainLock.unlock();
        }
    }

        void interruptIfStarted() {
            Thread t;
            if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                try {
                    //采用中断线程的方式退出任务的执行，在runWorker会检查标志位
                    t.interrupt();
                } catch (SecurityException ignore) {
                }
            }
        }
```


#### 线程池如何维护线程在空闲时间内释放
```java

    //Worker 循环获取任务 获取得到执行，获取null退出 释放线程
private Runnable getTask() {
        boolean timedOut = false; 

        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }

            int wc = workerCountOf(c);

            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
            //判断是否超时 超时直接返回null
            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
                //poll阻塞一定时间内获取任务，时间设置为最大空闲时间，这段时间内结束没有返回任务说明到达最大空闲时间 返回null 退出线程
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
```


#### 调用submit方法执行callable接口
```java
    public <T> Future<T> submit(Callable<T> task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task);
        execute(ftask);
        return ftask;
    }

    //将callable接口包装FutureTask 然后正常执行
    protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        return new FutureTask<T>(callable);
    }

    //FutureTask类方法 调用 get() 方法获取结果
    public V get() throws InterruptedException, ExecutionException {
        int s = state;
        //判断任务执行情况 没有完成阻塞线程
        if (s <= COMPLETING)
            s = awaitDone(false, 0L);
        return report(s);
    }

    private int awaitDone(boolean timed, long nanos)
        throws InterruptedException {
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        WaitNode q = null;
        boolean queued = false;
        for (;;) {
            if (Thread.interrupted()) {
                removeWaiter(q);
                throw new InterruptedException();
            }

            int s = state;
            if (s > COMPLETING) {
                if (q != null)
                    q.thread = null;
                return s;
            }
            else if (s == COMPLETING) // cannot time out yet
                Thread.yield();
            else if (q == null)
                q = new WaitNode();
            else if (!queued)
                queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                     q.next = waiters, q);
            else if (timed) {
                nanos = deadline - System.nanoTime();
                if (nanos <= 0L) {
                    removeWaiter(q);
                    return state;
                }
                //阻塞线程
                LockSupport.parkNanos(this, nanos);
            }
            else
                LockSupport.park(this);
        }
    }


    //线程run方法 执行获取结构后唤醒线程
    public void run() {
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    //调用 call 方法，执行完成返回结果
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    //抛出异常 唤醒线程
                    setException(ex);
                }
                //设置结果 唤醒线程
                if (ran)
                    set(result);
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }

    //在
    private void finishCompletion() {
        // assert state > COMPLETING;
        for (WaitNode q; (q = waiters) != null;) {
            if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
                for (;;) {
                    Thread t = q.thread;
                    if (t != null) {
                        q.thread = null;
                        //唤醒线程
                        LockSupport.unpark(t);
                    }
                    WaitNode next = q.next;
                    if (next == null)
                        break;
                    q.next = null; // unlink to help gc
                    q = next;
                }
                break;
            }
        }

        done();

        callable = null;        // to reduce footprint
    }

```