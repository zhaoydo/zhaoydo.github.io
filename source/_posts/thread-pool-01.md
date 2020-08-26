---
title: JAVA线程池-源码分析
date: 2020-07-13 11:46:20
tags:
---
以ThreadPoolExecutor为例
<!--more--> 
# 整体结构
1. Executor 接口，只封装了execute方法
2. ExecutorServices 接口，扩展了很多方法
3. AbastractExecutorService 抽象类
4. ThreadPoolExecutor 线程池类
![](http://122.51.119.99:9000/blog/ThreadPoolExecutor.png)

# Executor
代码很简单，一个execute方法，传入Runnable对象。 用于执行任务
```
public interface Executor {
    void execute(Runnable command);
}
```
# ExecutorService
一般定义线程池的时候，用ExecutorService, 接口定义的方法比较全
```
//示例，创建一个线程数为10的线程池
ExecutorService executorService = Executors.newFixedThreadPool(10);
```
看源码，也很简单，定义了提交任务的方法和关闭终止的方法
```

public interface ExecutorService extends Executor {

    /**
     * 关闭线程池，不接受新任务提交
     * 正在运行和队列中的任务执行完后，关闭线程池
     * showdown方法不会等待已提交的任务执行完，需要等待线程池任务都执行完毕，用awaitTermination()方法
     */
    void shutdown();

    /**
     * 和shutdown的区别是会尝试关闭正在运行和队列中的任务
     */
    List<Runnable> shutdownNow();

    /**
     * 线程池是否关闭
     */
    boolean isShutdown();

    /**
     * 线程池是否停止，在shutdown()/shutdown()之后，并且所有任务都执行完毕
     */
    boolean isTerminated();

    /**
     * 阻塞等待所有任务执行完成/线程interrupted/超时,并且isTerminated()=true
     * 超时返回false， 其他返回true
     */
    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;

    /**
     * 提交一个带返回值的任务（或者抛出Exception）
     * Future.get()方法将阻塞获取返回值
     */
    <T> Future<T> submit(Callable<T> task);

    /**
     * 提交一个带返回值的任务， 因为Runnable不返回结果，需要额外的参数T result
     * @throws NullPointerException if the task is null
     */
    <T> Future<T> submit(Runnable task, T result);

    /**
     * 提交一个Runnable任务， 任务的Future不带返回值
     * @param task
     * @return
     */
    Future<?> submit(Runnable task);

    /**
     * 执行一批任务，返回Futures的list
     */
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;

    /**
     * 同上，增加了超时时间
     */
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;

    /**
     * 提交一批任务，只要1个任务执行完就可以返回
     */
    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException;

    /**
     * 同上，增加了时间参数
     */
    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}

```
# AbstractExecutorService

## FutureTask
Future是submit方法的返回类型，一般用于异步执行任务后，阻塞获取任务返回值，FutureTask是Future的实现类，用的比较多  
FutureTask的整体结构  
![FutureTask](http://122.51.119.99:9000/blog/FutureTask.png)
FutureTask实现了RunnableFutue，间接实现了Runnable、Future接口  
RunnableFutue代码  
```
public interface RunnableFuture<V> extends Runnable, Future<V>
```
 RunnableFuture继承了两者特性，能实现执行任务(run)、获取任务返回结果(get)以及中断任务(cancel)的功能
## AbstractExecutorService
抽象类，实现了几个主要接口方法，submit、newTaskFor、invokeAll、invokeAny。  
很简单，直接看代码
```java
public abstract class AbstractExecutorService implements ExecutorService {

    /**
     * 任务（Runnable）和返回值（T）封装为RunnableFuture
     * 包装成RunnableFuture的子类FutureTask返回,便于调用Future.get()阻塞获取返回结果
     * 备注，Runnable+T => Callable<T>(CallableAdapter<T>) => FutureTask
     */
    protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
        return new FutureTask<T>(runnable, value);
    }

    /**
     * 用带返回值的任务Callable，封装为RunnableFuture
     * 意义同上
     */
    protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        return new FutureTask<T>(callable);
    }

    /**
     * 提交任务，不带返回值
     */
    public Future<?> submit(Runnable task) {
        if (task == null) throw new NullPointerException();
        //任务包装成FutureTask
        RunnableFuture<Void> ftask = newTaskFor(task, null);
        //执行任务 注意这里只调用了接口方法，没有实现execute
        execute(ftask);
        return ftask;
    }

    /**
     * 提交带返回值的任务
     */
    public <T> Future<T> submit(Runnable task, T result) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task, result);
        execute(ftask);
        return ftask;
    }

    /**
     * 提交带返回值的任务
     */
    public <T> Future<T> submit(Callable<T> task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task);
        execute(ftask);
        return ftask;
    }

    /**
     * invokeAny的主要实现
     * 执行一批任务，返回第一个正常执行完成的任务结果，其余任务取消
     */
    private <T> T doInvokeAny(Collection<? extends Callable<T>> tasks,
                              boolean timed, long nanos)
        throws InterruptedException, ExecutionException, TimeoutException {
        //任务非空校验
        if (tasks == null)
            throw new NullPointerException();
        //任务数量（未提交的任务数）
        int ntasks = tasks.size();
        //任务非空校验
        if (ntasks == 0)
            throw new IllegalArgumentException();
        ArrayList<Future<T>> futures = new ArrayList<Future<T>>(ntasks);
        //ThreadPoolExecutor封装了一层，任务完成后结果保存到内部的completionQueue队列中
        ExecutorCompletionService<T> ecs =
            new ExecutorCompletionService<T>(this);

        try {
            //记录任务异常信息，如果doInvokeAny没有得到任何一个结果，可以跑出这里得到的最后一个任务的异常
            ExecutionException ee = null;
            final long deadline = timed ? System.nanoTime() + nanos : 0L;
            Iterator<? extends Callable<T>> it = tasks.iterator();

            //提交第一个任务， 剩下的任务在下面for循环中提交
            futures.add(ecs.submit(it.next()));
            //因为提交了任务，任务数-1
            --ntasks;
            /**正在执行的任务数*/
            int active = 1;
            //for循环提交任务
            for (;;) {
                //尝试获取任务（poll为非阻塞形式），如果已完成返回f！=null
                Future<T> f = ecs.poll();
                //当前没有任务完成
                if (f == null) {
                    if (ntasks > 0) {
                        //未提交的任务数>0,继续提交
                        --ntasks;
                        futures.add(ecs.submit(it.next()));
                        ++active;
                    }
                    else if (active == 0)
                        //提交完所有任务、并且所有任务都抛异常的时候，走这里的break跳出循环
                        break;
                    else if (timed) {
                        //提交完所有任务，设置了超时时间 ，阻塞等待
                        f = ecs.poll(nanos, TimeUnit.NANOSECONDS);
                        if (f == null)
                            throw new TimeoutException();
                        nanos = deadline - System.nanoTime();
                    }
                    else
                        //提交完所有任务，没有超时时间，阻塞等待
                        f = ecs.take();
                }
                //有任务完成，如果正常完成，return，如果抛异常则记录到ee，进入下一个循环
                if (f != null) {
                    --active;
                    try {
                        return f.get();
                    } catch (ExecutionException eex) {
                        ee = eex;
                    } catch (RuntimeException rex) {
                        ee = new ExecutionException(rex);
                    }
                }
                //for循环结束
            }
            //进入到这里一般是所有任务都抛出异常了
            if (ee == null)
                ee = new ExecutionException();
            throw ee;

        } finally {
            //最后取消任务，只要1个任务执行完成就可以了
            for (int i = 0, size = futures.size(); i < size; i++)
                futures.get(i).cancel(true);
        }
        /**
         * 总结一下上面的代码
         * 1、首先提交一个任务
         * 2、进入for循环，如果得到的future为空，则依次判断未提交任务>0、正在执行任务==0，阻塞获取结果（包括有timeout的）
         * 3、获取到结果后，所有任务都尝试取消，不用执行了
         * 比较优雅的地方在于通过ExecutorCompletionService维护了一个任务完成的阻塞队列completionQueue，
         */
    }

    /**
     * 同上不解释
     * @param tasks
     * @param <T>
     * @return
     * @throws InterruptedException
     * @throws ExecutionException
     */
    public <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException {
        try {
            return doInvokeAny(tasks, false, 0);
        } catch (TimeoutException cannotHappen) {
            assert false;
            return null;
        }
    }

    /**
     * 同上不解释，增加了超时时间
     */
    public <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                           long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException {
        return doInvokeAny(tasks, true, unit.toNanos(timeout));
    }

    /**
     * 执行所有任务
     */
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException {
        //参数校验
        if (tasks == null)
            throw new NullPointerException();
        ArrayList<Future<T>> futures = new ArrayList<Future<T>>(tasks.size());
        boolean done = false;
        try {
            //for循环提交任务
            for (Callable<T> t : tasks) {
                RunnableFuture<T> f = newTaskFor(t);
                futures.add(f);
                execute(f);
            }
            //for循环获取结果
            for (int i = 0, size = futures.size(); i < size; i++) {
                Future<T> f = futures.get(i);
                //如果未执行完毕，阻塞获取结果
                // TODO 没看懂，直接用future.get就好了啊,难道是future.get()有性能开销？
                if (!f.isDone()) {
                    try {
                        f.get();
                    } catch (CancellationException ignore) {
                    } catch (ExecutionException ignore) {
                    }
                }
            }
            done = true;
            return futures;
        } finally {
            if (!done)
                for (int i = 0, size = futures.size(); i < size; i++)
                    futures.get(i).cancel(true);
        }
    }

    /**
     * 执行所有任务，带时超时时间
     */
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                         long timeout, TimeUnit unit)
        throws InterruptedException {
        if (tasks == null)
            throw new NullPointerException();
        long nanos = unit.toNanos(timeout);
        //定义要返回的Future数组
        ArrayList<Future<T>> futures = new ArrayList<Future<T>>(tasks.size());
        boolean done = false;
        try {
            for (Callable<T> t : tasks)
                futures.add(newTaskFor(t));

            final long deadline = System.nanoTime() + nanos;
            final int size = futures.size();

            // Interleave time checks and calls to execute in case
            // executor doesn't have any/much parallelism.
            for (int i = 0; i < size; i++) {
                execute((Runnable)futures.get(i));
                //每次循环，nanos=剩余时间
                nanos = deadline - System.nanoTime();
                //每次提交任务，检测超时返回
                if (nanos <= 0L)
                    return futures;
            }

            for (int i = 0; i < size; i++) {
                Future<T> f = futures.get(i);
                if (!f.isDone()) {
                    //超时
                    if (nanos <= 0L)
                        return futures;
                    try {
                        f.get(nanos, TimeUnit.NANOSECONDS);
                    } catch (CancellationException ignore) {
                    } catch (ExecutionException ignore) {
                    } catch (TimeoutException toe) {
                        return futures;
                    }
                    nanos = deadline - System.nanoTime();
                }
            }
            done = true;
            return futures;
        } finally {
            if (!done)
                //任务超时时会进入这里， 此时取消任务
                for (int i = 0, size = futures.size(); i < size; i++)
                    futures.get(i).cancel(true);
        }
    }

}
```

# ThreadPoolExecutor
ThreadPoolExecutor 是JDK线程池的完整实现，实现了任务提交、线程管理，任务队列管理，任务监控等功能

## 整体结构
看一眼构造方法,如果是只使用线程池，用这个就够了，后面的的源码中，会一直贯穿这些参数  
```java
/**
 * 构造方法
 * @param corePoolSize 核心线程数
 * @param maximumPoolSize 最大线程数
 * @param keepAliveTime 线程空闲时间， 线程空闲时，实际线程数超出核心线程数时，超出线程空闲时间后，将被销毁
 * @param unit 时间单位
 * @param workQueue 缓存队列
 * @param threadFactory 线程工厂，在这里可以自定义一些线程
 * @param handler 四种任务拒绝策略
 */
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.acc = System.getSecurityManager() == null ?
            null :
            AccessController.getContext();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```
一些需要关注的变量  
```java
//线程池全局锁，源码中会一直用到
private final ReentrantLock mainLock = new ReentrantLock();
//workers 线程池内一个个Worker的Set集合，每个Worker对应池内的一个线程
private final HashSet<Worker> workers = new HashSet<Worker>();
//AQS的概念还没搞懂 TODO
private final Condition termination = mainLock.newCondition();
//是否允许关闭核心线程，如果为true，核心线程在超过keepAliveTime后也会关闭
private volatile boolean allowCoreThreadTimeOut;
//工作队列，对应构造函数中的workQueue
private final BlockingQueue<Runnable> workQueue;
```
## 线程池状态
线程池中维护了一个32位数字ctl，前三位表示线程池状态，后29位表示线程池数量  
直接看代码
```java
//线程池状态+线程池线程数， 32位数字，前三位表示状态，后29位表示线程数量
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
//29位二进制数，表示线程数
private static final int COUNT_BITS = Integer.SIZE - 3;
//线程数大小（000 11111111111111111111111111111）
//1<<29-1 = 2^29-1 = 536870911 线程池最大容量为536870911个，一般计算机都达不到这个数字
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

// runState is stored in the high-order bits
//ctl的32位数字中，0-3位存放线程池状态。 注意是线程池状态不是线程状态
//111 00000000000000000000000000000 运行中
private static final int RUNNING    = -1 << COUNT_BITS;
//000 00000000000000000000000000000 关闭中， 不接受新的任务提交，但是会继续执行已提交（运行中+队列中）的任务
private static final int SHUTDOWN   =  0 << COUNT_BITS;
//001 00000000000000000000000000000 停止中， 不接受新的任务提交，不处理队列中的任务，中断正在运行的任务
private static final int STOP       =  1 << COUNT_BITS;
//010 00000000000000000000000000000 所有任务都销毁了，workCount为0，线程池的状态转换为TIDYING后，会执行钩子方法terminated()
private static final int TIDYING    =  2 << COUNT_BITS;
//011 00000000000000000000000000000 terminated() 方法结束后，线程池的状态就会变成这个
private static final int TERMINATED =  3 << COUNT_BITS;

// Packing and unpacking ctl
//ctl的前三位，也就是线程池状态
private static int runStateOf(int c)     { return c & ~CAPACITY; }
//ctl的后29位，线程数量
private static int workerCountOf(int c)  { return c & CAPACITY; }
//线程状态rs, 工作线程数量wc 或操作， 组成32位ctl
private static int ctlOf(int rs, int wc) { return rs | wc; }
```

### 线程池状态定义
1. RUNNING 运行中，正常运行的的状态
2. SHUTDOWN 不接受新任务提交，但是会继续处理正在运行以及队列中的任务。 任务运行完之后关闭线程池
3. STOP 不接受新任务提交，不处理队列中的任务，中断正在执行的任务
4. TIDYING 所有任务都销毁了，workCount为0，线程池进入TIDYING后执行钩子方法terminated(),之后将线程池状态修改为TERMINATED
5. TERMINATED：terminated() 方法结束后，线程池的状态就会变成这个
### 各个状态转换
- RUNNING -> SHUTDOWN 调用了shutdown()方法
- (RUNNING/SHUTDOWN)-> STOP 调用了shutdownNow()方法
- SHUTDOWN -> TIDYING 调用tryTerminate()后，工作线程数为0&队列为空&SHUTDOWN状态，转换为TIDYING
- STOP  -> TIDYING 调用tryTerminate()后，工作线程数为0，转换为TIDYING
- TIDYING  -> TERMINATED 前面转换为TIDYING后，执行钩子方法terminated(),之后将线程池状态修改为TERMINATED

## Worker
ThreadPoolExecutor的内部类Worker  
Doug Lea将线程池中的线程包装成一个个Worker，每个Worker可以运行多个任务（Runnable），Worker数对应线程池中的线程数。  
中间还有一些AQS的内容，等我把AQS搞懂再说。  
代码，没啥内容，主要是构造方法中，用ThreadFactory来创建新线程  
```java
private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
    {
        private static final long serialVersionUID = 6138294804551838833L;

        //真正的线程
        final Thread thread;
        /** Initial task to run.  Possibly null. */
        Runnable firstTask;
        /** Per-thread task counter */
        //存放此线程完成的任务数
        volatile long completedTasks;

        /**
         * Creates with given first task and thread from ThreadFactory.
         * @param firstTask the first task (null if none)
         */
        /**
         * 构造方法，firstTask也可为空
         * 通过线程工厂创建一个线程
         * @param firstTask
         */
        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }

        /** Delegates main run loop to outer runWorker  */
        //调用外部的runWorker方法
        public void run() {
            runWorker(this);
        }

        // Lock methods
        //
        // The value 0 represents the unlocked state.
        // The value 1 represents the locked state.

        protected boolean isHeldExclusively() {
            return getState() != 0;
        }

        protected boolean tryAcquire(int unused) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        protected boolean tryRelease(int unused) {
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        public void lock()        { acquire(1); }
        public boolean tryLock()  { return tryAcquire(1); }
        public void unlock()      { release(1); }
        public boolean isLocked() { return isHeldExclusively(); }

        void interruptIfStarted() {
            Thread t;
            if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                }
            }
        }
    }
```
## execute
execute是线程池用的最多的方法，AbstractExecutorService中实现的submit方法最终也是调用execute方法  
execute并不直接执行任务，而是把任务添加到workQueue中或者新建一个Worker（addWorker）跑
看代码,看注释  
```java
/**
 * 执行任务
 * @param command
 */
public void execute(Runnable command) {
    //参数校验
    if (command == null)
        throw new NullPointerException();
    /**
     *
     * 总体分三步
     * 1、如果workCount<corePoolSize 尝试addWorker
     * 2、如果corePoolSize已满，尝试offer到workQueue队列中
     * 3、workQueue队列已满，尝试addWorker,如果workCount==maximumPoolSize,则失败，进入拒绝策略
     * 注意：执行一个任务，如果核心线程已满吗，先尝试加入workQueue,队列满之后，再尝试新建线程达到最大线程数。
     * 优先级：核心线程 > 缓存队列  > 最大线程数
     */
    int c = ctl.get();
    //1、如果工作线程数<核心线程数 直接添加一个worker，并把任务做作为Worker的第一个任务（firstTask）
    if (workerCountOf(c) < corePoolSize) {
        //添加成功，返回true，return
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    //运行到这里，说明当前线程数（workerCount）>=核心线程数

    //2、如果线程池在RUNNING状态， 任务添加到任务队列（workQueue）
    if (isRunning(c) && workQueue.offer(command)) {
        //进入这里，任务提交到workQueue

        int recheck = ctl.get();
        //再次判断线程池状态，如果非RUNNING，则移除已经入队的任务，进入拒绝处理（reject(command)）
        if (! isRunning(recheck) && remove(command))
            reject(command);
        //如果线程池状态为RUNNING，且线程数为0
        //（处理：任务已提交队列后，核心线程数为0的情况，这时候addWorker）
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    //3、前面workQueue队列满的情况下，进入到这里处理。
    //如果仍然失败，说明当前线程数以达到maxmumPoolSize
    else if (!addWorker(command, false))
		//调用具体的拒绝策略来处理
        reject(command);
}
```
## addWorker
如果当前workerCount小于核心线程数， 或者workQueue已满且workerCount小于最大线程数，则调用addWorker创建一个线程  
直接看源码  不停的判断状态，不太好理解，需要多看一下 
```java
/**
 * 创建并启用一个新线程（Worker）
 * @param firstTask 提交的第一个任务
 * @param core 是否为核心线程， 如果为true，workCount边界为corePoolSize,如果false则边界为maxmumpoolSize
 * @return 是否创建成功
 */
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        //线程状态
        int rs = runStateOf(c);
        /**
         * 如果线程已关闭并且满足
         * 下列条件之一，return false
         * 1、rs > SHUTDOWN 也就是 STOP, TIDYING, 或 TERMINATED
         * 2、firstTask !=null
         * 3、workQueue.isEmpty()
         * 分析：
         * 1、showdown时不允许提交任务，但是已提交的任务继续执行
         *  此时允许创建新线程来执行队列中的任务（rs==SHUTDOWN && firstTask ==null && workQueue.isNotEmpty()）
         * 2、rs > SHUTDOWN 也就是 STOP, TIDYING, 或 TERMINATED时，不允许提交任务，不创建线程。
         */
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        for (;;) {
            int wc = workerCountOf(c);
            //两个限制，
            // 1、不超过计算机最大线程数（CAPACITY），超过了29位二进制就无法表示了
            // 2、不超过corePoolSize or maximumPoolSize
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            //此时满足所有创建线程的条件，准备创建线程
            //cas给线程数+1，TODO 没理解这里为什么break
            if (compareAndIncrementWorkerCount(c))
                break retry;
            //重新读取ctl，如果此时线程池状态发生变更，重新最外层的for循环
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }
    //worker是否启动
    boolean workerStarted = false;
    //是否已经将worker加入到workers这个HashSet中
    boolean workerAdded = false;
    Worker w = null;
    try {
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            //获得整个线程池的全局锁
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                //
                //持有锁时，重新获取线程状态
                int rs = runStateOf(ctl.get());
                //运行中 or 线程关闭状态已提交的任务
                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    //加入到线程HashSet集合中
                    workers.add(w);
                    //更新线程数最大值largestPoolSize
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            if (workerAdded) {
                //开启
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        //如果线程没有启动，做一些清理工作
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
```
## addWorkerFailed
addWorkerFailed的处理  
代码很简单，删掉线程集合中的worker，workerCount-1
```java
/**
 * 1、workers中删掉worker
 * 2、workerCount-1
 * 3、调用tryTerminate尝试关闭线程池
 * @param w
 */
private void addWorkerFailed(Worker w) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        if (w != null)
            workers.remove(w);
        decrementWorkerCount();
        //注意是尝试关闭，也就是任何操作结束之后线程池可能被关闭后，就执行这个操作
        tryTerminate();
    } finally {
        mainLock.unlock();
    }
}
```
## runWorker
前面的Worker内部类实现的run方法，调用的是runWorker(this)
1. 取任务，循环从firstTask或者getTask()方法中返回，如果取不到任务，则进入worker销毁流程
2. 取到任务后task.run()，执行完毕之后，进入下一个循环
```java
/**
 * 实现Worker的run方法
 * 如果worker内有firstTask，执行firstTask；没有则从workQueue中取
 * @param w
 */
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        //循环获取任务，直到task不为空 TODO getTask决定了Worker何时被销毁
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            /**
             * 这里很难理解
             * 1、线程池停止(c>=STOP),则中断wt线程
             * 2、线程池未停止，清除wt的中断状态（Thread.interrupted()获取中断状态，并清除线程的中断状态）
             */
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                //钩子方法，留给需要的类实现
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    //钩子方法，留给需要的类实现
                    afterExecute(task, thrown);
                }
            } finally {
                //task置空，循环获取下一个任务
                task = null;
                //完成数累加
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        // 退出Worker
        processWorkerExit(w, completedAbruptly);
    }
}
```
## getTask
 从workQueue中获取任务， 如果返回为null， 前面的Worker就会执行退出流程（processWorkerExit）
1. 如果线程池SHOTDOWN+队列无值，返回null
2. 如果线程池>=STOP 返回null
3. 如果线程数大于核心线程数（workCount>corePoolSize）,workerQueue获取任务超时（keepAliveTime）返回null
4. 如果允许销毁核心线程（allowCoreThreadTimeOut=true）,workerQueue获取任务超时（keepAliveTime）返回null
其余情况，阻塞获取task
```java
/**
 * 从workQueue中获取任务， 如果返回为null， 前面的Worker就会执行退出流程（processWorkerExit）
 * 1、如果线程池SHOTDOWN+队列无值，返回null
 * 2、如果线程池>=STOP 返回null
 * 3、如果线程数大于核心线程数（workCount>corePoolSize）,workerQueue获取任务超时（keepAliveTime）返回null
 * 4、如果允许销毁核心线程（allowCoreThreadTimeOut=true）,workerQueue获取任务超时（keepAliveTime）返回null
 * 其余情况，阻塞获取task
 * @return
 */
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        //SHUTDWON+队列为空 or  >=STOP
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            //CAS减少工作线程数
            decrementWorkerCount();
            return null;
        }

        int wc = workerCountOf(c);

        // Are workers subject to culling?
        //是否允许销毁核心线程（Worker），如果为true,则超过keepAliveTime仍然获取不到任务，返回null，这时Worker会被销毁
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            //需要销毁worker， 返回null，同时较少线程数
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
            //获取任务
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

## processWorkerExit
做了两个事情：  
1. 移除worker，处理一些计数值
2. 根据前面worker是否异常退出，是否满足核心线程数来决定是否添加一个新worker

代码
```java
/**
 * Worker退出操作
 * 1、Workers移除worker
 * 2、根据前面worker是否异常退出，是否满足核心线程数来决定是否添加一个新worker
 * @param w
 * @param completedAbruptly 是否突然完成，true表示线程数（workerCount）未减
 */
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    //Worker突然退出，也就是有异常了，这时候没有减去workerCount
    if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
        decrementWorkerCount();

    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        //已完成的任务计数器
        completedTaskCount += w.completedTasks;
        //Workers集合移除掉
        workers.remove(w);
    } finally {
        mainLock.unlock();
    }

    tryTerminate();

    int c = ctl.get();
    if (runStateLessThan(c, STOP)) {
        if (!completedAbruptly) {
            //如果线程池状态<STOP,且前面Worker正常退出，则判断一下是否满足核心线程数
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;
            if (workerCountOf(c) >= min)
                return; // replacement not needed
        }
        //前面的Worker没有正常退出 or workerCount<核心线程数， 在这里增加Worker
        addWorker(null, false);
    }
}
```
## RejectedExecutionHandler
四种拒绝策略，代码没什么要解释的，一眼就懂
### AbortPolicy
丢弃任务，直接抛出RejectedExecutionException
### DiscardPolicy
丢弃任务，但是不抛异常
### DiscardOldestPolicy
丢弃任务队列中第一个任务，重新提交任务
### CallerRunsPolicy
用调用execute的线程直接run任务

# Executors
线程池工具类  
## 创建线程池
### newFixedThreadPool
```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
```
创建一个固定数量的线程池，corePoolSize=maxmumPoolSize=nThreads，提供无界队列LinkedBlockingQueue来缓存任务队列  
### newSingleThreadExecutor
```java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```
创建一个单线程线程池，和newFixedThreadPool一样，线程数固定为1就可以了  
### newCachedThreadPool
```java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```
创建一个缓存线程池(直译过来名字不太好理解)，就是不限制最大线程数，每进入一个任务就新创建一个线程。 任务队列SynchronousQueue，也就是队列中不缓存任何任务  
### 其他线程池
#### ScheduledThreadPoolExecutor
```java
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}
```
创建一个ScheduledThreadPoolExecutor，可以延迟、周期执行任务
ScheduledThreadPoolExecutor是ThreadPoolExecutor的子类，实现了ScheduledExecutorService来实现延时、周期执行
#### newWorkStealingPool
```java
public static ExecutorService newWorkStealingPool(int parallelism) {
    return new ForkJoinPool
        (parallelism,
         ForkJoinPool.defaultForkJoinWorkerThreadFactory,
         null, true);
}
```
newWorkStealingPool 是jdk7引入的线程池，继承AbstractExecutorService。 与ThreadPoolExecutor相比是把一个任务队列分隔成parallelism（并行数量，默认为cpu核心数）个任务队列，每个队列的任务执行完后会去其他队列偷任务来执行（workSteal）。能够更大效率利用cpu    
> 关于ForkJoinPool，一个使用场景，计算1-10000000之间的数字之和  
> 对于ThreadPoolExecutor,1、分解任务（单线程）2、执行任务（多线程）3、合并任务结果（单线程）  
> 用ForkJoinPool递归操作（任务中判断继续分解任务（fork）or执行任务，执行完后合并任务（join）），这三步都可以多线程执行
##### unconfigurableExecutorService
```java
public static ExecutorService unconfigurableExecutorService(ExecutorService executor) {
    if (executor == null)
        throw new NullPointerException();
    return new DelegatedExecutorService(executor);
}
```
将线程池封装为不可配置的的线程池，也就是不能修改线程池中的一些参数信息，unconfigurableScheduledExecutorService同理
## 总结
线程池的概念前面写的很详细了，过一遍就对概念都清楚了，也没什么好总结的。

