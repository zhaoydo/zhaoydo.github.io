---
title: Font导致堆外内存泄露排查解决
date: 2021-05-26 19:50:13
tags:
---
## 起因
周六收到服务器告警，线上一台32G服务器的内存占用90%+，远远超出JVM设置的内存-Xmx8192m   
top命令查看线上进程信息，发现一个java进程占用内存达到29G
<!--more-->
{% asset_img top.png 图1 image %}
  
## 分析JVM内存
jstat命令查看JVM各区内存情况
```
jstat -gc <pid> 2000
```
{% asset_img jstat.png 图2 image %}

看到堆中新生代分配了2.59G、老年代5.59G，metaspace 355M，并且各区都没有用满，考虑是堆外直接内存溢出。  

使用pmap命令输出进程内存映射到文件
```
pmap -x <pid> | sort -n -k3 > pmap.txt
```
查看文件，发现输出内容中有很多64M的内存地址。内存一样，数量很多，怀疑是这些内存泄露了  
{% asset_img pmap.png 图2 image %}  
现在要做的就是找出这些内存块中存储的内容或者创建这些内存空间线程的堆栈信息  

## 分析内存块
使用smaps命令输出内存块详细信息
```
cat /proc/<pid>/smaps > smaps.txt
```
{% asset_img smaps.png 图2 image %}
找到一个可疑的64M的空间地址，例如图上的7f93f4000000-7f93f7ffd000  
启动gdb:  
> GDB是一个由GNU开源组织发布的、UNIX/LINUX操作系统下的、基于命令行的、功能强大的程序调试工具。

```
gdb attach <pid>
```
dump内存地址到指定目录下(注意加上0x)
```
dump memory /tmp/0x7f93f4000000-0x7f93f7ffd000.dump 0x7f93f4000000 0x7f93f7ffd000
```
退出gdb
查看内存dump文件中超过10的字符串
```
strings -10 /tmp/0x7f93f4000000-0x7f93f7ffd000.dump > 10m.txt
```
{% asset_img dump.png 图2 image %}
搜索关键词，终于发现了业务相关的内容，一个字体文件。 继续dump分析了另外几个64M内存块，都发现有seguiemj.ttf相关的信息，确认是项目中对这个字体操作的相关代码导致的。   
查看业务代码是对图片加水印时读取了Font字体。 网上查询资料后发现是SunGraphics2D.drawString()造成了内存泄露。  
方法中调用了T2KFontScaler.initNativeScaler()这个native方法，会在非堆空间分配内存，但没有回收掉  
业务代码示例:
```
Font afont = Font.createFont(Font.PLAIN, new File("arial.ttf"));
BufferedImage bimg = new BufferedImage(100, 200, BufferedImage.TYPE_INT_ARGB);
Graphics2D gs = bimg.createGraphics();
gs.setFont(afont.deriveFont(Font.PLAIN, new AffineTransform(10, 0, 0, 10, 10, 10)));
gs.drawString("This is font load test", 0, 0);//这里造成了内存分配后没有被回收掉
```
在[JDK-7074159 : run out of memory](https://bugs.java.com/bugdatabase/view_bug.do?bug_id=7074159)中报告了此bug，直到2020-04-07在JDK8_8u281中才被修复
{% asset_img bugfix.png 图2 image %}
快速定位此类问题：  
输出jvm中FontScaler对象的个数
```
jmap -histo <pid> | grep FontScaler
```
{% asset_img objectcount.png 图2 image %}
如果FontScaler对象很多，大概率是这个原因导致的
## 解决方案
升级JDK至8u281及之后的版本
## 总结
1. 先查看JVM内存使用情况
2. 使用pmap、smaps查看进程物理内存情况
3. 使用gdb工具dump可疑的内存块
4. 分析内存dump信息，找出与业务代码关联的信息，定位到代码
5. 善用搜索引擎

参考：
* [记一次堆外内存泄漏排查过程](https://www.jianshu.com/p/dd7ad3838105)
* [记一次Font导致JVM堆外内存泄漏分析](https://juejin.cn/post/6844903866618609672)
* [JDK-7074159 : run out of memory](https://bugs.java.com/bugdatabase/view_bug.do?bug_id=7074159)
* [JDK-8209113 : Use WeakReference for lastFontStrike for created Fonts](https://bugs.java.com/bugdatabase/view_bug.do?bug_id=8209113)