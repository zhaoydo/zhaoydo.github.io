---
title: Linux CentOS7下 mysql-5.7.1x tar.gz包安装
categories:
- 数据库
date: 2016-07-19 16:34:38
tags:
- mysql
---
# 1、下载tar.gz包
官网下载MySQL安装包，Linux-Generic 64位（根据系统选择64or32）   
也可以用wget命令下载  
64位下载：
```
wget http://120.52.72.21/cdn.mysql.com/c3pr90ntc0td//Downloads/MySQL-5.7/mysql-5.7.13-linux-glibc2.5-x86_64.tar.gz
```
# 2、创建mysql用户和组
```
groupadd mysql
useradd -g mysql mysql
```
<!--more-->
# 3、解压mysql.tar.gz
注意：文件路径按照自己实际路径来，我安装后的的目录结构是  

```
/software/mysql  
      ----data  
      ----mysql-5.7.13
```

```
tar zxvf mysql-5.7.13-linux-glibc2.5-x86_64.tar.gz
mv mysql-5.7.13-linux-glibc2.5-x86_64 mysql-5.7.13
```
将mysql-5.7.13移动到合适的位置准备安装

将mysql-5.7.13/support-files/my-default.cnf 移动并重命名到/etc/my.cnf
```
cp mysql-5.7.13/support-files/my-default.cnf /etc/my.cnf
```
新建mysql目录，移动mysql-5.7.13到mysql下，在mysql下新建data目录 与下面basedir,datadir一致
修改my.cnf文件的basedir，datadir（mysql根目录，data根目录）：  
basedir = /software/mysql/mysql-5.7.13  
datadir = /software/mysql/data  
**至此准备工作完成。**  
# 4、安装mysql  
给mysql用户赋权限

```
chown -R mysql mysql/
```

