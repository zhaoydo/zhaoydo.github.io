---
title: 基于Redis实现分布式锁
date: 2020-09-22 17:05:26
tags:
---
# 问题背景
实际开发场景中，可能需要处理并发线程互斥地访问某个共享资源。 Java中通常用synchronized、Lockd来加锁，避免多线程中由于竞争导致的数据不一致问题。  

**问题是Java中的锁，只在同一JVM环境下才生效。**  

分布式系统中，应用部署在多台服务器。这时候需要使用分布式锁，利用一个第三方节点来获取/释放锁。  

分布式锁实现方式有数据库、redis、zookeeper等，基本原理是一样的：标识锁状态+锁的唯一持有者。 这里只介绍单节点下redis实现
![两种架构下锁示意图](http://122.51.119.99:9000/blog/redis-lock/redis-lock02.png)
<!-- more -->
# 分布式锁要求
1. 互斥性（加锁）：任意时刻只能有一个客户端的一个线程持有锁
2. 避免死锁（锁超时）：一个客户端持有锁期间崩溃而没有主动解锁，其他客户端应该也能获取到锁
3. 只允许释放自己的锁（解锁）：A持有锁执行业务，执行时间很长，锁过期后仍未执行完。 B获得锁执行业务期间A执行完成后释放锁，A不能把B的锁释放掉
4. 代码非侵入性：锁操作尽量与业务代码解耦

# 实现
## demo1 最原始的锁操作
redis操作：SETNX key value  
1. key 不存在，SETNX会设置key-value，并返回true
2. key存在，不做操作，返回false  

基于这个特性：
1. 客户端SETNX(key)返回true， 认为获得了锁。
2. 客户端del(key),认为释放了锁  

demo1,代码显式获取/释放锁：  
```java
/**
 *  demo1
 *  代码显式获取/释放锁
 */
@RequestMapping("demo1")
public String demo1() throws InterruptedException{
    String lockKey = "demo1LockKey";
    String lockValue = "demo1LockValue";
    long expireSecond = 10;
    //循环获取锁
    while (true){
        boolean isLock = simpleRedisClient.setNX(lockKey, lockValue, expireSecond);
        if(isLock){
            //获取到锁，跳出循环
            break;
        }else{
            //休眠50ms
            Thread.sleep(50L);
        }
    }
    //获取到锁，执行业务逻辑
    try {
        //模拟业务逻辑，执行5s
        Thread.sleep(5000L);
    }catch (Exception e){
        // demo代码忽略异常处理
    }finally {
        //释放锁
        simpleRedisClient.del(lockKey);
    }
    return "demo1";
}
```
redis实现分布式锁的原理就是基于SETNX。 以下部分都是基于demo1的代码进行封装。
## demo2 封装锁对象
有几个要注意的参数
1. key:锁的key，任一时间不能有两个相同key的锁存在
2. expire:锁超时时间，根据业务大小自行设定(处理获取锁后，客户端崩溃)
3. safetyTime:锁的安全时间（处理获取锁SETNX期间，redis崩溃导致expire未生效）。
> 由于setNX与expire是分两步操作，如果A setNX后挂掉，导致锁不会超时，B进来永远等不到A释放锁，这时候就会死锁
> 解决方法是B获取锁的时候，如果超出safetyTime，强制把A的锁覆盖掉
4. waitMillisPer: 等待时间,获取锁的while循环，每次等待50ms，减少对CPU和redis资源的占用

> 前面提到的只允许释放自己的锁（解锁)
> 获取锁时，设置一个value，释放锁操作时，比较value，只释放自己的锁。 value这里取了线程的ID+name。 
> 不同客户端中碰到线程ID、name一样，并且一方锁超期的情况概率很小。 这里不考虑这种情况了。 
> 优化：lock、unlock方法增加一个UUID的value参数（但是UUID不能实现可重入锁） 或者 ThreadLocal来记录lock、unlock的value  

