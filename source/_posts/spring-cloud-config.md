---
title: spring cloud配置中心示例
date: 2018-08-08 20:10:38
tags:
---
spring cloud配置中心示例  
[示例代码](https://gitee.com/zhaoydo/spring-cloud-config)  
<!--more-->
**server端:**  
管理所有项目的配置文件
**client端**  
就是我们需要运行的项目，项目所有的配置文件放到配置中心内，启动时从server端读取配置  

首先在git新建一个文件夹config-repo存放项目的配置文件：
myconfig-dev.properties  
myconfig-pro.properties  
分别配置
```
hello=helloworld dev
```
```
hello=helloworld pro
```

# server端
## 添加依赖
 新建一个springboot的maven项目，添加spring-cloud-config-server依赖  
 ```xml
 <dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-config-server</artifactId>
	<version>2.0.0.RELEASE</version>
	</dependency>
 ```
## 配置文件
application.properties配置：
```
server.port=8888
spring.application.name=spring-cloud-config-server
spring.cloud.config.uri=http://localhost:8888
# 配置文件git路径
spring.cloud.config.server.git.uri=https://gitee.com/zhaoydo/spring-cloud-config.git
# 配置文件git根目录下相对路径
spring.cloud.config.server.git.search-paths=config-repo
# git用户账号
spring.cloud.config.server.git.username=zhaoyundi@126.com
spring.cloud.config.server.git.password=password
```
## 启动类
启动类增加@EnableConfigServer注解
```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.config.server.EnableConfigServer;

@EnableConfigServer
@SpringBootApplication
public class SpringCloudConfigServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(SpringCloudConfigServerApplication.class, args);
	}
}
```
## 测试
启动server，访问：http://localhost:8888/myconfig/dev  
返回：
```json
{
    "name":"myconfig",
    "profiles":[
        "dev"
    ],
    "label":null,
    "version":"bda92cce03b36c2cc8d047c3dc483c9a16c14540",
    "state":null,
    "propertySources":[
        {
            "name":"https://gitee.com/zhaoydo/spring-cloud-config.git/config-repo/myconfig-dev.properties",
            "source":{
                "hello":"helloworld dev"
            }
        }
    ]
}
```
正常读取到git中的myconfig-dev.properties配置文件  
# client端
## 添加依赖
```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-config</artifactId>
	<version>2.0.0.RELEASE</version>
</dependency>
```
## 配置文件
client读取server的配置放在bootstrap,properties中，此配置文件会在application.properties之前加载
application.properties:
```
spring.application.name=spring-cloud-config-client
server.port=8080
```
bootstrap.proerties
```
spring.cloud.config.uri=http://localhost:8888
spring.cloud.config.name=myconfig
spring.cloud.config.profile=dev
spring.cloud.config.label=master
```
spring.application.name：对应{application}部分  
spring.cloud.config.profile：对应{profile}部分  
spring.cloud.config.label：对应git的分支。如果配置中心使用的是本地存储，则该参数无用  
spring.cloud.config.uri：配置中心server地址  
## 测试
client端当前配置的是读取：myconfig-dev.properties
```
hello=helloworld dev
```
client端新建一个Controller代码：
```java
@RestController
public class Index {
    @Value("${hello}")
    private String hello;
    @RequestMapping("index")
    public String index(){
        return hello;
    }
}
```
启动client项目访问：http://localhost:8080/index
返回：helloworld dev

# 安全验证
配置中心任何人都能访问时不安全的，可以加一个安全验证
## server
增加spring-boot-starter-security依赖：
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
```
配置文件配置用户名密码：
```
pring.security.user.name=zhaoyundi
spring.security.user.password=123456
```
现在访问：http://localhost:8888/myconfig/dev 就需要输入用户名密码了
## client
bootstrap.properties增加
```
spring.cloud.config.username=zhaoyundi
spring.cloud.config.password=123456
```
配置完成
参考：  
[纯洁的微笑：配置中心git示例](http://www.ityouknow.com/springcloud/2017/05/22/springcloud-config-git.html)  
[spring-cloud-config reference documents](http://cloud.spring.io/spring-cloud-static/spring-cloud-config/2.0.1.RELEASE/single/spring-cloud-config.html)