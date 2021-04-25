---
title: G1垃圾收集器
date: 2021-04-25 14:37:00
tags:
---

G1是一款面向服务端应用的垃圾收集器，HotSpot开发团队设计用来取代CMS收集器，特点是追求低并且可控(预测)的停顿时间、在此基础上尽可能提高系统吞吐量。  

<!--more-->

## G1特点

### 并行与并发

G1能充分利用多CPU(核心)的优势，来缩短Stop-The-World时间。部分其他收集器需要停顿Java线程来执行的GC动作，G1可以通过并发的方式来让Java线程继续执行。

### 分代收集

与其他收集器一样，G1仍然会用不同方式对新创建对象和存活一定时间、经历多次GC的旧对象进行收集。但是新生代、老年代不再是物理隔离的了，都是一部分Region的集合。

### 空间整合



### 可预测的停顿

## G1堆内存结构

![G1堆结构](https://www.oracle.com/webfolder/technetwork/tutorials/obe/java/G1GettingStarted/images/slide9.png)

G1将整个堆内存划分成一个个相等内存的Region（区域），每一块都会充当Eden、Surivivor、Old三种角色。G1会跟踪各个Region中的垃圾堆积价值的大小（回收所获得的空间以及回收所需时间得出的一个值），在后台维护一个优先列表，每次根据允许的收集时间，优先回收价值最大的Region（这也是Garbage-First名字的含义），保证G1收集器在限定时间内获取尽可能高的收集效率。  

**GC停顿时间**  

G1中用-XX:MaxGCPauseMillis=200来指定期望的停顿时间，G1每次只是回收堆内部分区块，根据期望停顿时间，和之前垃圾收集的数据统计来估算出用户在停顿时间内能回收多少个区块。

**G1内存占用**  

G1会比CMS有更多的内存消耗，多了两个内存结构：  

1. Remembered Sets：每个区块Region都有一个与之对应的Remembered Set来记录不同区块间对象引用（其他收集器只是记录新生代、老年代间互相引用）。用来避免全堆扫描
2. Collection Sets：将要被回收区块对象的集合，GC时收集器将这些区块对象复制到其他区块中

## G1工作流程



### Young GC

![young GC](https://www.oracle.com/webfolder/technetwork/tutorials/obe/java/G1GettingStarted/images/slide11.png)

Young GC时：  

1. 将年轻代中存活的对象复制到Survivor或者晋升到Old区
2. Young GC会Stop-The-World，GC时多线程并行执行
3. GC后会调整Eden、Survivor区大小

GC后内存结构如下图：  

![young GC](https://www.oracle.com/webfolder/technetwork/tutorials/obe/java/G1GettingStarted/images/slide12.png)

### Mixed GC

当越来越多的对象晋升到Old Region中时，虚拟机会触发一次Mixed GC，回收整个Young Region和部分Old Region。  

1. 初始标记：STW，标记GC Root直接关联的对象
2. 并发标记：找出初始标记找到的对象衍生出来的存活对象，耗时长，与用户线程并行执行
3. 最终标记：标记上一步并发标记时用户线程运行导致标记变化的对象。STW、多线程标记
4. 筛选回收

### Full GC

如果对象内存增长过快，Mixed GC时来不及回收，导致老年代被填满，会触发一次Full GC。 Full GC算法就是单线程执行的serial old gc，会导致比较长的STW时间。   所以要避免Full GC

导致Full GC三种原因：  

1. 并发标记期间，老年代被填满，G1会放弃标记
2. 垃圾回收时年轻代晋升速度大于老年代回收速度，导致老年代被填满
3. 年轻代垃圾收集时，Survivor和Old区没有足够的空间容纳所有存活对象

## G1调优

调优的目标是在限定的期望停顿时间内避免触发Full GC

* 增加堆大小，调整年轻代、老年代比例
* 增加并发周期线程数量，加速并发周期执行
* 让并发周期尽早开始，设置堆使用比例（例如45%），这样避免堆被占满导致Full GC
* Mixed GC回收时回收更多的老年代区块

一些参数：  

* **-XX:+UseG1GC** 使用G1收集器
* **-XX:MaxGCPauseMillis=200** 期望停顿时间
* **-XX:InitiatingHeapOccupancyPercent=45** 堆使用比例45%，达到阈值触发Mixed GC
* **-XX:SurvivorRatio=n** 新生代Eden、Survivor区比例
* **-XX:ConcGCThreads=n** 并发标记线程数  (ParallelGCThreads + 2) / 4^3
* **-XX:ParallelGCThreads=n** 并行收集线程数 cpu<=8?cpu:5*cpu/8+3