直接看代码，SyncLock中封装lock、unLock函数
```java
/**
 * 基于redis的分布式同步锁
 * @author zhaoyd
 * @date 2020-05-21
 */
public class SyncLock extends BaseLogger {
    private String key;
    private SimpleRedisClient simpleRedisClient;
    /**redis锁时间 ms*/
    private long expire;
    /**获取redis锁安全时间，默认expire*5 */
    private long safetyTime;
    /**每次获取锁等待时间*/
    private long waitMillisPer = 50L;

    /**
     * 有参构造函数
     * @param key 锁key
     * @param simpleRedisClient redisClient
     * @param expire 锁超时时间
     * @param safetyTime 安全时间
     */
    public SyncLock(String key, SimpleRedisClient simpleRedisClient, Long expire, Long safetyTime){
        this.key = key;
        this.simpleRedisClient = simpleRedisClient;
        this.expire = expire;
        if(safetyTime == null){
            //默认expire*5
            this.safetyTime = expire*5;
        }else {
            this.safetyTime = safetyTime;
        }
    }

    /**
     * 尝试获取锁（立即返回）
     * @return 是否获取成功
     */
    public boolean lockNow() {
        //获取成功时，加锁并返回true
        String value = Thread.currentThread().getId() + "-" + Thread.currentThread().getName();
        boolean locked = simpleRedisClient.setNX(key, value, expire/1000);
        return locked;
    }

    /**
     * 阻塞获取锁
     * @return 是否获取成功
     */
    public boolean lock() throws InterruptedException{
        String value = Thread.currentThread().getId() + "-" + Thread.currentThread().getName();
        log.info("lock start key={},value={}", key, value);
        //已等待毫秒数
        long waitMillisAlready = 0L;
        //最大等待毫秒数
        long maxWaitMillis = this.safetyTime;
        while (waitMillisAlready < maxWaitMillis) {
            boolean locked = simpleRedisClient.setNX(key, value, expire/1000);
            if(locked){
                //成功获取锁
                log.info("lock success key={},value={}", key, value);
                return true;
            }else{
                Thread.sleep(waitMillisPer);
                waitMillisAlready += waitMillisPer;
            }
        }
        //超过safetyTime仍然未获取到锁，强制获取锁
        log.info("lock fail key={},value={}, try force lock", key, value);
        simpleRedisClient.set(key, value, expire/1000);
        return true;
    }

    /**
     * 释放锁， 只释放自己的
     */
    public void unLock() {
        String value = Thread.currentThread().getId() + "-" + Thread.currentThread().getName();
        String redisValue = simpleRedisClient.get(key);
        if(redisValue == null){
            //已经失效了
            return;
        }
        if (StringUtils.equals(redisValue, value)) {
            simpleRedisClient.del(key);
            log.info("unlock key={},value={}", key, value);
        }else{
            //不释放其他线程的锁
            log.info("unlock fail key={},value={},redisValue={}", key, value,redisValue);
        }
    }

    public String getKey() {
        return key;
    }

    public void setKey(String key) {
        this.key = key;
    }

    public SimpleRedisClient getSimpleRedisClient() {
        return simpleRedisClient;
    }

    public void setSimpleRedisClient(SimpleRedisClient simpleRedisClient) {
        this.simpleRedisClient = simpleRedisClient;
    }

    public long getExpire() {
        return expire;
    }

    public void setExpire(long expire) {
        this.expire = expire;
    }

    public long getSafetyTime() {
        return safetyTime;
    }

    public void setSafetyTime(long safetyTime) {
        this.safetyTime = safetyTime;
    }
}
```

demo2 封装获取/释放锁代码。 这里与单机环境下使用java显式锁ReentrantLock的代码基本一样
```java
/**
 *  demo2
 *  封装获取/释放锁 SyncLock
 */
@RequestMapping("demo2")
public String demo2() throws InterruptedException{
    String lockKey = "demo2LockKey";
    String lockValue = "demo2LockValue";
    long expireSecond = 10;
    //创建锁对象
    SyncLock syncLock = new SyncLock(lockKey, simpleRedisClient, expireSecond*1000L, null);
    //阻塞获取锁
    Boolean isLock = syncLock.lock();
    AssertUtils.isTrue(isLock, "获取锁失败,超时");
    //获取到锁，执行业务逻辑
    try {
        //模拟业务逻辑，执行5s
        Thread.sleep(5000L);
    }catch (Exception e){
        // demo代码忽略异常处理
    }finally {
        //释放锁
        syncLock.unLock();
    }
    return "demo2";
}
```

## demo3 与Spring集成
上面demo1、demo2中的代码中，加锁、解锁的操作都会对业务代码有侵入性。  
自定义注解+Spring AOP封装，使用时一行注解实现  

