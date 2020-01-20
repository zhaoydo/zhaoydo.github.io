---
title: tomcat8 一次日志jar包冲突，log4j、slf4j
categories:
- 开发
date: 2018-03-14 16:34:38
tags:
- tomcat
- maven
---
## 问题描述
tomcat8.5.9 , maven3.3.9  启动失败，提示如下错误
tomcat7.0.2, maven3.3.9 启动成功，无报错
错误信息：发现有2个jar包（**Detected both log4j-over-slf4j.jar AND bound slf4j-log4j12.jar on the class path, preempting StackOverflowError**）  
log4j-over-slf4j.jar 和 slf4j-log4j12.jar冲突了
<!--more-->

```
14-Mar-2018 09:57:34.247 信息 [RMI TCP Connection(4)-127.0.0.1] org.apache.jasper.servlet.TldScanner.scanJars At least one JAR was scanned for TLDs yet contained no TLDs. Enable debug logging for this logger for a complete list of JARs that were scanned but no TLDs were found in them. Skipping unneeded JARs during scanning can improve startup time and JSP compilation time.
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/Users/zhaoyd/development/code/epean_trade/modules/lms/target/lms-1.0.0-SNAPSHOT/WEB-INF/lib/slf4j-log4j12-1.7.25.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/Users/zhaoyd/development/code/epean_trade/modules/lms/target/lms-1.0.0-SNAPSHOT/WEB-INF/lib/logback-classic-1.1.11.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
SLF4J: Detected both log4j-over-slf4j.jar AND bound slf4j-log4j12.jar on the class path, preempting StackOverflowError. 
SLF4J: See also http://www.slf4j.org/codes.html#log4jDelegationLoop for more details.
14-Mar-2018 09:57:34.278 严重 [RMI TCP Connection(4)-127.0.0.1] org.apache.catalina.core.ContainerBase.addChildInternal ContainerBase.addChild: start: 
 org.apache.catalina.LifecycleException: Failed to start component [StandardEngine[Catalina].StandardHost[localhost].StandardContext[/lms]]
	at org.apache.catalina.util.LifecycleBase.start(LifecycleBase.java:167)
	at org.apache.catalina.core.ContainerBase.addChildInternal(ContainerBase.java:752)
	at org.apache.catalina.core.ContainerBase.addChild(ContainerBase.java:728)
	at org.apache.catalina.core.StandardHost.addChild(StandardHost.java:734)
	at org.apache.catalina.startup.HostConfig.manageApp(HostConfig.java:1702)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.apache.tomcat.util.modeler.BaseModelMBean.invoke(BaseModelMBean.java:300)
	at com.sun.jmx.interceptor.DefaultMBeanServerInterceptor.invoke(DefaultMBeanServerInterceptor.java:819)
	at com.sun.jmx.mbeanserver.JmxMBeanServer.invoke(JmxMBeanServer.java:801)
	at org.apache.catalina.mbeans.MBeanFactory.createStandardContext(MBeanFactory.java:482)
	at org.apache.catalina.mbeans.MBeanFactory.createStandardContext(MBeanFactory.java:431)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.apache.tomcat.util.modeler.BaseModelMBean.invoke(BaseModelMBean.java:300)
	at com.sun.jmx.interceptor.DefaultMBeanServerInterceptor.invoke(DefaultMBeanServerInterceptor.java:819)
	at com.sun.jmx.mbeanserver.JmxMBeanServer.invoke(JmxMBeanServer.java:801)
	at javax.management.remote.rmi.RMIConnectionImpl.doOperation(RMIConnectionImpl.java:1468)
	at javax.management.remote.rmi.RMIConnectionImpl.access$300(RMIConnectionImpl.java:76)
	at javax.management.remote.rmi.RMIConnectionImpl$PrivilegedOperation.run(RMIConnectionImpl.java:1309)
	at javax.management.remote.rmi.RMIConnectionImpl.doPrivilegedOperation(RMIConnectionImpl.java:1401)
	at javax.management.remote.rmi.RMIConnectionImpl.invoke(RMIConnectionImpl.java:829)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at sun.rmi.server.UnicastServerRef.dispatch(UnicastServerRef.java:324)
	at sun.rmi.transport.Transport$1.run(Transport.java:200)
	at sun.rmi.transport.Transport$1.run(Transport.java:197)
	at java.security.AccessController.doPrivileged(Native Method)
	at sun.rmi.transport.Transport.serviceCall(Transport.java:196)
	at sun.rmi.transport.tcp.TCPTransport.handleMessages(TCPTransport.java:568)
	at sun.rmi.transport.tcp.TCPTransport$ConnectionHandler.run0(TCPTransport.java:826)
	at sun.rmi.transport.tcp.TCPTransport$ConnectionHandler.lambda$run$0(TCPTransport.java:683)
	at java.security.AccessController.doPrivileged(Native Method)
	at sun.rmi.transport.tcp.TCPTransport$ConnectionHandler.run(TCPTransport.java:682)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
	at java.lang.Thread.run(Thread.java:745)
Caused by: java.lang.ExceptionInInitializerError
	at org.slf4j.impl.StaticLoggerBinder.<init>(StaticLoggerBinder.java:72)
	at org.slf4j.impl.StaticLoggerBinder.<clinit>(StaticLoggerBinder.java:45)
	at org.slf4j.LoggerFactory.bind(LoggerFactory.java:150)
	at org.slf4j.LoggerFactory.performInitialization(LoggerFactory.java:124)
	at org.slf4j.LoggerFactory.getILoggerFactory(LoggerFactory.java:412)
	at ch.qos.logback.classic.util.StatusViaSLF4JLoggerFactory.addStatus(StatusViaSLF4JLoggerFactory.java:32)
	at ch.qos.logback.classic.util.StatusViaSLF4JLoggerFactory.addInfo(StatusViaSLF4JLoggerFactory.java:20)
	at ch.qos.logback.classic.servlet.LogbackServletContainerInitializer.onStartup(LogbackServletContainerInitializer.java:32)
	at org.apache.catalina.core.StandardContext.startInternal(StandardContext.java:5178)
	at org.apache.catalina.util.LifecycleBase.start(LifecycleBase.java:150)
	... 42 more
Caused by: java.lang.IllegalStateException: Detected both log4j-over-slf4j.jar AND bound slf4j-log4j12.jar on the class path, preempting StackOverflowError. See also http://www.slf4j.org/codes.html#log4jDelegationLoop for more details.
	at org.slf4j.impl.Log4jLoggerFactory.<clinit>(Log4jLoggerFactory.java:54)
	... 52 more
```
## 问题分析  
maven命令：mvn dependency:tree  查看项目中jar的依赖关系（如果项目依赖了其他项目，需要先mvn install安装依赖的项目），分别找到两个冲突的jar，发现：  
pring-boot-starter依赖的的log4j-over-slf4j  
webmagic-core依赖的的slf4j-log4j12  

