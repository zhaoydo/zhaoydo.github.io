---
title: 读AQS源码
date: 2021-09-24 20:39:00
tags: 源码
---

## AQS简介

AbstractQueueSynchronizer，是java.util.concurrent中最重要的类，是许多常用的并发工具类的基础，ReentrantLock、CountDownLatch、Semaphore、FutureTask 等都是在AQS抽象类的基础上实现而来。 学习AQS源码有助于更好的理解并发工具类的实现原理，对写出更高效、健壮的代码很有帮助。

<!--more-->

## AQS结构

先看AQS中有那些属性

```java
//等待队列head，也就是当前持有锁的线程
private transient volatile Node head;
//等待队列尾，新节点进来插入到当前tail后
private transient volatile Node tail;
//锁占用状态，0表示无锁占用，>=1表示有锁占用(锁重入次数)
private volatile int state;
//继承自AbstractOwnableSynchronizer， 表示当前持有独占锁的线程
private transient Thread exclusiveOwnerThread;
```



等待队列是一个双向链表，队列中的每个线程被包装为一个Node实例，下面是Node的源码

```java
static final class Node {
  //共享模式 标识
  static final Node SHARED = new Node();
  //独占模式 标识
  static final Node EXCLUSIVE = null;
  /**
   * 定义waitStatus的几种状态
   * CANCELLED 1 当前线程取消抢占锁
   * SIGNAL -1 节点的后续节点需要继续运行，unpaking
   * CONDITION -2 表示当前线程在等待CONDITION（在CONDITION队列中）
   * PROPAGATE TODO
   * 0 在正常同步队列中
   */
  /** waitStatus value to indicate thread has cancelled. */
  static final int CANCELLED =  1;
  /** waitStatus value to indicate successor's thread needs unparking. */
  static final int SIGNAL    = -1;
  /** waitStatus value to indicate thread is waiting on condition. */
  static final int CONDITION = -2;
  /**
   * waitStatus value to indicate the next acquireShared should
   * unconditionally propagate.
   */
  static final int PROPAGATE = -3;
  // 节点状态
  volatile int waitStatus;
	// 节点前后指针
  volatile Node prev;
  volatile Node next;
  // 节点线程
  volatile Thread thread;
}
```

## ReentrantLock

ReentrantLock

### 初始化

ReentrantLock初始化时可选择公平锁or非公平锁，分别由不同的Sync内部类实现

```java
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

抽象类Sync继承了AQS

```java
abstract static class Sync extends AbstractQueuedSynchronizer {
}
```



### 加锁

加锁操作由Sync类acquire方法实现

```java
/**
 * Sync object for fair locks
 * 公平锁
 */
static final class FairSync extends Sync {
    //ReentrantLock加锁方法
    public void lock() {
        //直接调用sync
        sync.acquire(1);
    }
    /**
     * AQS提供的获取锁方法，公平锁和非公平锁都走这个
     */
    public final void acquire(int arg) {
        /**
         * 1、子类实现tryAcquire 先尝试获取一次锁，如果当前无线程持有锁，直接成功，不用走排队流程了
         * 2、如果获取失败，走AQS的acquireQueued，将线程放到阻塞队列中，返回线程的中断状态
         */
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            //前面获取锁时，没有对interrupted状态进行相应，如果返回interrupted状态为true
            //前面获取中断状态时调用isInterrupted(true)获取并清除中断状态，所以这里需要再中断一下线程
            selfInterrupt();
    }
    /**
     * Fair version of tryAcquire.  Don't grant access unless
     * recursive call or no waiters or is first.
     */
    /**
     * 公平锁尝试获取一次锁，如果有其他线程占用锁，直接返回false
     * @param acquires
     * return true 表示获取到了锁（当前无锁，且队列中没有其他线程排队 or 线程持有锁重入）
     * return false 没有获取到锁
     */
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        // 锁状态，0表示没有锁占用，1表示有锁占用 >1表示锁重入次数
        // if c==0 说明当前无线程持有锁
        if (c == 0) {
            /**
             * 无线程持有锁 && 队列中也没有其他线程在排队，直接CAS尝试占用
             */
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                //占用成功，设置一下当前独占的线程current，返回true
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        // else if 当前持有锁的线程是自己，说明是重入了，修改state+=acquires，返回true
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        // else tryAcquire失败，返回true
        return false;
    }
  
