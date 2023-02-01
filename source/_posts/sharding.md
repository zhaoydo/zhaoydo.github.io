---
title: 数据库水平分表落地
date: 2022-06-28 22:00:00
tags: 
- 源码
---
> 本文只讨论水平分表，垂直分表属于业务重构问题

# 为什么分表
参考下图，t_user表根据业务字段id拆分，奇数放入table1，偶数放入table2
![水平分片](https://shardingsphere.apache.org/document/current/img/sharding/horizontal_sharding.png)

通过一个或多个字段/规则将数据分散到多个数据库、数据表中，使得业务数据突破单表性能瓶颈。  
<!--more-->
为什么效率高：  
1. 减少单表数据、索引大小，提高读写效率
2. 开多个线程并发读写分片数据（十个线程分别从一千万表中读 VS 一个线程从一亿表中读）
3. 提前缩小读写范围，sql条件中有id（分片列），只需要读写对应分片表即可

带来的问题：  
1. 分表后表结构维护，数据运维
2. 多线程写、跨库写时事务问题
3. 复杂sql，insert into select,分表字段列update、limit、order by、group by某些情况下不能正确执行

> 举个例子 TODO

# 技术选型

|          | ShardingJDBC                       | Mycat                 |
| -------- | ---------------------------------- | --------------------- |
| 分库     | 支持                               | 支持                  |
| 分表     | 支持                               | 支持                  |
| 实现方式 | 解析改写SQL                        | 中间层代理            |
| 性能     | 损耗低                             | 损耗略高              |
| 使用     | 引用jar包                          | 需单独安装管理端      |
| 社区     | star 15.1k Apache-2.0许可          | star 9.2k GPL-2.0许可 |
| 官网     | https://shardingsphere.apache.org/ | http://mycat.org.cn/  |
|          |                                    |                       |
|          |                                    |                       |
|          |                                    |                       |

选择ShardingJDBC， 分表性能损耗低，实现方式容易理解，无需单独安装服务端。 11月刚发布了5.0正式版

> ShardingJDBC4.0后，官方将ShardingJDBC、ShardingProxy统一为ShardingSphere



# ShardingJDBC

## 内置分片算法

| *SPI 名称*        | *详细说明* |
| :---------------- | :--------- |
| ShardingAlgorithm | 分片算法   |

| *已知实现类*                        | *详细说明*                 |
| :---------------------------------- | :------------------------- |
| BoundaryBasedRangeShardingAlgorithm | 基于分片边界的范围分片算法 |
| VolumeBasedRangeShardingAlgorithm   | 基于分片容量的范围分片算法 |
| ComplexInlineShardingAlgorithm      | 基于行表达式的复合分片算法 |
| AutoIntervalShardingAlgorithm       | 基于可变时间范围的分片算法 |
| ClassBasedShardingAlgorithm         | 基于自定义类的分片算法     |
| HintInlineShardingAlgorithm         | 基于行表达式的Hint分片算法 |
| IntervalShardingAlgorithm           | 基于固定时间范围的分片算法 |
| HashModShardingAlgorithm            | 基于哈希取模的分片算法     |
| InlineShardingAlgorithm             | 基于行表达式的分片算法     |
| ModShardingAlgorithm                | 基于取模的分片算法         |

官方提供了几种常用的分片算法，分为三大类

### 精确分片算法

根据分片key，直接计算出数据表，一般用于非范围分片key，buyerId，orderId等

#### InlineShardingAlgorithm

根据表达式t_order_$->{order_id % 4}，计算分片表

```java
inlineShardingAlgorithm = new InlineShardingAlgorithm();
        inlineShardingAlgorithm.getProps().setProperty("algorithm-expression", "t_order_$->{order_id % 4}");
        inlineShardingAlgorithm.getProps().setProperty("allow-range-query-with-inline-sharding", "true");
```



#### ComplexInlineShardingAlgorithm

多个分片key  

t_order表根据type、order列分表，分片表为t_order_0_0、t_order_0_1、t_order_1_0、t_order_1_1

```java
complexInlineShardingAlgorithm = new ComplexInlineShardingAlgorithm();
complexInlineShardingAlgorithm.getProps().setProperty("algorithm-expression", "t_order_${type % 2}_${order_id % 2}");
complexInlineShardingAlgorithm.getProps().setProperty("sharding-columns", "type,order_id");
complexInlineShardingAlgorithm.init();
```

#### HashModShardingAlgorithm

将t_order根据buyer_id(varchar 20) hash后分为四片，  key.hashCode()%4, 

```
shardingAlgorithm = new HashModShardingAlgorithm();
        shardingAlgorithm.getProps().setProperty("sharding-count", "4");
```

#### HintInlineShardingAlgorithm

强制指定要查询的表，不根据SQL解析。例如下面代码查询表为t_order_0%4

```java
hintInlineShardingAlgorithm = new HintInlineShardingAlgorithm();
Properties props = new Properties();
props.setProperty("algorithm-expression", "t_order_$->{value % 4}");
hintInlineShardingAlgorithm.setProps(props);
hintInlineShardingAlgorithm.init();
//手动指定分片
final HintManager hintManager = HintManager.getInstance();
hintManager.addTableShardingValue("t_order", 0);
orderService.get(122343L);
```



### 范围分片算法

根据设置的分片key范围，定位数据表。datetime、id等

#### BoundaryBasedRangeShardingAlgorithm

自定义分片边界算法

-∞~10000  table0  

10000~20000 table1  

20000~30000 table2    

30000~∞ table3  

```java
BoundaryBasedRangeShardingAlgorithm shardingAlgorithm = new           BoundaryBasedRangeShardingAlgorithm();
    shardingAlgorithm.getProps().setProperty("sharding-ranges", "10000,20000,30000");
    shardingAlgorithm.init();
```



### 容量分片算法

其实就是范围分片的变种，分片规则以容量配置，一般用于有序并知道预期范围分片key

#### VolumeBasedRangeShardingAlgorithm

自定义分片容量

分片key预期在0-100万，将此区间内数据均匀分片，每片最多10万数据

```java
VolumeBasedRangeShardingAlgorithm shardingAlgorithm = new VolumeBasedRangeShardingAlgorithm();
shardingAlgorithm.getProps().setProperty("range-lower", "0");
shardingAlgorithm.getProps().setProperty("range-upper", "1000000");
shardingAlgorithm.getProps().setProperty("sharding-volume", "100000");
shardingAlgorithm.init();
```

#### AutoIntervalShardingAlgorithm

根据分片key时间均分  

t_order表 根据order_time（预期20年-21年），每天分一片，分片表为t_order_0，t_order_1...t_order_366

```java
shardingAlgorithm = new AutoIntervalShardingAlgorithm();
        shardingAlgorithm.getProps().setProperty("datetime-lower", "2020-01-01 00:00:00");
        shardingAlgorithm.getProps().setProperty("datetime-upper", "2021-01-01 00:00:16");
        shardingAlgorithm.getProps().setProperty("sharding-seconds", String.valueOf(3600*24));
        shardingAlgorithm.init();
```

#### IntervalShardingAlgorithm

基于时间单位分片  

t_order 根据order_time，每月分一片，分片表为t_order_0,t_order_1，t_order_2...t_order_59

```java
 shardingAlgorithmByMonth = new IntervalShardingAlgorithm();
        shardingAlgorithmByMonth.getProps().setProperty("datetime-pattern", "yyyy-MM-dd HH:mm:ss");
        shardingAlgorithmByMonth.getProps().setProperty("datetime-lower", "2016-01-01 00:00:00");
        shardingAlgorithmByMonth.getProps().setProperty("datetime-upper", "2021-12-31 00:00:00");
        shardingAlgorithmByMonth.getProps().setProperty("sharding-suffix-pattern", "yyyyMM");
        shardingAlgorithmByMonth.getProps().setProperty("datetime-interval-amount", "1");
        shardingAlgorithmByMonth.getProps().setProperty("datetime-interval-unit", "Months");
        shardingAlgorithmByMonth.init();
```

### 自定义算法

自己实现分片规则

#### ClassBasedShardingAlgorithm

自定义类分片算法

最灵活的分片算法，任意实现业务逻辑

```java
shardingAlgorithm = new ClassBasedShardingAlgorithm();
        shardingAlgorithm.getProps().setProperty("strategy", "standard");
        shardingAlgorithm.getProps().setProperty("algorithmClassName", ClassBasedStandardShardingAlgorithmFixture.class.getName());
        shardingAlgorithm.init();
```

# 使用

## 项目配置

application-dev.properties

```
# 直接看项目配置
```

TradeMybatisConfigurer.java 定义主数据源

## 动态数据源切换

将分表数据源定义为主数据源，对于某些SQL，ShardingJDBC不支持，这种情况需要我们手动切换为普通数据源（druid）

使用Spring提供的AbstractRoutingDataSource实现数据源切换

```java
/**
 * 动态事务
 * @author zhaoyd
 * @date 2021-12-27
 */
public class DynamicDataSource extends AbstractRoutingDataSource {

    private static final InheritableThreadLocal<DataSourceKeyEnum> CONTEXT_HOLDER = new InheritableThreadLocal<>();
    /**
     * 初始化数据源
     * @param defaultTargetDataSource
     * @param targetDataSources
     */
    public DynamicDataSource(DataSource defaultTargetDataSource, Map<Object, Object> targetDataSources) {
        super.setDefaultTargetDataSource(defaultTargetDataSource);
        super.setTargetDataSources(targetDataSources);
        super.afterPropertiesSet();
    }
    @Override
    protected Object determineCurrentLookupKey() {
        return getDataSourceKey();
    }
    public static void setDataSourceKey(DataSourceKeyEnum dataSource) {
        CONTEXT_HOLDER.set(dataSource);
    }

    public static DataSourceKeyEnum getDataSourceKey() {
        return CONTEXT_HOLDER.get();
    }

    public static void clearDataSourceKey() {
        CONTEXT_HOLDER.remove();
    }
}
```

自定义数据源切换注解@EnableDataSource(DataSourceKeyEnum)，通过AOP拦截下注解。很简单

```java
/**
     * 合并所有订单
     */
    @EnableDataSource(DataSourceKeyEnum.TRADE)//表示使用非分片的druid数据源
    @RequestMapping("mergeallorder.html")
    public JsonDataDto mergeAllOrder(PlatOrderMergeSearchDto queryBean, HttpServletRequest request) throws BaseAppException{
        Boolean isIgnoreLogis = BooleanUtils.toBoolean(request.getParameter("isIgnoreLogis"));
        AssertUtils.notNull(isIgnoreLogis);
        List<String> mergeResult = mergeOrderBusiService.mergeAllOrder(queryBean, isIgnoreLogis);
        return new JsonDataDto(mergeResult);
    }
```



# 分布式ID

分表后，无法使用单表自增ID，需要使用分布式ID方案来生成唯一ID  

分布式ID方案有三种

|          | 示例                                                         | 优点                                 | 缺点                               |
| -------- | ------------------------------------------------------------ | ------------------------------------ | ---------------------------------- |
| UUID     | 91c606f6-d87f-4dd1-80b2-414b6ac38ad0                         | 无单点问题，实现最简单               | 无序，32位字符串长，可读性差       |
| 号段模式 | 211790001-211800001（Long）                                  | 效率高，趋势递增，和自增ID基本一致   | 依赖单点生成，实现复杂(需解决并发) |
| 雪花算法 | ![](https://pic4.zhimg.com/80/v2-0ca4a4125b1cbda69cfa972b1e568ffb_720w.jpg) | 无单点问题，效率高，趋势递增有可读性 | 64位太长，需要解决时钟回拨问题     |

> 时钟回拨： 服务器每隔一段时间会从网络时钟服务器校时，如果某次校时，由15:01退回到15:00.这时候ID就会重复。  
>
> 大部分解决时钟回拨的方法是服务将历史一段时间内（例如一小时）每个毫秒的最大自增序号保存起来



项目采用的是号段模式，主要考虑到下面几点  

1. 雪花算法需要对每台应用单独维护一个workerID，无论是配置文件直接配置还是通过redis、zookeeper注册都很麻烦，[看这个](https://blog.csdn.net/u013465194/article/details/83385198)
2. UUID需要修改ID数据类型，而且是无序的。插入数据时会重构索引B+树（回忆下白泽上树老师的mysql分享），占用空间大
3. 号段模式生成ID，在数据格式，长度和原来自增一致，唯一缺点时不能保证严格的自增（两台cps申请号段分别为100-200,200-300，会出现id=250的create_time 小于id=150的情况）

### 号段模式实现

1. 数据库维护一张号段表，每台服务根据biz_tag取一定长度的号段，放在服务内号段缓存中
2. 每次数据插入，先从服务内号段缓存取一个id，如果缓存用完，则从号段表中初再初始化一批号段

| biz_tag            | next_id   | step  | remark         | update_time      |
| ------------------ | --------- | ----- | -------------- | ---------------- |
| prod_sync_p_shopee | 211790001 | 10000 | shopee同步父表 | 2022-1-6 22:26   |
| prod_sync_s_shopee | 750310001 | 10000 | shopee同步子表 | 2022-1-6 23:18   |
| test               | 1         | 100   | test           | 2021-11-24 15:33 |

给实体类加上注解即可，实现细节：SegmentIdGenerator.java

```java
public class ProdSyncPShopee extends ShardingBaseBean {
    @Id
    @KeySql(genId = SegmentIdGenerator.class)
    private Long id;
}
```

## 事务处理

>  思考一个问题，前面说分表时以多线程执行每个分片SQL，事务如何处理

ShardingJDBC默认使用本地事务，支持绝大部分单应用场景下跨库和非跨库事务。[官网说明](https://shardingsphere.apache.org/document/current/cn/features/transaction/use-norms/)

| 场景    | 事务 | 备注                                                         |
| ------- | ---- | ------------------------------------------------------------ |
| 普通sql | 支持 | 单个事务连接commit or rollback                               |
| 分表SQL | 支持 | 多个事务连接commit or rollback，不能处理commit阶段异常(数据库宕机、网络中断) |
| 分库SQL | 支持 | 多个事务连接commit or rollback，不能处理commit阶段异常(数据库宕机、网络中断) |

源码中本地事务的实现如下

> 对prod_sync_p_shopee表根据id更新，测试库分了三张表，开启三条数据库连接并发执行update语句

提交事务，遍历当前事务所有的事务Connection，依次执行commit()

![commit](http://zentao.epean.cn/zentao/file-read-11179.png)

异常时回滚事务，遍历当前事务打开的所有事务Connection，依次执行rollback()回滚数据

![rollback](http://zentao.epean.cn/zentao/file-read-11180.png)

# 遇到的问题

## 提前初始化陷阱

提前初始化导致事务失效

shiroFilter引入了UserService。 导致bean在BeanPostProcessor之前被初始化，AOP基于BeanPostProcessor实现事务导致相关的bean不生效

详细看这个：[引入shiro后userService事务不生效原因](https://www.jianshu.com/p/b1209cd3686d)



## 分表关联笛卡尔积

父子表10*10 

```sql
select * from t_parent as p inner join t_detail as d on d.p_id=p.id
```

如果父子表都有分表，会关联t_parent0-9 * t_detail0_9 执行100条sql语句

解决方案是绑定父子表分片，这样t_parent_0至于t_detail_0 连接查询，执行10条sql语句

```
spring.shardingsphere.rules.sharding.binding-tables[0]=t_parent,t_detail
```

> 这个问题在生产上造成了两周系统卡顿，没仔细阅读官网文档

## 特殊SQL支持



特殊的sql不支持，类似下面这些

insert into select * from xxx;

left join 分表table

非分表table相关sql用动态数据源注解切换为普通数据源。分表table改写sql用其他方式实现



# 总结

上线分表后有两个问题

1. 未做父子表关联绑定，查询笛卡尔积，导致生产系统两周数据库高负载。父子分片绑定后解决
2. 部分sql不支持，报错。 统一排查报错原因一个个改

分表看起来香，落地难。要解决数据源、事务、分布式id、特殊sql处理这些问题，还不太成熟。如果从0设计一个系统，会倾向于分布式数据库。

# 参考资料

[ShardingSphere官网文档](https://shardingsphere.apache.org/document/5.0.0/en/overview/)

[ShardingJDBC分片算法和解析流程](https://www.cnblogs.com/wuer888/p/14531046.html)

[美团分布式ID生成服务](https://tech.meituan.com/2019/03/07/open-source-project-leaf.html)

[大众点评订单系统分库分表实践](https://tech.meituan.com/2016/11/18/dianping-order-db-sharding.html)

[基于AbstractRoutingDataSource实现动态数据源](https://www.cnblogs.com/qlqwjy/p/13423021.html)

[引入shiro后userService事务不生效原因](https://www.jianshu.com/p/b1209cd3686d)