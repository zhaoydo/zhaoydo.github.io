---
title: 四种分布式事务解决方案
date: 2021-04-11 15:40:00
tags:
---

## 四种事务模式
### AT
个人理解：一阶段各个分支提交本地事务、如果有任一分支事务一阶段失败，则二阶段各分支回滚数据，否则标记AT事务成功（二阶段成功）。 通过本地数据库事务配合实现，中间要考虑脏读、脏写的问题  

<!--more-->

#### 前提  
支持本地ACID事务的关系型数据库  
Java应用，通过JDBC访问数据库  
#### 整体机制
两阶段提交的演变：  
1. 一阶段： 业务数据和回滚日志在一个本地事务中提交（提交前先获得一个全局锁），然后释放本地锁和连接资源
2. 二阶段：
    * 提交异步化非常快速的完成（就是删除undolog）
    * 回滚通过一阶段的undolog完成
    * 并释放全局锁

理解两个概念：  
**全局事务**： 包括所有分支事务（分支事务包括一阶段和二阶段）  
**全局锁**： 是一个分支事务内要修改数据的锁，例如update一条记录，根据条件拿到更新数据的id，以表+id作为全局锁，防止分支事务二阶段回滚时，另外一个全局事务修改了数据导致脏写问题

#### 执行流程
以分支事务执行：update product set name = 'GTS' where name = 'TXC'; 为例
##### 一阶段
1. 解析sql得到类型：update、表：product、条件：name='TXC'
2. 根据条件查询到预期被修改的数据，作为beforeImage（假设id=1）
3. 执行sql，更新name
4. 根据beforeImage中的id，查询更新后的记录afterImage
5. 将业务sql、beforeImage、afterImage组成一条回滚记录，插入到undolog中
6. 提交本地事务前，向TC注册分支，获取produc表中，id=1的全局锁
7. 本地事务提交（业务数据以及undolog）
8. 将事务提交结果上报给TC
  如果TC分支事务一阶段都成功，则执行二阶段提交。 有失败的，二阶段回滚    
##### 二阶段-提交
9. 收到TC的提交请求，放入异步队列执行、并立即响应TC
10. 异步队列删除undolog。释放全局锁是不是异步的？没找到文档说明  
##### 二阶段-回滚
9. 收到TC的分支回滚请求，开启本地事务
10. 通过XID和BranchId定位到undolog
11. 数据校验：拿undolog中afterImage数据与当前数据比较
    * 如果一致，根据beforeImage与业务sql生成回滚sql
    * 如果不一致，根据配置的配置策略来处理
12. 提交本地事务，将本地事务回滚结果上报给TC。释放分支事务的全局锁  

#### 脏写的处理
全局锁实现，全局事务会竞争全局锁，不会出现脏读问题。  
对于全局事务和其他事务修改了一条数据，根据具体的配置策略处理：
#### 脏读处理
Seata AT模式默认全局数据隔离级别为“读未提交”（及时本地数据库是“读已提交”or更高），不考虑脏读问题。  

如果需要实现全局“读已提交”，seata通过预先代理执行“SELECT FOR UPDATE ”加锁

### TCC
相对于AT模式，TCC不需要本地数据库事务的支持，自己实现prepare、commit、rollback的逻辑，很容易理解。  
* 一阶段prepare：调用自己的prepare逻辑，做一些资源预留，保证commit能成功执行
* 二阶段commit：调用自己的commit逻辑，执行实际的业务操作
* 二阶段rollback：调用自己的rollback逻辑，补偿或释放资源
![image](http://seata.io/img/seata_tcc-1.png)  
TM(事务管理器):用来控制整个分布式事务的管理，发起全局事务的Begin/Commit/Rollback  
RM(资源管理器):用来注册自己的分支事务，接受TC的Commit或者Rollback请求
#### 执行流程
以“扣钱”场景为例，从账户中扣30元  
1. 一阶段prepare：检查余额是否充足，冻结30元，这一步来保证commit能成功
2. 二阶段commit：扣除一阶段锁定的30元
3. 二阶段rollback：释放prepare扣除的30元  
#### TCC设计-允许空回滚
空回滚：prepare未执行、rollback执行了。 一般是prepare接口丢包没有收到请求，事务触发了回滚，调用rollback方法。  
解决方法： rollback方法执行时发现拿不到事务xid，至返回回滚成功
#### TCC设计-防悬挂
悬挂：rollback比try先执行。 一般是prepare执行超时，事务管理器回滚，执行rollback后，prepare也执行成功了。 这时候rollback时认为事务未发生执行空回滚，然后prepare也执行了  
解决方法:rollback空回滚时也生成事务xid，prepare执行前判断xid存在则不执行逻辑  
#### TCC设计-幂等
幂等：网络拥堵或超时时，事务管理器重试操作，要保持幂等性
解决方案：通过xid和业务主键来判断，例如prepare前判断xid不为空

### Saga
与TCC模式类似，业务方自己实现正向操作、补偿操作。因为没有prepare来锁定资源，会出现脏数据问题。 
![Saga](https://img.alicdn.com/tfs/TB1Y2kuw7T2gK0jSZFkXXcIQFXa-445-444.png)
适用场景：  
1. 业务流程长
2. 包含其他公司项目、遗留项目、其他语言开发的项目，无法提供TCC的三个接口  
### XA
XA是X/Open DTP组织（X/Open DTP group）定义的两阶段提交协议，XA被许多数据库（如Oracle、DB2、SQL Server、MySQL）和中间件等工具(如CICS 和 Tuxedo)本地支持。
X/Open DTP模型（1994）包括应用程序（AP）、事务管理器（TM）、资源管理器（RM）。  
XA接口函数由数据库厂商提供。XA规范的基础是两阶段提交协议2PC。  

XA方式实现分布式事务，其实就是根据由数据库自身提供的XA强一致性方案来实现。
#### 前提
* 支持XA事务数据库
* Java应用，通过JDBC访问数据库
#### 整体结构
![XA](https://img.alicdn.com/tfs/TB1hSpccIVl614jSZKPXXaGjpXa-1330-924.png)
执行阶段：  
    * 可回滚：业务SQL操作在XA分支中进行，由XA协议保证可回滚  
    * 持久化：XA分支完成后，执行XAprepare，由XA协议保证持久化
完成阶段：  
    * 分支提交：执行XA分支commit
    * 分支回滚：执行XA分支的rollback

## 几种分布式事务模式对比
### AT
优点：对业务代码无侵入，学习使用成本低  
缺点：性能不高  
适用于不希望对业务代码进行改造的场景
### TCC
优点：自己实现prepare、commit、rollback不需要本地数据事务支持，性能高  
缺点: 代码侵入性强  
适用于核心系统对性能要求高的地方
### Saga
优点：不需要像TCC一样prepare中锁资源、无锁、实现简单  
缺点：会出现脏数据  
适用于长流程下保证最终一致性，对接其他公司项目或者遗留项目等不能改造成TCC的场景
### XA
优点：基于数据库体用的XA协议实现，简单，数据强一致性  
缺点：性能低  
场景：几乎不使用

## 参考
[Seata官方文档](http://seata.io/zh-cn/docs/dev/mode/at-mode.html)  
[分布式事务的4种模式](https://www.jianshu.com/p/75217db81c99)