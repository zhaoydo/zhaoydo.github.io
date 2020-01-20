---
title: 一些常用命令
date: 2018-11-01 19:33:13
tags:
---
# firewall防火墙
查看防火墙状态
```
firewall -cmd --state
```
关闭防火墙
```
systemctl stop firewalld.service
```
开启防火墙
```
systemctl start firewalld.service
```
重启防火墙
```
firewall-cmd --reload
```
查看防火墙已经开放了的端口
```
firewall-cmd --list-port
```
查看防火墙规则
```
firewall-cmd --list-all
```
开放防火墙端口
```
# 不加--permanent参数下次重启失效
firewall-cmd --zone=public --add-port=80/tcp --permanent
```
关闭防火墙端口
```
firewall-cmd --zone=public --remove-port=80/tcp --permanent
```
查看firewall更多命令
```
firewall-cmd --help
```
# nginx

# mysql