```
[INFO] |  |  +- org.springframework.boot:spring-boot-starter:jar:1.5.9.RELEASE:c                                                                                                                                                                                               ompile[INFO] |  |  |  +- org.springframework.boot:spring-boot-starter-logging:jar:1.5.                                                                                                                                                                                               9.RELEASE:compile[INFO] |  |  |  |  +- ch.qos.logback:logback-classic:jar:1.1.11:compile[INFO] |  |  |  |  |  \- ch.qos.logback:logback-core:jar:1.1.11:compile[INFO] |  |  |  |  +- org.slf4j:jul-to-slf4j:jar:1.7.25:compile[INFO] |  |  |  |  \- org.slf4j:log4j-over-slf4j:jar:1.7.25:compile
```
```
[INFO] +- us.codecraft:webmagic-core:jar:0.7.2:compile[INFO] |  +- org.apache.httpcomponents:httpclient:jar:4.5.3:compile[INFO] |  |  \- org.apache.httpcomponents:httpcore:jar:4.4.8:compile[INFO] |  +- us.codecraft:xsoup:jar:0.3.1:compile[INFO] |  +- org.slf4j:slf4j-api:jar:1.7.25:compile[INFO] |  +- org.slf4j:slf4j-log4j12:jar:1.7.25:compile
```
## 解决方法  
引入webmegic-core的时候排除掉slf4j-log4j12即可   
```
<dependency>
            <groupId>us.codecraft</groupId>
            <artifactId>webmagic-core</artifactId>
            <version>0.7.2</version>
			<exclusions>
				<exclusion>
					<groupId>org.slf4j</groupId>
					<artifactId>slf4j-log4j12</artifactId>
				</exclusion>
			</exclusions>
        </dependency>
```