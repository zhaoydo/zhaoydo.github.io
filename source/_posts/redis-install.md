---
title: Linux CentOS7下 redis安装
categories:
- 数据库
date: 2018-01-24 16:41:38
tags:
- redis
---
<!--more-->
安装参考：https://redis.io/download  
# 安装  
CentOs7.2，redis4.0.6  
cd进入要安装redis的文件夹，也可安装之后再修改路径  
开始安装  
{% codeblock %}
$ wget http://download.redis.io/releases/redis-4.0.6.tar.gz
$ tar xzf redis-4.0.6.tar.gz
$ cd redis-4.0.6
$ make
{% endcodeblock %}  

安装结束  
# 测试  
cd进入redis-4.0.6根目录，redis执行文件（redis-server、redis-cli）都在src目录中  
启动redis-server  
{% codeblock %}
$ src/redis-server
{% endcodeblock %}  
另开命名窗口启动redis-cli  

{% codeblock %}
$ src/redis-cli
redis> set foo bar
OK
redis> get foo
"bar"
{% endcodeblock %}

# 配置  
https://redis.io/topics/config 参考redis.conf自述文件  
https://www.jianshu.com/p/68d214f09032