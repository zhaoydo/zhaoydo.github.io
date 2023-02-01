---
title: 服务无损下线实现
categories:
- springcloud
date: 2023-01-31 20:00:00
tags:
- springcloud
---


服务关闭时，异常主要有两个场景

场景一： 服务关闭时，服务内部正在执行的请求未完成

场景二：服务下线后，其他调用者获取下线通知有延迟。造成短时间调用失败

![](https://raw.githubusercontent.com/zhaoydo/pickgo_bed/master/img/image-20230131150918825.png)

Spring Boot 2.3新增了Graceful Shutdown，在服务关闭时，检查服务内部工作线程执行完毕后再结束服务，对于期间新请求全部HTTP 503拒绝。这种处理方式只解决了场景一，没有解决场景二的问题。

网上找到能同时解决两种场景的方案，只能自己实现了。

<!--more-->

# 无损下线步骤

下面的步骤可以实现服务关闭时，正在执行的请求执行完毕，下线期间不会被其他服务调用

服务下线步骤

![image-20230131153136946](https://raw.githubusercontent.com/zhaoydo/pickgo_bed/master/img/image-20230131153136946.png)

1. 服务B主动告知注册中心下线
2. 延迟xx秒，所有调用者感知到服务下线
3. 延迟xx秒，等待服务内请求处理完毕
4. 服务关闭/重启



# 运维考虑



服务主动下线代码很容易实现，需要考虑的是如何在服务关闭前触发主动下线。

 网上大部分实现都是通过暴露一个接口来下线，

```shell
# 调用自定义接口，主动下线
curl http://127.0.0.1:8080/shutdown
# 延迟30s
sleep 30
# 关闭服务
kill <PID>
```

这种方案有两个缺点

1. 不安全
2. 增加了运维配置(需要配置下线url)。 

查找资料后找到一种方式可以直接通过kill pid触发主动下线，延迟关闭的实现。

# 代码实现

先看一下Spring是如何关闭服务的

```java
public abstract class AbstractApplicationContext extends DefaultResourceLoader
		implements ConfigurableApplicationContext {
    @Override
	public void registerShutdownHook() {
		if (this.shutdownHook == null) {
			// No shutdown hook registered yet.
			this.shutdownHook = new Thread(SHUTDOWN_HOOK_THREAD_NAME) {
				@Override
				public void run() {
					synchronized (startupShutdownMonitor) {
						doClose();
					}
				}
			};
			Runtime.getRuntime().addShutdownHook(this.shutdownHook);
		}
	}
}
```

从源码可以看出，Spring启动时注册了一个shutdownHook，在JVM关闭时执行Context.close()的操作。只要重写这个shutdownHook即可实现我们自定义的关闭流程

```java
public static void main(String[] args) {
    SpringApplication springApplication = new SpringApplication(DemoApplication.class);
    //启动， 关闭时延迟30s销毁容器
    startWithDelayShutdown(springApplication,args, 30);
}
/**
     * 延迟停机。服务关闭时，先主动下线服务，延迟一定时间后再关闭服务
     * @param springApplication
     * @param args
     */
public static void startWithDelayShutdown(SpringApplication springApplication, String[] args, Integer delaySeconds){
    // 默认deleay 30s
    if(delaySeconds == null){
        delaySeconds = 30;
    }
    //关闭自带的SpringContextShutdownHook
    springApplication.setRegisterShutdownHook(false);
    //启动Spring
    ConfigurableApplicationContext context = springApplication.run(args);
    //注册自定义SpringContextShutdownHook
    Integer finalDelaySeconds = delaySeconds;
    Thread shutdownHook = new Thread("MY-SpringContextShutdownHook") {
        @Override
        public void run() {
            //我这里是nacos，其他注册中心都是一样的
            log.info("服务停止，主动下线");
            NacosAutoServiceRegistration nacosAutoServiceRegistration = SpringContextUtils.getBean(NacosAutoServiceRegistration.class);
            nacosAutoServiceRegistration.stop();
            //下线30s后停止
            log.info("停止，休眠{}s", finalDelaySeconds);
            try {
                Thread.sleep(finalDelaySeconds *1000L);
            } catch (InterruptedException e) {
                log.error(e.getMessage(), e);
            }
            log.info("停止，休眠{}s结束，销毁容器", finalDelaySeconds);
            context.close();
        }
    };
    Runtime.getRuntime().addShutdownHook(shutdownHook);
}
```

项目启动后，kill PID时，会在服务下线后，延迟30s去处理服务内正在执行的请求和服务调用者感知下线延迟期间的请求。

```shell
2023-01-31 15:51:56.013 [MY-SpringContextShutdownHook] [DemoApplication.java:54] INFO  com.xxx.trade.demo.DemoApplication - 服务停止，主动下线
2023-01-31 15:51:56.013 [Thread-28] [HttpClientBeanHolder.java:108] WARN  com.alibaba.nacos.common.http.HttpClientBeanHolder - [HttpClientBeanHolder] Start destroying common HttpClient
2023-01-31 15:51:56.013 [Thread-25] [NotifyCenter.java:136] WARN  com.alibaba.nacos.common.notify.NotifyCenter - [NotifyCenter] Start destroying Publisher
2023-01-31 15:51:56.014 [Thread-25] [NotifyCenter.java:153] WARN  com.alibaba.nacos.common.notify.NotifyCenter - [NotifyCenter] Destruction of the end
2023-01-31 15:51:56.015 [MY-SpringContextShutdownHook] [NacosServiceRegistry.java:94] INFO  c.a.cloud.nacos.registry.NacosServiceRegistry - De-registering from Nacos Server now...
2023-01-31 15:51:56.015 [Thread-28] [HttpClientBeanHolder.java:114] WARN  com.alibaba.nacos.common.http.HttpClientBeanHolder - [HttpClientBeanHolder] Destruction of the end
2023-01-31 15:51:56.018 [MY-SpringContextShutdownHook] [NacosServiceRegistry.java:114] INFO  c.a.cloud.nacos.registry.NacosServiceRegistry - De-registration finished.
2023-01-31 15:51:56.359 [MY-SpringContextShutdownHook] [DemoApplication.java:58] INFO  com.xxx.trade.demo.DemoApplication - 停止，休眠30s
2023-01-31 15:52:26.362 [MY-SpringContextShutdownHook] [DemoApplication.java:64] INFO  com.xxx.trade.demo.DemoApplication - 停止，休眠30s结束，销毁容器
2023-01-31 15:52:26.364 [MY-SpringContextShutdownHook] [ExecutorConfigurationSupport.java:218] INFO  o.s.scheduling.concurrent.ThreadPoolTaskScheduler - Shutting down ExecutorService 'Nacos-Watch-Task-Scheduler'
2023-01-31 15:52:26.516 [MY-SpringContextShutdownHook] [ExecutorConfigurationSupport.java:218] INFO  o.s.scheduling.concurrent.ThreadPoolTaskExecutor - Shutting down ExecutorService 'applicationTaskExecutor'
2023-01-31 15:52:26.998 [MY-SpringContextShutdownHook] [DruidDataSource.java:2071] INFO  com.alibaba.druid.pool.DruidDataSource - {dataSource-1} closing ...
```