先进入mysql bin目录  切换到mysql
```
cd mysql/mysql-5.7.13/bin
su mysql
```
先初始化一下
```
./mysql_install_db --user=mysql --basedir=/software/mysql/mysql-5.7.13 --datadir=/software/mysql/data
```
打印出
```
2016-07-19 03:43:07 [WARNING] mysql_install_db is deprecated. Please consider switching to mysqld --initialize
2016-07-19 03:43:10 [WARNING] The bootstrap log isn't empty:
2016-07-19 03:43:10 [WARNING] 2016-07-19T03:43:07.693147Z 0 [Warning] --bootstrap is deprecated. Please consider using --initialize instead
2016-07-19T03:43:07.693815Z 0 [Warning] Changed limits: max_open_files: 1024 (requested 5000)
2016-07-19T03:43:07.693825Z 0 [Warning] Changed limits: table_open_cache: 431 (requested 2000)
```
如果报这个错（linux libaio.so.1: cannot open shared object file: No such file or directory），缺少包，切换到root账号执行下面命令
```
yum install libaio*
```
安装
如果没有上一步的初始化会报错（Can't open and lock privilege tables: Table 'mysql.user' doesn't exist）
```
./mysqld --user=mysql --basedir=/software/mysql/mysql-5.7.13 --datadir=/software/mysql/data
```
打印出
```
2016-07-19T03:43:22.592404Z 0 [Warning] Changed limits: max_open_files: 1024 (requested 5000)
2016-07-19T03:43:22.592545Z 0 [Warning] Changed limits: table_open_cache: 431 (requested 2000)
2016-07-19T03:43:22.796657Z 0 [Warning] TIMESTAMP with implicit DEFAULT value is deprecated. Please use --explicit_defaults_for_timestamp server option (see documentation for more details).
2016-07-19T03:43:22.796713Z 0 [Warning] 'NO_ZERO_DATE', 'NO_ZERO_IN_DATE' and 'ERROR_FOR_DIVISION_BY_ZERO' sql modes should be used with strict mode. They will be merged with strict mode in a future release.
2016-07-19T03:43:22.796717Z 0 [Warning] 'NO_AUTO_CREATE_USER' sql mode was not set.
2016-07-19T03:43:22.796755Z 0 [Warning] Insecure configuration for --secure-file-priv: Current value does not restrict location of generated files. Consider setting it to a valid, non-empty path.
2016-07-19T03:43:22.796802Z 0 [Note] ./mysqld (mysqld 5.7.13) starting as process 6151 ...
2016-07-19T03:43:22.803685Z 0 [Note] InnoDB: PUNCH HOLE support not available
2016-07-19T03:43:22.803740Z 0 [Note] InnoDB: Mutexes and rw_locks use GCC atomic builtins
2016-07-19T03:43:22.803746Z 0 [Note] InnoDB: Uses event mutexes
2016-07-19T03:43:22.803750Z 0 [Note] InnoDB: GCC builtin __sync_synchronize() is used for memory barrier
2016-07-19T03:43:22.803753Z 0 [Note] InnoDB: Compressed tables use zlib 1.2.3
2016-07-19T03:43:22.803757Z 0 [Note] InnoDB: Using Linux native AIO
2016-07-19T03:43:22.804286Z 0 [Note] InnoDB: Number of pools: 1
2016-07-19T03:43:22.804513Z 0 [Note] InnoDB: Using CPU crc32 instructions
2016-07-19T03:43:22.807882Z 0 [Note] InnoDB: Initializing buffer pool, total size = 128M, instances = 1, chunk size = 128M
2016-07-19T03:43:22.824980Z 0 [Note] InnoDB: Completed initialization of buffer pool
2016-07-19T03:43:22.845095Z 0 [Note] InnoDB: If the mysqld execution user is authorized, page cleaner thread priority can be changed. See the man page of setpriority().
2016-07-19T03:43:22.857908Z 0 [Note] InnoDB: Highest supported file format is Barracuda.
2016-07-19T03:43:22.872927Z 0 [Note] InnoDB: Creating shared tablespace for temporary tables
2016-07-19T03:43:22.872995Z 0 [Note] InnoDB: Setting file './ibtmp1' size to 12 MB. Physically writing the file full; Please wait ...
2016-07-19T03:43:22.920946Z 0 [Note] InnoDB: File './ibtmp1' size is now 12 MB.
2016-07-19T03:43:22.922389Z 0 [Note] InnoDB: 96 redo rollback segment(s) found. 96 redo rollback segment(s) are active.
2016-07-19T03:43:22.922408Z 0 [Note] InnoDB: 32 non-redo rollback segment(s) are active.
2016-07-19T03:43:22.926137Z 0 [Note] InnoDB: Waiting for purge to start
2016-07-19T03:43:22.976327Z 0 [Note] InnoDB: 5.7.13 started; log sequence number 2523957
2016-07-19T03:43:22.976809Z 0 [Note] Plugin 'FEDERATED' is disabled.
2016-07-19T03:43:22.980544Z 0 [Note] InnoDB: Loading buffer pool(s) from /software/mysql/data/ib_buffer_pool
2016-07-19T03:43:22.985598Z 0 [Note] InnoDB: Buffer pool(s) load completed at 160719  3:43:22
2016-07-19T03:43:22.994529Z 0 [Note] Found ca.pem, server-cert.pem and server-key.pem in data directory. Trying to enable SSL support using them.
2016-07-19T03:43:22.997854Z 0 [Warning] CA certificate ca.pem is self signed.
2016-07-19T03:43:23.000124Z 0 [Note] Server hostname (bind-address): '*'; port: 3306
2016-07-19T03:43:23.000239Z 0 [Note] IPv6 is available.
2016-07-19T03:43:23.000261Z 0 [Note]   - '::' resolves to '::';
2016-07-19T03:43:23.000282Z 0 [Note] Server socket created on IP: '::'.
2016-07-19T03:43:23.014390Z 0 [Note] Event Scheduler: Loaded 0 events
2016-07-19T03:43:23.014866Z 0 [Note] ./mysqld: ready for connections.
Version: '5.7.13'  socket: '/tmp/mysql.sock'  port: 3306  MySQL Community Server (GPL)
```
安装成功
# 5、修改初始密码
mysql旧版本安装之后root初始密码为空，直接登录就可以，5.7以后版本安装后会分配一个随机密码
切换到root输入命令：
```
cat /root/.mysql_secret
```
打印  
```
Password set for user 'root@localhost' at 2016-07-19 02:50:11 
oezidZnrF:+a
```
初始密码为：oezidZnrF:+a  

进入mysql bin目录，登录mysql

```
su mysql
cd mysql/mysql-5.7.13/bin
./mysql -uroot -p
```
输入初始密码登录，设置新密码
```
mysql>SET PASSWORD = PASSWORD('newpasswd'); 
```
更多关于重设密码参考另一篇文章：[Linux下mysql5.7.x版本忘记root初始密码](http://blog.csdn.net/yisheyuanzhang/article/details/50560835)  
# 6、添加系统服务
切换到root，进入mysql目录

```
cp ./support-files/mysql.server /etc/init.d/mysql
chown mysql /etc/init.d/mysql
```
将mysql.server拷贝到/etc/init.d/下并重命名为mysql
赋给mysql用户权限
切换到mysql用户，重启mysql
```
su mysql
service mysql restart
```
另外两个命令是：
service mysql start
service mysql stop  
安装结束