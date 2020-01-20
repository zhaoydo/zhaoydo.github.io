---
title: eclipse修改类不重启tomcat 自动加载项目
categories:
- 开发
date: 2016-07-26 16:34:38
tags:
- tomcat
---
**修改两个地方实现修改java代码时tomcat只加载被修改的类，不会重新部署整个项目:**  
1、Package Explorer 打开Servers项目，修改server.xml  
{% asset_img 1.png 图1 image %}
<!--more-->
2、server视图，tomcat上右键open，选择Modules，将auto reloads改为disabled  
{% asset_img 2.png 图2 image %}  
注意：debug模式下有效，tomcat需要重启一下  

PS:推荐用jetty