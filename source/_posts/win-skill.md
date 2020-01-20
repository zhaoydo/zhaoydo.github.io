---
title: windows系统kill占用特定端口的进程
categories:
- 开发
date: 2018-06-06 09:39:00
tags:
- windows
---
windows环境启动server提示55853端口被占用，尝试杀掉占用此端口的进程
<!--more-->
## 根据端口号查找进程号
```
netstat -ano | findstr 55853
```
输出
```
TCP    192.168.0.17:55853     192.168.0.248:8082     ESTABLISHED     10756
```
得到进程号10756  

## 根据进程号找进程名
```
tasklist | findstr 10756
```
输出
```
seaf-daemon.exe              10756 Console                    1      9,696 K
```
得到进程名seaf-daemon.exe
## 杀掉进程
任务管理器找到进程，杀掉。  
如果找不到，直接根据进程号强制杀掉
```
taskkill -PID <进程号> -F //强制关闭某个进程  
```
结束