    //tryAcquire获取锁失败，将线程加入等待队列，并阻塞队列出队获取锁 acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
    //先将线程封装为Node，加入队列中，这里很简单
    /**
     * 上一步trycquire获取锁失败，将线程进入到等待队列中
     * @param mode mode表示Node.EXCLUSIVE  or Node.SHARED
     */
    private Node addWaiter(Node mode) {
        //初始化node
        Node node = new Node(mode);
        //将node放入队列尾或者是队列的head()
        for (;;) {
            Node oldTail = tail;
            if (oldTail != null) {
                //放到队列尾
                node.setPrevRelaxed(oldTail);
                if (compareAndSetTail(oldTail, node)) {
                    oldTail.next = node;
                    return node;
                }
            } else {
                //当前无队列，直接初始化为head
                initializeSyncQueue();
            }
        }
    }
    //队列阻塞出队，acquireQueued，返回线程interrupted状态，此方法返回说明获取到了锁
    /**
     * addWaiter(Node.EXCLUSIVE)方法后，node进入到了阻塞队列
     * 方法在线程成功获得锁后退出，返回当前线程的中断状态
     * @param node 当先线程排队的node
     * @param arg
     * @return 正常返回false，如果返回true，说明线程在等待时中断了interrupted，后续会进入selfInterrupt()
     */
    final boolean acquireQueued(final Node node, int arg) {
        boolean interrupted = false;
        try {
            for (;;) {
                //获取上一个节点
                final Node p = node.predecessor();
                /**
                 * 上一个节点是header（当前持有锁的线程）、说明下一个就是node了，尝试获取锁
                 * 如果获取成功，当前节点进入头节点，返回
                 */
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    return interrupted;
                }
                /**
                 * 没有抢到锁
                 * shouldParkAfterFailedAcquire 表示没有抢到锁后是否park挂起线程
                 * 如果pred节点waitStatus=SINGAL，则park挂起
                 */
                if (shouldParkAfterFailedAcquire(p, node))
                    // park挂起线程，记录当前线程的interrupted状态(调用isInterrupted(true)清除掉中断状态)
                    interrupted |= parkAndCheckInterrupt();
            }
        } catch (Throwable t) {
            cancelAcquire(node);
            if (interrupted)
                selfInterrupt();
            throw t;
        }
    }
}
```

对于非公平锁，在tryAcquire方法的实现有所区别，核心是少了hasQueuedPredecessors()判断是是否有线程在排队的逻辑

```java
        /**
         * 非公平锁尝试获取锁，对比公平锁少了hasQueuedPredecessors()判断是是否有线程在排队的逻辑
         * @param acquires
         * @return
         */
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                //无线程持有锁时，直接cas占用锁
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```

### 解锁

解锁操作由Sync类release方法实现

```java
public void unlock() {
  sync.release(1);
}
/**
  * 释放锁操作
  */
public final boolean release(int arg) {
  if (tryRelease(arg)) {
    //释放锁成功后，唤醒后续节点线程（unpark）(回忆一下上面的加锁时节点的park操作)
    Node h = head;
    if (h != null && h.waitStatus != 0)
      unparkSuccessor(h);
    return true;
  }
  return false;
}
/**
  * 独占锁释放锁操作
  * 1、完全释放锁，state最终为0
  * 2、不完全释放，嵌套重入锁的情况，只释放一层锁
  */
protected final boolean tryRelease(int releases) {
  int c = getState() - releases;
  //判断是否为当前占用锁的线程
  if (Thread.currentThread() != getExclusiveOwnerThread())
    throw new IllegalMonitorStateException();
  //free 表示是否完全释放锁s
  boolean free = false;
  if (c == 0) {
    //完全释放锁
    free = true;
    setExclusiveOwnerThread(null);
  }
  setState(c);
  return free;
}
```

## Condition

一个demo展示Condition的使用场景

```java
public class BoundedBuffer {
   	private ReentrantLock lock = new ReentrantLock();
    private Condition notEmpty = lock.newCondition();
    private Condition notFull = lock.newCondition();
    private Object[] items = new Object[100];
    private int putIdx,takeIdx,count;

