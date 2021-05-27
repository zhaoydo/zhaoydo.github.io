---
title: 一些常用命令
date: 2018-11-01 19:33:13
tags:
---
记录日常用的命令，备忘

<!--more-->

# 日常使用

```bash
# 统计字符在文件内出现次数
cat cps-info.2019-08-13* |grep "字符" |wc -l
# 子文件及文件夹占磁盘大小
du -h --max-depth=1
# 
ll -Art | tail -n 10
# catalina.out日志置空
cat /dev/null > catalina.out
# 查看日志前后
grep -C 5 foo file # 显示file文件中匹配foo字串那行以及上下5行
grep -B 5 foo file # 显示foo及前5行
grep -A 5 foo file # 显示foo及后5行
# 查看线程总数
ps -eLf | wc -l
top -H
# 查看进程线程数
ps -o nlwp -PID
top -Hp -PID
# 查看服务器版本
cat /etc/redhat-release
```

# JVM相关

```bash
# jstack日志
jstack -l [pid] > xx.log 
# 导出未回收的对象
jmap -histo [pid] > gc.log
# 导出当前堆情况
jmap -dump:format=b,file=xxx.bin [pid]
# 查看gc情况 每5000ms刷新
jstat -gc [pid] 5000
# 查看gc原因
jstat -gccause [pid] 5000
```

jstatc gc相关的一些命令解析  
https://www.cnblogs.com/ListenWind/p/5230118.html  
https://blog.csdn.net/zhanlanmg/article/details/48463905

# Jconsole远程配置

远程主机tomcat配置，以便于本地jconsole连接查看jvm运行情况

```
JAVA_OPTS="-Xms1024m -Xmx5080m -Xss1024k -XX:MetaspaceSize=512m -XX:MaxMetaspaceSize=2048m"
CATALINA_OPTS="$JAVA_OPTS -Djava.rmi.server.hostname=114.55.237.143 -Dcom.sun.management.jmxremote"
CATALINA_OPTS="$CATALINA_OPTS -Dcom.sun.management.jmxremote.port=18082"
CATALINA_OPTS="$CATALINA_OPTS -Dcom.sun.management.jmxremote.authenticate=true"
CATALINA_OPTS="$CATALINA_OPTS -Dcom.sun.management.jmxremote.ssl=false"
CATALINA_OPTS="$CATALINA_OPTS -Dcom.sun.management.jmxremote.pwd.file=/usr/local/epean/software/jdk1.8.0_181/jre/lib/management/jmxremote.password"
```

需要注意的地方：  

1. 设置CATALINA_OPTS，而不是JAVA_OPTS， JAVA_OPTS会在showdown.sh时也生效，关闭tomcat时提示jmxremote端口占用
2. 复制jmxremote.password.template为jmxremote.password，文件为只读，修改时需要!wq强制保存。 修改后要chmod 0400 jmxremote.password 修改文件为tomcat启动用户只读，否则启动报错
3. 如果启动不成功，看一下catalina.out日志

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

安装：  

参考官方文档：http://nginx.org/en/docs/configure.html，各项配置都有解释

```bash
# 下载好tar.gz文件解压后进入目录
# 自定义一些配置，各项配置都有默认值，参考官方文档
./configure
    --sbin-path=/usr/local/nginx/nginx
    --conf-path=/usr/local/nginx/nginx.conf
    --pid-path=/usr/local/nginx/nginx.pid
    --with-http_ssl_module
    --with-pcre=../pcre-8.44
    --with-zlib=../zlib-1.2.11
# 编译
make
# 安装
make install
```

使用：  

进入到nginx的sbin目录

```bash
# 启动
./nginx
# 停止
./nginx -s stop
# 检查配置文件是否正确
./nginx -t
# 重新加载配置文件
./nginx -s reload
# 查看版本信息
./nginx -v
# 查看版本和编译信息（安装时configure的配置）
./nginx -V
# 查看所有命令说明
./nginx -h
```

升级了公司的nginx支持水印图模块，其实就是用旧版本的configure信息对新版本nginx源码编译，得到nginx二进制文件，替代原来的nginx文件  

nginx升级参考以及遇到的坑：  
1、https://blog.51cto.com/14482279/2435975  
2、https://blog.csdn.net/edwin19911212/article/details/82811818  
3、https://www.jianshu.com/p/999339387920

# mysql



# npm

```bash
npm outdated # 检查模块是否已经过时
npm update 模块名  #更新模块
npm update 模块名@版本号  # 更新到某个版本
npm update 模块名@latest  # 最新版本
npm update 模块名 --save # 更新并写入package.json的dependencies模块
```



