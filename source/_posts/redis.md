---
title: Redis介绍
categories:
- redis
date: 2022-12-24 20:00:00
tags:
- redis
---


# 介绍

redis是一个开源（BSD许可证）的内存结构存储，用作数据库、缓存、消息、消息代理和流引擎。提供string、hashes、lists、sets、[sorted sets](https://redis.io/docs/data-types/sorted-sets/) （有序集合）、bitmaps（位图）、hyperloglogs（用于基数统计）、geospatial indexes（地理空间索引）、 streams。 redis内置了复制、Lua脚本、LRU删除、事务和不同级别的磁盘持久化（RDB和AOF），并通过Redis Sentinel实现高可用，通过Redis Cluster实现自动分区。

<!--more-->

# 安装

省略，参考官网https://redis.io/docs/getting-started/

# 数据类型

## String

字符串，最基本的数据类型，包括text、序列化的对象、二进制数组。通常用于缓存，也能实现计数器和按位运算

默认一个字符串最多存储512MB的内容

### 设置和获取对象

```shell
# set一个key
SET user 1
# set一个key，只有key不存在时才能设置成功
SETNX user 1
# get 
GET user
# 批量获取key
MGET user user1
```

###  计数器

```shell
# 设置key自增1，如果key不存在，默认0 然后自增1
INCR key
# 设置key自增指定的数量
INCRBY key 5
# 同上，支持浮点数
INCRBYFLOAT key 0.1

```

### 位运算

参考后续的bitmap

## List

字符串值的链表结构，最大长度是 2^32 - 1。用法有

1. 实现栈和队列
2. 为后台工作系统构建队列管理

### 实现队列（先进先出）

```shell
# 左侧插入元素1，返回长度。如果list不存在，新建一个
LPUSH work:queue:ids 1
(integer) 1
LPUSH work:queue:ids 2
(integer) 2
# 右侧取出元素
RPOP work:queue:ids
"1"
RPOP work:queue:ids
"2"
```

### 实现栈（先进后出）

RPOP换成LPOP即可

```shell
# 左侧取出元素
LPOP work:queue:ids
```

### 阻塞命令

有点像java中的阻塞队列.如果list为空，则阻塞等待有新元素插入或者超时

```shell
BLPOP work:queue:ids 10
(nil)
(10.00s)
```

## Set

字符串值的无序、唯一集合，最大长度为 2^32 - 1 。 用法有

1. 维护唯一条目的集合（例如追踪一个博客文章的访问ip）
2. 表示关系（例如一个角色的所有用户集合）
3. set常用的操作，交集、并集、差集

```shell
# 添加元素
SADD user:123:favorites 347
(integer) 1
# 判断元素是否存在
SISMEMBER user:123:favorites 347
(integer) 1
SISMEMBER user:123:favorites 456
(integer) 0
# 获取两个集合交集
SINTER user:123:favorites user:456:favorites
1) "347"
2) "123"
# 获取集合元素个数
SCARD user:123:favorites
(integer) 3
```

对于大型的数据集，集合元素检查可能会占用大量内存。 如果考虑内存占用，且不需要完美精度，考虑使用布隆过滤器or布谷过滤器替代。

## Hash

Hash结构是field-value对的集合。 可以使用Hash表示基本对象和存储计数器分组。最大长度为 2^32 - 1 

```shell
# 设hash元素多个field-value
HSET user:123 username martina firstName Martina lastName Elisa country GB
(integer) 4
# 获取某个field
HGET user:123 username
"martina"
#设置key：device:777:stats field:pings 自增1
HINCRBY device:777:stats pings 1
(integer) 1
HINCRBY device:777:stats pings 1
(integer) 2
# 获取元素
HGET device:777:stats pings
"3"
```

## Sorted set

Sorted set是一个有序、唯一的集合，根据集合元素关联的score排序。如果score相同，按照字典顺序排序。

1. 排行榜
2. 限流器，构建滑动窗口实现限流

```shell
# 向集合添加user:1 score：100
ZADD leaderboard:455 100 user:1
(integer) 1
ZADD leaderboard:455 75 user:2
(integer) 1
ZADD leaderboard:455 101 user:3
(integer) 1
# 获取top3的元素
ZRANGE leaderboard:455 0 2 REV WITHSCORES
1) "user:2"
2) "275"
3) "user:3"
4) "101"
5) "user:1"
6) "100"
#获取user:2 的排名
ZREVRANK leaderboard:455 user:2
(integer) 0
```



## Stream

Stream是一个类似只追加日志的数据类型，可以使用Stream实时记录（and  simultaneously syndicate events 没看懂）。 Redis Stream使用场景包括：

1. 事件追踪（跟踪用户行为、点击等）
2. 传感器监测（现场设备的读数）
3. 通知（在一个特定Stream中存储每个用户的通知记录）

redis为Stream中每个entry生成一个唯一ID，可以使用ID检索关联的entry，或者读取和处理Stream中所有后续的entry

Redis Stream支持几种削减策略（防止Stream无线增长），和多种消费策略（XREAD、XREADGROUP、XRANGE）

