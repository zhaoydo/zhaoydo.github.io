---
title: jenkins持续集成maven项目
date: 2018-07-28 22:24:16
tags:
---
开发中需要频繁对项目进行打包并部署到测试服务器，这些重复的工作可以用jenkins来完成。
<!--more--> 
简单介绍jenkins对git管理的一个maven项目打包和部署  
开发环境：
1. jdk
2. maven
3. git  

# 安装jenkins
下载[jenkins.war](https://mirrors.tuna.tsinghua.edu.cn/jenkins/war/2.134/jenkins.war)  
jenkins内嵌了jetty，可以直接用java命令启动：  
```
java -jar jenkins.war --httpPort=8080
```

第一次启动jenkins时，会自动生成一个密码，会控制台中输出。浏览器输入jenkins地址：  
[http://localhost:8080](http://localhost:8080)  
进入jenkins安装页面，选择默认的插件配置（记得有两个：默认插件配置|自定义）  

# 配置环境
首先配置jdk、git、maven环境
首页->系统管理->全局工具配置
{% asset_img global_tool_config.png %}

# 配置项目
## 新建一个任务  
{% asset_img new_task.png %}  

## 源码管理
{% asset_img task_git.png %}
Repository URL 是git仓库地址，可以是http或者ssh
Credentials 是git凭据。  
- 如果是http类型的git仓库地址，使用Username with password类型的凭据  
- 如果是ssh类型的git仓库地址，使用SSH Username with private key类型的凭据， private是本机的ssh私钥  

默认master分支  

## 构建触发器  
{% asset_img task_trigger.png %}
每两小时检查一次git，有新提交就触发构建  

## 构建  
{% asset_img task_build.png %}
目标：maven构建命令  
POM：制定一个pom.xml 默认为根目录下pom.xml  

## 构建后操作(部署到服务器)
两种部署到服务器的方式
### 复制文件到服务器
项目通过jenkins构建后会生成war包，可以直接上传到服务器并复制到tomcat/webapps下，tomcat会自动部署项目  
**先配置服务器**  
1. 首页->插件管理->可选插件 安装Publish Over SSH插件  
2. 首页->系统管理 配置服务器地址、端口、用户名、密码  

{% asset_img ssh_config.png %}
任务配置->构建后操作增加传输的服务器的设置
{% asset_img task_after_build.png %}
source files是本地待传输war包的路径  
remove prefix 设置上传到服务器后要移除的路径  
remote directory 是上传到服务器的文件夹，上一步ssh server设置的远程路径是/home，所以war文件将上传到/home/package/文件夹下  
Exec command 填写上传后执行的命令，这里将war包复制到tomcat/webapps下  

### 通过tomcat部署
**首先配置tomcat管理用户**
修改tomcat根目录下conf/tomcat-user.xml文件，增加管理用户  
重启后进入tomcat首页，验证是否修改成功
{% asset_img tomat_manage.png %}
配置构建后操作
{% asset_img deploy_tomcat.png %}
WAR/EAR 是要部署的war包  
Context path 是项目根路径  
Credentials 配置tomcat的管理用户和密码  
Tomcat URL配置tomcat的管理url(一般是http:\//ip:port)