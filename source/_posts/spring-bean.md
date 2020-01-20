---
title: 学习Spring bean配置
categories:
- spring学习
date: 2016-07-26 16:34:38
tags:
- spring
---

配置文件
```
<bean id="hello" name="hello1,hello2" class="com.zhao.chapter2.HelloWorldImpl"/>
<alias name="hello1" alias="hello3"/>
<bean id="hello4" class="com.zhao.chapter2.HelloWorldImpl">
    <constructor-arg index="0" value="HelloSpring!"/>
</bean> 
```
测试代码
```
@Test
public void sayHelloTest() {
	@SuppressWarnings("resource")
	ApplicationContext context = new ClassPathXmlApplicationContext("helloWorld.xml");
	HelloWorld hello1 = context.getBean("hello1",HelloWorld.class);
	HelloWorld hello2 = context.getBean("hello2",HelloWorld.class);
	HelloWorld hello3 = context.getBean("hello3",HelloWorld.class);
	HelloWorld hello4 = context.getBean("hello4",HelloWorld.class);
	hello1.sayHello();
	hello2.sayHello();
	hello3.sayHello();
	hello4.sayHello();
}
```
<!--more-->
运行结果
```
HelloWorld!
HelloWorld!
HelloWorld!
HelloSpring!
```
1、bean的id，name和alias标识bean  
2、IOC容器中id必须唯一（推荐用id标识bean）  
3、同一文件中name不能相同，不同文件可以相同（后面的会覆盖前面的）  
4、如果没有定义name和id，类全名作为标识符  
5、constructor-arg 构造器注入，可以根据索引（index）、参数类型（type）、名称（name）注入
6、property标签注入，<property name="message" value="Hello World!"/>  通过name注入，可注入各种类型（Spring会将字符串自动转换对应的类型）、bean的id（idref标签或者value="beanId"）、bean实例（ref或者ref="beanId"）  