### 整体结构
![整体结构](http://122.51.119.99:9000/blog/redis-lock/redis-lock01.png)
### 工厂类创建锁对象
工厂类创建SyncLock对象，目的是为了对相同的key，不重复创建SyncLock对象。 并发高时对GC友好
```java
/**
 * SyncLock工厂类
 * @author zhaoyd
 * @date 2020-05-21
 */
@Component
public class SyncLockFactory {
    @Resource
    private SimpleRedisClient simpleRedisClient;

    private Map<String,SyncLock> syncLockMap = new ConcurrentHashMap<>();

    /**
     * 创建SyncLock
     * 这里搞个类似单例的双重检查锁性能更好
     * @param key Redis key
     * @param expire Redis TTL/秒，默认10秒
     * @param safetyTime 安全时间/秒，为了防止程序异常导致死锁，在此时间后强制拿锁，默认 expire * 5 秒
     */
    public synchronized SyncLock build(String key,long expire, Long safetyTime){
        if (!syncLockMap.containsKey(key)) {
            syncLockMap.put(key, new SyncLock(key, simpleRedisClient, expire, safetyTime));
        }
        return syncLockMap.get(key);
    }
}
```
### 注解类
```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface SyncLockable {
    /**锁key*/
    String key() default "";
    /**锁超时时间*/
    long expire() default 10000L;
}
```
### AOP处理
注解aop实现，拦截@SyncLockable、@SyncLockableNow,目标方法前后分别加上获取锁、释放锁的操作
```java
/**
 * 注解AOP实现
 * @author zhaoyd
 * @date 2020-05-21
 */
@Aspect
@Component
public class SyncLockHandle {
    @Resource
    private SyncLockFactory syncLockFactory;

    /**
     * 在方法上执行同步锁
     */
    @Around("@annotation(syncLockable)")
    public Object around(ProceedingJoinPoint point, SyncLockable syncLockable) throws Throwable{
        String key = syncLockable.key();
        Long expire = syncLockable.expire();
        if(StringUtils.isBlank(key)){
            key = this.getDetaultKey(point);
        }
        SyncLock lock = syncLockFactory.build(key, expire, null);
        try {
            boolean isLock = lock.lock();
            AssertUtils.isTrue(isLock, "获取锁失败");
            return point.proceed();
        } finally {
            lock.unLock();
        }
    }

    @Around("@annotation(syncLockableNow)")
    public Object around(ProceedingJoinPoint point, SyncLockableNow syncLockableNow) throws Throwable{
        String key = syncLockableNow.key();
        Long expire = syncLockableNow.expire();
        if(StringUtils.isBlank(key)){
            key = this.getDetaultKey(point);
        }
        SyncLock lock = syncLockFactory.build(key, expire, null);
        try {
            boolean isLock = lock.lockNow();
            AssertUtils.isTrue(isLock, "获取锁失败，有并发");
            return point.proceed();
        } finally {
            lock.unLock();
        }
    }

    /**
     * 获取className+methodName
     * @param point
     * @return
     * @throws NoSuchMethodException
     */
    private String getDetaultKey(ProceedingJoinPoint point) throws NoSuchMethodException{
        Signature sig = point.getSignature();
        MethodSignature msig = null;
        if (!(sig instanceof MethodSignature)) {
            throw new IllegalArgumentException("该注解只能用于方法");
        }
        msig = (MethodSignature) sig;
        Object target = point.getTarget();
        Method currentMethod = target.getClass().getMethod(msig.getName(), msig.getParameterTypes());
        String methodName = currentMethod.getName();
        String className = target.getClass().getName();
        return className + "-" + methodName;
    }

}
```
### 客户端使用 自定义注解  
```java
/**
 *  demo3
 *  aop注解实现
 */
@SyncLockable
@RequestMapping("demo3")
public String demo3() throws InterruptedException{
    //模拟业务逻辑，执行5s
    Thread.sleep(5000L);
    return "demo3";
}
```
## demo4 扩展，支持el表达式
场景： 对SKU扣减库存，这时候只要保证同一个仓库中的SKU操作不出现并发即可。这时候我们希望分布式锁的key可以根据参数来确定。  
在AOP代码中加上el表达式的解析逻辑即可
```java
/**
 * 解析el表达式
 * @param key
 * @param point
 * @return
 */
private String parseEL(String key, ProceedingJoinPoint point){
    //解析el表达式，将#id等替换为参数值
    ExpressionParser expressionParser = new SpelExpressionParser();
    //el表达式需要用#{}包裹
    Expression expression = expressionParser.parseExpression(key, ParserContext.TEMPLATE_EXPRESSION);
    EvaluationContext context = new StandardEvaluationContext();
    String[] parameterNames = ((MethodSignature)point.getSignature()).getParameterNames();
    Object[] args = point.getArgs();
    if(parameterNames==null || parameterNames.length<=0){
        return key;
    }
    for (int i = 0; i <parameterNames.length ; i++) {
        context.setVariable(parameterNames[i],args[i]);
    }
    key = expression.getValue(context).toString();
    return key;
}
```
客户端使用 支持el表达式的注解key
```java
/**
 *  demo4
 *  aop注解实现 + el表达式
 */
@SyncLockable(key = "demo4#{#whId+'_'+#sku}")
@RequestMapping("demo4")
public String demo4(Integer whId,String sku) throws InterruptedException{
    //模拟业务逻辑，执行5s
    Thread.sleep(5000L);
    return "demo4";
}
```

# 总结

## 需要改进的地方
1. 当前版本是非公平锁。 多个客户端抢占锁资源时，不能按照先来先得的顺序获得锁
2. key支持el表达式后，key的值根据方法参数变化。 SyncLockFactory中syncLockMap缓存的SyncLock对象可能会很大导致内存溢出。 
> 考虑把syncLockMap改进为弱引用,不使用时Map中对象允许被回收