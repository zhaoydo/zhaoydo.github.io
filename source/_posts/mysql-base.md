---
title: MySQL基础知识总结
date: 2021-04-11 15:36:03
tags:
---

对mysql基础知识的备忘总结，事务、执行计划、存储结构等

<!--more-->

## mysql事务

### ACID
事务的基本要素，满足以下几个条件
1. 原子性（Atomicity） 事务操作要么全部成功，要么失败后回滚到初始状态，不能部分成功。 事务操作像原子一样是不可分隔的一部分
2. 一致性（Consistency）事务开始结束后，数据的一致性没有遭到破坏，例如A向B转账，不能A扣钱了，B没收到
3. 隔离性（Isolation） 多个事务对统一数据操作是隔离的，A事务对数据操作完成之前，B事务不能对这个数据进行操作
4. 持久性（Durability）事务完成后，所有变动都持久化到数据库，不能再被回滚

### 事务并发可能产生的问题
1. 脏读：事务A读取了事务B更新的数据，B回滚，A读到的是脏数据
2. 不可重复读：事务A多次读取数据中，事务B更新了数据并提交事务，导致事务A多次读取到的数据不一致
3. 幻读：事务A修改表中所有学生分数为成绩等级（ABCD）时，事务B向表中插入一条新数据并提交，这时后用户看到一条没有成绩等级的记录，像发生幻觉一样（就是在事务B插入还未提交的时候，事务A更新表中数据，导致事务A不能看到事务B插入的记录）  
不可重复读一般是修改时读取，解决方案是锁行，幻读一般是插入删除，解决方案是锁表。

### 事务隔离级别

| 事务隔离级别 | 脏读 | 不可重复读 | 幻读 |
| ------------ | ---- | ---------- | ---- |
| 读未提交     | 是   | 是         | 是   |
| 读已提交     | 否   | 是         | 是   |
| 可重复读     | 否   | 否         | 是   |
| 串行化       | 否   | 否         | 否   |

mysql默认事务隔离级别是：可重复读 
#### 读未提交
事务修改了数据，还未提交时，就会被其他事务读取到结果，如果事务回滚了，会造成脏读
#### 读已提交
事务修改数据并提交事务后，才能被其他事务读到，避免了脏读问题
#### 可重复读
在一个事务中查询结果都是一致的，避免了不可重复读
#### 串行化
完全串行话操作，每次操作都获得表锁，操作不能并行执行

## mysql常见的各种log
### binlog
记录数据库表结构、表数据的变更，不记录查询。因此可以用来复制和恢复数据。
1. mysql主从模式，slave根据master的binlog来同步变更数据
2. mysql数据被破坏时，可以根据binlog恢复数据

### redolog
mysql修改数据时，先把对应页的数据找到，加载到内存，修改后再持久化到磁盘。  
修改时在内存中把数据改了，还没落到磁盘时，数据库挂掉，这样数据就丢失了。redolog就是用来保证服务重启后这种数据恢复的。  
redolog、binlog区别：  
1. redolog记录的是物理日志，即某数据做了什么修改。 binlog是逻辑日志，即原始的sql操作
2. redolog是innodb引擎独有，binlog是mysql server层功能
3. redolog是一个临时日志，有一定空间循环写。binlog是永久日志
4. redlog是事务执行过程中写入，binlog是事务最终提交前写入。时序上 redo log 先 prepare ， 再写 binlog ，最后再把 redo log commit。

数据崩溃恢复规则：
1、redolog是commit的，直接提交
2、redolog是prepare的，通过事务xid找binlog中是否有记录，如果有则提交，如果无则回滚事务

### undolog
undolog两个作用：事务回滚、多版本控制（MVCC）。在事务提交后即删除  
undolog存储的是逻辑日志，与实际执行的sql正好相反，如果insert操作，undolog存一条delete的日志  
数据修改时记录redolog和undolog，如果事务失败要回滚时，用undolog来进行回滚  
MVCC读取时，发现数据版本大于当前版本，从undolog链中读到当前版本的数据快照

## mysql执行计划
mysql执行计划字段
id	select_type	table	partitions	type	possible_keys	key	key_len	ref	rows	filtered	Extra  

### 字段解释
其中最重要的字段为：id、type、key、rows、Extra
#### id
表示查询的序列号，表示查询的执行顺序，多次关联查询时可能会有多个序列号。  
例如三表关联查询时，三个序列号都是1。前天查询时，序列号不一样。  
#### select_type
查询类型，区分普通查询、联合查询、子查询等复杂的查询  
1. SIMPLE 简单的查询，不包含子查询或union
2. PRIMARY 查询中包含了复杂的子部分，则外层的查询标记为PRIMARY
3. SUBQUERY select/where中有子查询
4. DERIVED from后包含的子查询，表示DERIVED（衍生），结果集为临时表
5. UNION union后的select查询
6. UNION RESULT 从union表中汇集结果的操作，每个union都有UNION RESULT

#### type
访问类型，不同的访问类型直接决定了数据查询的速度，对sql优化很重要  
访问速度依次为：system > const > eq_ref > ref > fulltext > ref_or_null > index_merge > unique_subquery > index_subquery > range > index > ALL  
1. system 表只有1行记录
2. const 通过索引访问，直接命中1条记录
3. eq_ref 两表通过唯一索引join，例如a.id=b.id 两表之间只会产生1条关联记录
4. ref 两表通过索引join
5. range 通过索引扫描，得到多条结果
6. index 只查询索引树就能得到结果（覆盖索引），不用回表，比如select id from xx where id=1 
7. ALL 全表扫描，不走索引。。。

#### possible_keys
查询字段涉及到的索引，但不一定被使用
#### key
实际用到的索引
#### key_len
查询中使用的索引长度
#### ref
查询关联了另一张表的字段
#### rows
查询需要扫描(经过索引过滤后)的行数
#### extra
一些重要的额外信息
1. Using filesort  无法利用索引排序（1、排序字段无索引，2、不是联合索引最左）
2. Using temporary  使用了临时表保存结果，比如order by group by
3. Using index 使用了覆盖索引，从索引中能拿到查询结果，不用回表了
4. Using where 使用了where过滤
5. Using join buffer 两表连接时，如果不能利用索引连接，执行器会使用join buffer优化连接速度。例如两表无索引字段连接时
6. Impossible WHERE where条件互斥，一定查不到时，例如where id=1 and id=2
7. select tables optimized away 没有group by的情况下，对于MIN/MAX优化，可以提前计算
8. distinct 对distinct操作优化，找到第一个就停止