    public void put(Object item) throws InterruptedException{
        lock.lock();
        try{
            //数组满的情况
            while (count == items.length){
                notFull.await();
            }
            items[putIdx] = item;
            if(++putIdx == items.length){
                putIdx = 0;
            }
            ++count;
            notEmpty.signal();
        }finally {
            lock.unlock();
        }
    }
    public Object take() throws InterruptedException{
        lock.lock();
        try{
            //数组空的情况
            while (count == 0){
                notEmpty.await();
            }
            Object item = items[takeIdx];
            if(++takeIdx == items.length){
                takeIdx = 0;
            }
            count--;
            notFull.signal();
            return item;
        }finally {
            lock.unlock();
        }
    }
}
```

先说流程，方便理解代码

>   区分两个概念，阻塞队列：asyncQueue, 条件队列：conditionQueue

前置条件，当前线程持有锁

1.   condition.await()，向条件队列(firstWaiter，lastWaiter)中加入node，从阻塞队列（head，tail）中取node
2.   condition.signal()，将条件队列firstWaiter转入阻塞队列
3.   condition.await()，第一步后续从阻塞队列中排队取node成功，进入await后续代码执行

线程中断的情况

1.   condition.signal()之前线程中断，条件队列的node也会进入阻塞队列，但是await()排队到阻塞队列后会throw new InterruptedException()响应中断，不会执行后续逻辑
2.   condition.signal()之后线程中断，condition.await()从排队获取到后，执行selfInterrupt()，中断由await()后续代码来响应

### await

先看一眼await，了解大概流程，分析里面每一步的方法逻辑后，再看一眼这个方法才能理解

```java

public final void await() throws InterruptedException {
  //响应线程中断,对比awaitUninterruptibly()方法未响应线程中断
  if (Thread.interrupted())
    throw new InterruptedException();
  //加入的Condition等待队列中
  Node node = addConditionWaiter();
  //释放掉当前线程持有的所有锁
  int savedState = fullyRelease(node);
  int interruptMode = 0;
  //isOnSyncQueue 返回true表示在阻塞队列中 
  while (!isOnSyncQueue(node)) {
    LockSupport.park(this);
    // 检测到线程中断，break
    if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
      break;
  }
  /**
   * 进入这里有两种情况
   * 1. 条件队列node已经进入阻塞队列，Condition.signal或者线程中断都可能
   * 2. 线程中断 THROW_IE or REINTERRUPT
   */
  // 进入阻塞队列，重新获取锁
  if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
    interruptMode = REINTERRUPT;
  //Q:为什么有nextWaiter，说明取消了？？
  //A:注意前面的两种情况Condition.signal时会将node的nextWaiter清除掉，如果!=null则一定是节点取消了
  if (node.nextWaiter != null) // clean up if cancelled
    //清除掉所有不为Node.CONDITION状态的节点
    unlinkCancelledWaiters();
  //响应线程中断
  if (interruptMode != 0)
    reportInterruptAfterWait(interruptMode);
}
```

#### addConditionWaiter

向条件队列中加入一个Node，很简单

```java
//加入condition queue
private Node addConditionWaiter() {
  //判断下当前锁的独占线程是否为自己
  if (!isHeldExclusively())
    throw new IllegalMonitorStateException();
  Node t = lastWaiter;
  // 如果等待队列最后一个节点状态！=CONDITION，清除队列所有不为CONDITION的节点
  if (t != null && t.waitStatus != Node.CONDITION) {
    unlinkCancelledWaiters();
    t = lastWaiter;
  }
  //初始化CONDITION NODE
  Node node = new Node(Node.CONDITION);

  if (t == null)
    firstWaiter = node;
  else
    t.nextWaiter = node;
  lastWaiter = node;
  return node;
}
```

#### fullyRelease

释放当前线程持有的锁，也就是前面demo中的lock锁，方在await的时候让其他线程可以操作资源

```java
    /**
     * 完全释放节点的独占锁
     * 1.savedState表示当前线程持有锁的重入次数（可重入锁）
     * 2.全部释放，如果失败，条件队列节点标记为取消状态，并抛出
     * @param node
     * @return
     */
    final int fullyRelease(Node node) {
        try {
            int savedState = getState();
            if (release(savedState))
                return savedState;
            throw new IllegalMonitorStateException();
        } catch (Throwable t) {
            node.waitStatus = Node.CANCELLED;
            throw t;
        }
    }
```

#### 等待条件被唤醒

这里由两部分代码组成：  

1.   等待进入阻塞队列
2.   从阻塞队列中获取node

while (!isOnSyncQueue(node)), 直到进入阻塞队列（或者线程中断）后跳出while循环

```java
    /**
     * 前提：signal时会将条件队列转移到阻塞队列
     * 判断node是否转移到了阻塞队列
     * @param node
     * @return
     */
    final boolean isOnSyncQueue(Node node) {
        //状态为CONDITION 说明还在条件队列，进入阻塞队列一定会给条件node加一个prev（tail）
        if (node.waitStatus == Node.CONDITION || node.prev == null)
            return false;
        if (node.next != null) // If has successor, it must be on queue
            return true;
        //遍历阻塞队列，判断node是否在阻塞队列中
        return findNodeFromTail(node);
    }
