---
title: (译)RabbitMq官方使用教程
date: 2021-01-10
tags:
---

*翻译[RabbitMQ Tutorials](https://www.rabbitmq.com/getstarted.html)*

# RabbotMQ 教程

这些教程介绍了使用RabbitMQ创建消息发送应用程序的基础知识。  

如果需要RabbitMq server安装教程，请参阅[安装教程](https://www.rabbitmq.com/download.html)或者使用[Docker image](https://registry.hub.docker.com/_/rabbitmq/)

教程的[可执行版本](https://github.com/rabbitmq/rabbitmq-tutorials)和[官方网站](https://github.com/rabbitmq/rabbitmq-website)一样都是开源的  
<!--more-->
*译者注：官方文档提供了不同开发语言的示例，以下只翻译java的*

## Hello World

### 介绍

RabbitMq是一个消息代理：接收并转发消息。你可以把它理解成一个邮局：当你把想要投递的邮件放入一个邮箱，你可以确定邮递员最终会把这个邮件送达给接收人。在这个类比中，RabbitMQ扮演者信箱、邮局、邮递员的角色。  

RabbitMQ和邮局最主要的区别在于，它不处理纸张，而是接收、存储、转发二进制数据块消息。  

一般情况下，RabbitMQ、消息收发用到的术语：

- *Producing* 发送。一个发送消息的程序叫*producer* 生产者

  ![](https://www.rabbitmq.com/img/tutorials/producer.png)

- *Queue* 队列是RabbitMQ中信箱的名称。尽管消息通过RabbitMQ和应用程序，但是只储存在队列中。队列只受主机内存和磁盘的限制，本质上是一块大的消息buffer。多个生产者可以向一个队列发送消息，多个消费者可以尝试从一个队列接收数据。我们用下图表示队列:  

  ![queue_name](https://www.rabbitmq.com/img/tutorials/queue.png)

- *Consuming* 表示接收（receiving）。 消费者（consumer）大多是一个程序、等待接收消息。

  ![C](https://www.rabbitmq.com/img/tutorials/consumer.png)

  

  注意，生产者(producer)、消费者(consumer)、代理(broker)不必在同一台主机上；实际上，大部分应用程序中也不在一台机器上。一个应用程序也可以同时是生产者和消费者。

  ### "Hello World"

  （使用java客户端实现）  

  在教程的这部分，我们将编写两个java程序；一个生产者发送一条消息，一个消费者接受消息并打印出来。我们会忽略Java API的一些细节，主要关注这个简单的东西。下面是一个“Hello World”消息。  

  下图中，“P”代表生产者、“C”代表消费者，中间的盒子表示队列-一个RabbitMQ代表消费者保留的消息缓冲区  

  ![](https://www.rabbitmq.com/img/tutorials/python-one.png)

  

> **The Java client library**
>
> RabbitMQ 支持多种协议。本教程使用AMQP 0-9-1，是一个用于消息传递的开放、通用协议。针对不同开发语言有不同的RabbitMQ客户端。我们将使用RabbitMQ提供的Java客户端。  
>
> 下载客户端及其依赖（SLF4J API和SLF4J Simple）。 复制这些文件以及教程中的Java文件到你的工作目录。  
>
> 请注意 对于本教程来说SLF4JSimple是足够的，但是在在生产环境中你应该使用例如Logback这样的完整的日志库。  
>
> （RabbitMQ的java客户端也在Maven的中央仓库中，groupId：com.rabbitmq，artifactId:amqp-client）  

现在，我们有Java客户端库和依赖，可以开始写代码了。

### Sending

![sending](https://www.rabbitmq.com/img/tutorials/sending.png)

我们将调用消息发布者(sender) 发送，消息消费者（receiver）接收。发布者将连接RabbitMQ并发送一条消息，然后退出。  

在Send.java中，我们需要引入一些class  

```java
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.Channel;
```

编写类和命名队列

```java
public class Send {
  private final static String QUEUE_NAME = "hello";
  public static void main(String[] argv) throws Exception {
      ...
  }
}
```

然后我们可以创建到一个到server的连接

```java
ConnectionFactory factory = new ConnectionFactory();
factory.setHost("localhost");
try (Connection connection = factory.newConnection();
     Channel channel = connection.createChannel()) {

}
```

该连接抽象了socket连接，为我们提供了协议版本协商和认证等功能。这里我们连接一个本地机器上的RabbitMQ节点（指localhost）。如果我们想连接节点到其他的机器，只需要指定主机名或者ip地址。  

接下来我们将创建一个channel，它是大部分API处理的地方（翻译存疑）。注意我们可以使用 try-with-resources语句，因为连接（connection）和通道（channel）都实现了java.io.Closeable，我们 不需要在代码中显式关闭他们。  

要发送消息，我们必须声明要消息发送到的队列； 然后才可以将消息发布到该队列，主要所有这些代码都在try with resources语句中：  

*（译者注：try with resources语句是java7新特性，try(resource)可以在执行结束后自动关闭resource，而不用显式的关闭）*  

```java
channel.queueDeclare(QUEUE_NAME, false, false, false, null);
String message = "Hello World!";
channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
System.out.println(" [x] Sent '" + message + "'");
```

声明一个队列是幂等的（idempotent）- 只有不存在时才会被创建。 消息内容是一个字节数组，因此你可以编码任何你喜欢的内容。  

[这里是完整的Send.java类](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/Send.java)

> **发送不起作用**
>
> 如果你是第一次使用RabbitMQ，没有看到“Send”消息你看会挠头，疑惑哪里可能出问题了。可能代理启用时没有足够的磁盘空间（默认需要至少200MB可用空间）导致拒绝接受消息。检查代理日志文件确认，可以在必要的时候减少此限制。[配置文档](https://www.rabbitmq.com/configure.html#config-items)将向你展示如何设置disk_free_limit  

### Receiving

以上是我们的发布者。我们的消费者监听来自RabbitMQ的消息，因此与单个消息的发布者不同，我们将保持消费者运行以便监听消息并且打印出来。  

![receiving](https://www.rabbitmq.com/img/tutorials/receiving.png)

接收端的代码（in Recv.java）跟发送端代码要引入的类基本一致

```java
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.DeliverCallback;
```

我们将额外使用DeliverCallback接口来缓冲服务器推送给我们的消息。  

代码配置与发布服务器相同；我们打开一个连接和通道，并声明我们要消费的队列。注意，队列要与send发布的队列相匹配。  

```java
public class Recv {

  private final static String QUEUE_NAME = "hello";

  public static void main(String[] argv) throws Exception {
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("localhost");
    Connection connection = factory.newConnection();
    Channel channel = connection.createChannel();

    channel.queueDeclare(QUEUE_NAME, false, false, false, null);
    System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

  }
}
```

注意，我们在消费端也声明了队列，因为我们可能会在发布端之前启动消费端，所以要在尝试消费消息之前确保队列存在。（*译者注：前面提到队列的声明是幂等的，存在则不重复创建*）  

为什么不使用try with resource语句来自动关闭通道和连接？ 如果这样做会让程序继续执行，关闭所有内容然后退出！这是不合适的，因为我们期望消费者监听的异步消息到达时，进程保持活动状态。（*译者注：文档太啰嗦，发送端发送1个消息连接一次，消费端始终连接侦听*）  

我们要告诉服务器把队列中的消息投递给我们。由于服务器推送消息是异步的，我们以对象的形式提供一个回调，以便于缓冲信息知道我们使用它。这就是DeliverCallback。  

```java
DeliverCallback deliverCallback = (consumerTag, delivery) -> {
    String message = new String(delivery.getBody(), "UTF-8");
    System.out.println(" [x] Received '" + message + "'");
};
channel.basicConsume(QUEUE_NAME, true, deliverCallback, consumerTag -> { });
```

[这里是完整的Recv.java类](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/java/Recv.java)  

### 放一起运行

你可以指定RabbitMQ java客户端编译这两个文件  

```sh
javac -cp amqp-client-5.7.1.jar Send.java Recv.java
```

运行他们，你需要rabbitmq-client.jar 以及它的依赖。在终端中运行消费者（接收者）  

```sh
java -cp .:amqp-client-5.7.1.jar:slf4j-api-1.7.26.jar:slf4j-simple-1.7.26.jar Recv
```

然后运行发布者（发送者）  

```sh
java -cp .:amqp-client-5.7.1.jar:slf4j-api-1.7.26.jar:slf4j-simple-1.7.26.jar Send
```
> Windows中，使用";"代替":"来分隔类路径
消费者将打印通过RabbitMQ从消息发布者获得的消息。消费者将保持运行，等待消息（ctrl+c终止），因此请尝试从另一个中断运行发布者（*译者：没看明白*）  

> **Listing queues**  
> 你也许希望看到对了中有那些队列，队列中有多少消息。可以通过*rabbitmqctl*工具来实现（有查看权限的用户）  
>  ```
>  sudo rabbitmqctl list_queues
>  ```  
>  Windows中省略sudo  
>  ```
>  rabbitmqctl.bat list_queues
>  ```  
## Work Queues
![work queues](https://www.rabbitmq.com/img/tutorials/python-two.png)
在第一个教程中(Hello World)我们写了程序从一个指定的队列中来发送和接收消息。在本章教程中，我们将创建一个工作队列（Work Queue）用来将耗时的任务分配给多个Worker。  

Work Queues(又名：Task Queues)主要目的是：对于资源密集型任务，我们会避免立即执行并等待任务完成，而是安排任务晚一点执行。我们将任务封装成一个对象发送到队列。一个后台运行的Worker进程将取走任务并执行。当运行多个Workers时，任务将共享给每个Worker。  
在一个短Http请求窗口中要处理一个复杂的任务是不可能的,对于这种情况，Work Queues这个概念是非常有用的。  
### 准备
在教程的上一部分中，我们发送了一条包含“Hello World!”的消息。现在我们将发送一个字符串来表示复杂的任务。我们没有一个真实的任务，像调整图片大小或者渲染pdf，因此我们用Thread.sleep()方法来模拟很忙的任务。 我们将以字符串中点的数量来模拟任务的复杂性，每个点将占用1秒来工作。 例如，一个模拟任务“Hello...”,将会花费三秒钟。  

我们将略微修改一下上一个示例中的Send.java代码，来允许从命令行发送任何消息。 这个程序将吧任务安排到工作队列中，所以我们将它命名为NewTask.java:  
```java
String message = String.join(" ", argv);

channel.basicPublish("", "hello", null, message.getBytes());
System.out.println(" [x] Sent '" + message + "'");
```
我们之前的Recv.java也需要做一些变动：需要对消息体中的每个点模拟1秒的工作时间。 它将处理已投递的消息并执行任务，我们叫它Worker.java:  
```java
DeliverCallback deliverCallback = (consumerTag, delivery) -> {
  String message = new String(delivery.getBody(), "UTF-8");

  System.out.println(" [x] Received '" + message + "'");
  try {
    doWork(message);
  } finally {
    System.out.println(" [x] Done");
  }
};
boolean autoAck = true; // acknowledgment is covered below
channel.basicConsume(TASK_QUEUE_NAME, autoAck, deliverCallback, consumerTag -> { });
```
模拟任务来模拟执行时间
```java
private static void doWork(String task) throws InterruptedException {
    for (char ch: task.toCharArray()) {
        if (ch == '.') Thread.sleep(1000);
    }
}
```
编译这里两段代码，参考教程1（使用工作目录中的jar文件和环境变量CP）
```sh
javac -cp $CP NewTask.java Worker.java
```
### 循环调度
使用任务队列的一个优点是能轻松地进行并行工作。如果我们正在积累