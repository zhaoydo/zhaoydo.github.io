---
title: Spring线程池ThreadPoolTaskExecutor
categories:
- 开发
date: 2018-07-04 19:19:00
tags:
- spring
---

spring框架提供了线程池:ThreadPoolTaskExecutor,配置一下可以直接用  
<!-- more -->
# 配置
## xml方式
```xml
<!-- spring thread pool executor -->           
    <bean id="taskExecutor" class="org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor">
        <!-- 线程池维护线程的最少数量 -->
        <property name="corePoolSize" value="5" />
        <!-- 允许的空闲时间 -->
        <property name="keepAliveSeconds" value="200" />
        <!-- 线程池维护线程的最大数量 -->
        <property name="maxPoolSize" value="10" />
        <!-- 缓存队列 -->
        <property name="queueCapacity" value="20" />
        <!-- 对拒绝task的处理策略 -->
        <property name="rejectedExecutionHandler">
            <bean class="java.util.concurrent.ThreadPoolExecutor.CallerRunsPolicy" />
        </property>
    </bean>
```
## SpringBoot 配置类  
```java
@EnableAsync
@Configuration
public class ThreadPoolConfig {
    //线程池维护线程的最少数量
    private final static int CORE_POOL_SIZE = 10;
    //线程池维护线程的最大数量
    private final static int MAX_POOL_SIZE = 50;
    //缓存队列
    private final static int QUEUE_CAPACITY = 20;
    //允许的空闲时间
    private final static int KEEP_ALIVE = 60;

    @Bean(taskExecutor)
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(CORE_POOL_SIZE);
        executor.setMaxPoolSize(MAX_POOL_SIZE);
        executor.setQueueCapacity(QUEUE_CAPACITY);
        executor.setThreadNamePrefix("taskExecutor_");
        // rejection-policy：当pool已经达到max size的时候，如何处理新任务
        // CALLER_RUNS：不在新线程中执行任务，而是由调用者所在的线程来执行
        //对拒绝task的处理策略
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.setKeepAliveSeconds(KEEP_ALIVE);
        executor.initialize();
        return executor;
    }
}
```
# 配置说明  
* corePoolSize：线程池维护线程的最少数量
* keepAliveSeconds：允许的空闲时间
* maxPoolSize：线程池维护线程的最大数量
* queueCapacity：缓存队列
* rejectedExecutionHandler：对拒绝task的处理策略  

**rejectedExecutionHandler策略(超出线程池的任务处理策略): ** 
* AbortPolicy，抛出RejectedExecutionException。
* CallerRunsPolicy，用调用者线程执行。
* DiscardOldestPolicy，它放弃最旧的未处理请求，然后重试execute。
* DiscardPolicy，默认情况下它将丢弃被拒绝的任务  
**说明：**
当前项目用的是CallerRunsPolicy，超出线程池处理能力后，用调用者线程执行，这样不会丢失任务  
用户可以选择使用自定义策略，只需实现RejectedExecutionHandler接口即可（没实现过，**有时间看看**）  

# 执行过程
1. 当一个任务被提交到线程池时，首先查看线程池的核心线程是否都在执行任务，否就选择一条线程执行任务，是就执行第二步。
2. 查看核心线程池是否已满，不满就创建一条线程执行任务，否则执行第三步。
3. 查看任务队列是否已满，不满就将任务存储在任务队列中，否则执行第四步。
4. 查看线程池是否已满，不满就创建一条线程执行任务，否则就按照策略处理无法执行的任务。  

在ThreadPoolExecutor中表现为:  
* 如果当前运行的线程数小于corePoolSize，那么就创建线程来执行任务（执行时需要获取全局锁）。
* 如果运行的线程大于或等于corePoolSize，那么就把task加入BlockQueue。
* 如果创建的线程数量大于BlockQueue的最大容量，那么创建新线程来执行该任务。
* 如果创建线程导致当前运行的线程数超过maximumPoolSize，就根据饱和策略来拒绝该任务。

# 使用
## 手动方式
直接获取bean，通过execute方法执行，和Executors.newFixedThreadPool(threadNum)方式一致
```java
@Resource
private Executor taskExecutor;
@Test
public void taskTest(){
    taskExecutor.execute(new Runnable() {
        @Override
        public void run() {
            System.out.println(Thread.currentThread().getName());
        }
    });
}
```
## @Async方式
线程池执行方法加@Async注解，表示这个方法是异步执行的
```java
//任务执行方法
@Async
public Future<Integer> asyncTask(final Integer param){
	//返回结果
    Integer result = param;
    return new AsyncResult<>(result);
}
```
获取所有线程返回结果，与Executors方式一样
```
//异步调用并获取所有返回结果
List<Future<Integer>> futures = new ArrayList<>();
for(final Integer id : ids){
    try {
        Future<Integer> future = xxx.asyncTask(final Integer param);
        futures.add(future);
    } catch (Exception e){
        //线程执行方法的异常不会在这里捕获，这里为了防止编译错误
        log.error(e.getMessage(), e);
    }
}
List<Integer> results = new ArrayList<>();
for(Future<Integer> future : futures){
    try {
        //future.get()会阻塞线程，直至获取返回结果
        Integer result = future.get();
        results.add(result);
    } catch (Exception e) {
        //线程执行方法的异常将在这里捕获
        log.error(e.getMessage(), e);
    }
}
//得到所有线程返回结果results
```