```

从阻塞队列中获取node,再看一眼await()里的代码, 注意线程中断时也会进入阻塞队列，根据interruptMode来处理抛异常还是指标记中断

```java
/**
 * 进入这里有两种情况
 * 1. 条件队列node已经进入阻塞队列，Condition.signal或者线程中断都可能
 * 2. 线程中断 THROW_IE or REINTERRUPT
 */
// 进入阻塞队列，重新获取锁
if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
  interruptMode = REINTERRUPT;
//Q:为什么有nextWaiter，说明取消了？？
//A:注意前面的两种情况Condition.signal时会将node的nextWaiter清除掉，如果!=null则一定是节点取消了
if (node.nextWaiter != null) // clean up if cancelled
  //清除掉所有不为Node.CONDITION状态的节点
  unlinkCancelledWaiters();
//响应线程中断
if (interruptMode != 0)
  reportInterruptAfterWait(interruptMode);
```

### signal

将条件队列第一个未取消node，转入阻塞队列，这样前面的await()方法就能从阻塞队列中拿到node，继续执行任务了

```java
      /**
         * 从条件队列first开始向后遍历
         * @param first
         */
        private void doSignal(Node first) {
            do {
                //firstWaiter 向后遍历一位
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;
                // fitst单独拿出来，准备放入阻塞队列，next指针断掉
                first.nextWaiter = null;
            // 如果first转移到阻塞队列失败，转下一个
            } while (!transferForSignal(first) &&
                     (first = firstWaiter) != null);
        }
    /**
     * 将Node从条件队列转移到阻塞队列
     */
    final boolean transferForSignal(Node node) {
        /*
         * If cannot change waitStatus, the node has been cancelled.
         */
        // cas将节点状态Node.CONDITION->0,如果失败，说明node被取消了，返回false转下一个即可
        if (!node.compareAndSetWaitStatus(Node.CONDITION, 0))
            return false;

        //将node塞入阻塞队列的队尾，返回node在阻塞队列中的前驱节点
        Node p = enq(node);
        int ws = p.waitStatus;
        //ws>0说明node前驱节点取消了 唤醒当前node线程
        //，ws<=0,将前驱节点设为Node.SIGNAL
        if (ws > 0 || !p.compareAndSetWaitStatus(ws, Node.SIGNAL))
            //如果前驱节点取消，或者cas失败，唤醒当前线程 TODO
            LockSupport.unpark(node.thread);
        return true;
    }
```

### 中断的处理

简单来说，两种情况：

1.   signal()前中断，interruptMode=THROW_IE await()中抛异常
2.   signal()后中断，interruptMode=REINTERRUPT await()中selfInterrupt()重新标识中断，具体是否响应交给后面的业务代码来处理

```java
// await()中，线程被唤醒后，检查中断
while (!isOnSyncQueue(node)) {
  LockSupport.park(this);
  // 检测到线程中断，break
  if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
    break;
}

/**
 * 发生中断时
 * 在signal之前中断，THROW_IE
 * 在signal之后中断，REINTERRUPT
 */
private int checkInterruptWhileWaiting(Node node) {
  return Thread.interrupted() ?
    (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
  0;
}
/**
 * 线程中断后将node转移到阻塞队列
 * true 说明在signal之前转移
 * false说明在signal之后转移
 * @param node
 * @return
*/
final boolean transferAfterCancelledWait(Node node) {
  //CAS将node设为0,如果失败，说明已经执行过signal方法了，node已经转移到了阻塞队列
  if (node.compareAndSetWaitStatus(Node.CONDITION, 0)) {
    //将节点转移到阻塞队列
    enq(node);
    return true;
  }
  // cas失败，说明已经signal转移阻塞队列了，这里自旋等待进入阻塞队列即可
  while (!isOnSyncQueue(node))
    Thread.yield();
  return false;
}

// 再回到await()，对中断的两种情况处理
/*
 * 对interruptMode处理
 * THROW_IE,抛出InterruptedException
 * REINTERRUPT,重新将线程标识位interrupt状态  
 * 其他， do nothing
 */
private void reportInterruptAfterWait(int interruptMode)
    throws InterruptedException {
    if (interruptMode == THROW_IE)
      throw new InterruptedException();
    else if (interruptMode == REINTERRUPT)
      selfInterrupt();
  }
```

