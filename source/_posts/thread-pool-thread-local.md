---
title: 线程池配合ThreadLocal使用的坑
date: 2020-08-26 10:07:53
tags:
---
可以直接看结论，很简单
# 项目中遇到的一个bug
项目中一个记录日志的操作，用了线程池中的线程来异步记录，发现操作人取的不正确
- 使用的相关框架
1. Shiro 权限框架
2. ThreadPoolTaskExecutor Spring线程池，在ThreadPoolExecutor上封装了一层  
<!-- more -->
订单操作日志如下  

|订单号    |操作    |操作人    |日期    |
| :----: | :----: | :----: | :----: |
| 1101 |修改收件人|张三 |2020-08-26 11:08:00|
| 1101 |修改订单 |张三 |2020-08-26 11:11:11|
| 1101 |更新成本 |李四 |2020-08-26 11:11:11|
| 1101 |计算利润 |王五 |2020-08-26 11:11:11|  

确认后得知，订单操作人一直是张三，修改订单、更新成本计算利润是一次操作，却记录了三个操作人。

# 分析原因
## Shiro获取当前登录人
登录人状态是存在session中的，看shiro获取Session的代码
```java
org.apache.shiro.SecurityUtils.getSubject().getSession()
```
可以看到是从SecurityUtils.getSubject()中获取的，点进去找到源头 ，可以看到是使用了InheritableThreadLocalMap（extends InheritableThreadLocal）  
{% asset_img 1.png [] %}

看到这里可以大概推推测一到bug的原因，线程池内的线程都是复用的，下一个人使用池内的子线程，取到的是上一个人向ThreadLocal中存的Subject  

## ThreadLocal与InheritableThreadLocal
两者都与线程绑定，与线程的生命周期一致
InheritableThreadLocal特殊的地方，线程创建时会复制父线程的InheritableThreadLocal给子线程，看一眼源码就清楚了  
> Thread的init()方法中会将父线程的inheritableThreadLocals复制给子线程  
{% asset_img 2.png [] %}

## BUG分析
背景：用户请求父线程进行操作，子线程（线程池内线程）记录日志
1. 项目启动，线程池内子线程数量为0
2. 用户A操作，线程池创建子线程taskExecutor1，复制父线程InheritableThreadLocal，taskExecutor1线程中记录日志。 正常
3. 用户B操作，线程池内已有线程taskExecutor1，用taskExecutor1线程记录日志。 此时InheritableThreadLocal中还是用户A的Session，导致用户B操作，日志中操作人是A
# 解决方案
很简单只要每次使用线程的时候，重新进行一下父线程InheritableThreadLocal复制到子线程的操作即可  
Spring提供的ThreadPoolTaskExecutor提供了一个TaskDecorator，相当于封装了一层任务（Runnable），先看一下源码
```java
ThreadPoolExecutor executor;
if (this.taskDecorator != null) {
	executor = new ThreadPoolExecutor(
			this.corePoolSize, this.maxPoolSize, this.keepAliveSeconds, TimeUnit.SECONDS,
			queue, threadFactory, rejectedExecutionHandler) {
		//重写execute方法，封装command
		@Override
		public void execute(Runnable command) {
			super.execute(taskDecorator.decorate(command));
		}
	};
}
```
我们只需要时写一下TaskDecorator.decorate的实现逻辑就好，在这里加上复制父子线程的InheritableThreadLocal操作
> 悲催的是InheritableThreadLocal相关的操作作用域都是default，只好用反射来实现了
```java
new ThreadPoolTaskExecutor().setTaskDecorator(new TaskDecorator() {
    @Override
    public Runnable decorate(Runnable runnable) {
        try {
            //取父线程的inheritableThreadLocals
            Thread pThread = Thread.currentThread();
            Field parentInheritableThreadLocalsField = Thread.class.getDeclaredField("inheritableThreadLocals");
            parentInheritableThreadLocalsField.setAccessible(true);
            Object parentInheritableThreadLocalsMap = parentInheritableThreadLocalsField.get(pThread);
            parentInheritableThreadLocalsField.setAccessible(false);
            return new Runnable() {
                @Override
                public void run() {
                    Thread currentThread = Thread.currentThread();
                    //复制给子线程
                    try {
                        Field inheritableThreadLocalsField = Thread.class.getDeclaredField("inheritableThreadLocals");
                        inheritableThreadLocalsField.setAccessible(true);
                        inheritableThreadLocalsField.set(currentThread, parentInheritableThreadLocalsMap);
                        //赋值后将该成员变量的访问权限关闭
                        inheritableThreadLocalsField.setAccessible(false);
                    }catch (Exception e){
                        log.error(e.getMessage(), e);
                    }
                    runnable.run();
                }
            };
        }catch (Exception e){
            //异常情况， 打印日志，继续执行任务
            log.error(e.getMessage(), e);
            return runnable;
        }
    }
});
```
修改后测试BUG解决。 对于ThreadPoolExecutor，自己实现ThreadPoolTaskExecutor中重写execute方法的逻辑即可

# 结论
- 允许复用的线程，一定要注意ThreadLocal的生命周期，使用完清除ThreadLocal或者使用时清除ThreadLocal