---
title: "记一次由 MySQL UPDATE 语句导致锁等待后引发的服务炸裂"
author: "axiaoxin"
date: 2022-05-07T20:35:35+08:00
subtitle: ""
image: ""
tags: ["mysql"]
slug: ""
---

## 场景描述

接收到 P99 超时告警，定位到某接口导致，接口是由同事实现的，逻辑较简单，是一个 MySQL 的 INSERT OR UPDATE 逻辑，
一个请求过来，判断某个非主键字段是否存在，不存在则 INSERT 插入，存在则按该字段UPDATE更新其他字段。
超时告警在触发与恢复之间反复触发，平均耗时 6 秒，接口最近无改动，已上线一段时间运行正常。

表结构：


```sql
CREATE TABLE `CAPTION` (
    `id` int(11) NOT NULL AUTO_INCREMENT COMMENT '主键',
    `created_at` int(11) NOT NULL DEFAULT '0' COMMENT '创建时间',
    `updated_at` int(11) NOT NULL DEFAULT '0' COMMENT '更新时间',
    `user_id` int(11) NOT NULL DEFAULT '0' COMMENT '用户ID',
    `role_type` tinyint(4) NOT NULL DEFAULT '0' COMMENT '用户角色类型',
    `caption_no` varchar(255) NOT NULL DEFAULT '' COMMENT '字幕编号',
    `room_id` varchar(255) NOT NULL DEFAULT '' COMMENT '房间ID',
    `nickname` varchar(150) NOT NULL DEFAULT '' COMMENT '用户昵称',
    `head_url` varchar(255) NOT NULL DEFAULT '' COMMENT '用户头像',
    `text` varchar(255) NOT NULL DEFAULT '' COMMENT '字幕文本',
    `caption_created_at` int(11) NOT NULL DEFAULT '0' COMMENT '字幕实际创建时间',
    PRIMARY KEY (`id`)
) ENGINE=InnoDB;
```



更新语句：

```sql
UPDATE `CAPTION` SET `text`='アプリ別。',`updated_at`=1651759790 WHERE caption_no='c9060d5-5019-4afb-942b-df784a92311b'
```

随后而来的是大面积的接口超时告警，因为最近总是出现带宽告警，确认带宽正常。

这里其实方向有误，如果是带宽问题，应该是所有服务都会告警，而不单单只是这个业务才有告警。

查看接口内相关外部依赖接口调用，也是正常。

查看监控，goroutine 数量暴增，内存也增长，大量接口请求耗时慢到离谱，请求量有增加但也在正常范围内，服务基本处于不可用状态。

![goroutine 数量暴增](https://user-images.githubusercontent.com/2876405/167233640-4d66d248-1547-4db7-8de1-f7204300fb16.png)

![内存也增长](https://user-images.githubusercontent.com/2876405/167233670-6c02ec56-17c3-415c-a9c5-db7cce6e5f6e.png)

![大量接口请求耗时慢到离谱](https://user-images.githubusercontent.com/2876405/167112541-9218bbb9-0ce6-44ab-9a73-d197932e43c4.png)

在这里浪费了大量时间排查问题，也没找到原因，目测这种现象重启服务应该是能恢复正常的，
但由于没有确定问题在哪里纠结要不要现在就重启的时候，又收到了一条关键的云数据库运行线程数超过阈值的告警，这才大概找到了问题排查的方向。

这里的操作也是有问题的，应该在初步排查后已经没有进一步的思路时就应该第一时间尝试重启服务。但是如果第一时间重启服务大概率后续无法触发数据库告警，可能更难定位到原因。

在排查过程中，没有与上面提到的那个接口告警联系起来，接口逻辑只涉及数据库，如果详细排查错误日志，是能发现有打印错误日志的：

```text
Error 1205: Lock wait timeout exceeded; try restarting transaction
```

而这里的错误刚好又没有加sentry捕获错误，日志被淹没不易发现，如果加了基本上可以立刻定位问题。

查看 MySQL 异常诊断提示中的诊断项，有死锁、等待行锁等错误：

![诊断提示](https://user-images.githubusercontent.com/2876405/167129786-319d0556-8390-48ac-a371-c8a91c5383cb.png)

查看致命死锁，现场描述中的 SQL 语句不是这个接口产生的，同 DBA 确认是历史一直存在；再看严重等级的等待行锁，里面存在大量 `updating` 状态的 sql，确认这里的 sql 就是这个接口执行的。

![等待行锁](https://user-images.githubusercontent.com/2876405/167130557-4174dee5-b29f-4c32-a5e1-bf6518402a23.png)

在 MySQL 性能趋势监控，全表扫描数和InnoDB等待行锁次数在这个时间段内也看到了明显增多，其余指标没有明显的异常。

![性能趋势监控](https://user-images.githubusercontent.com/2876405/167128275-99209841-6ad3-4141-b1be-09d8c2279b69.png)

确认后立即重启服务，重启后各监控指标恢复正常，告警恢复。

被更新的表当前数据量 46 万条记录，DBA 提示需要对 WHERE 条件的字段添加索引，更新是全表锁，添加后目前观测正常。

## 原因分析

故障解决后，脑中充满了疑问，久久无法入睡，以下主要整理记录2个问题学习和分析。

### 1. 为何UPDATE会导致锁等待

在 InnoDB 事务中，对记录加锁带基本单位是 next-key 锁（记录锁 + 间隙锁），UPDATE 语句的 WHERE 条件字段没有使用索引，就会进行全表扫描，于是就会对所有记录加上 next-key 锁，相当于把整个表锁住了。


Mysql造成锁的情况有很多，下面我们就列举一些情况：

- 执行DML操作没有commit，再执行删除操作就会锁表。
- 在同一事务内先后对同一条数据进行插入和更新操作。
- 表索引设计不当，导致数据库出现死锁。
- 长事务，阻塞DDL，继而阻塞所有同表的后续操作。

我们这里的场景刚好就是这样，字段没有索引，因此 UPDATE 会使用“表锁”进行更新，再加上大约 30TPS 的并发下，
堆积了大量的 updating 状态的更新语句在排队等待，而此时的记录数有 46 万条，
没有索引检索整个表的锁表更新时间会很长，超过 50s。

因此有以下错误日志出现。

```
Error 1205: Lock wait timeout exceeded; try restarting transaction
```


**Lock wait timeout exceeded 与 Dead Lock 是不一样的**。

`Lock wait timeout exceeded`：后提交的事务等待前面处理的事务释放锁，但是在等待的时候超过了mysql的锁等待时间，就会引发这个异常。

`Dead Lock`：两个事务互相等待对方释放相同资源的锁，从而造成的死循环，就会引发这个异常。

**innodb_lock_wait_timeout 与l ock_wait_timeout 也是不一样的**。

`innodb_lock_wait_timeout`：innodb的dml操作的行级锁的等待时间

`lock_wait_timeout`：数据结构ddl操作的锁的等待时间

查看innodb_lock_wait_timeout的具体值（默认50s）：

```sql
SHOW VARIABLES LIKE 'innodb_lock_wait_timeout'
```


**解决方案：**

由以上知识点的学习，这里解决方式就是为WHERE条件使用的字段添加索引并确认索引生效。

如果 where 条件带上了索引列，但是优化器最终扫描选择的是全表，而不是索引的话，可以使用 `force index([index_name])` 可以告诉优化器使用哪个索引，以此避免有几率锁全表带来的隐患。

也可以将 MySQL 里的 `sql_safe_updates` 参数设置为 1，开启安全更新模式。
当 sql_safe_updates 设置为 1 时。UPDATE 和 DELETE 语句必须满足如下条件之一才能执行成功：

- 使用 WHERE，并且 WHERE 条件中必须有索引列；
- 使用 LIMIT；
- 同时使用 WHERE 和 LIMIT，此时 WHERE 条件中可以没有索引列；

> 参考文章:
>
> [mysql的update更新及delete删表记录where不带索引字段导致死锁](https://blog.51cto.com/lookingdream/4811416)
>
> [MySQL事务锁等待超时 Lock wait timeout exceeded; try restarting transaction](https://juejin.cn/post/6844904078749728782)

### 2. 单个表的锁等待为何会影响整体服务

就算是表被锁了，按道理来说受影响的也只是涉及到这个表的接口会超时，但其他纷纷超时的接口和这个表毫无关联，这个说不通呀。

因为监控指标上有看到内存有飙升，这个解释是由于UPDATE阻塞导致goroutine堆积从而使内存占用增加，难道是因此pod高负载了导致所有响应变慢？

但是细看究竟飙升了多少内存时发现不过600M，这个完全无压力才对，这个假设无法说服自己，那问题还可能出现在哪里呢？因为和数据库有关，那应该只可能是服务配置的数据库最大连接数用完了，全部连接都被排队等待的updating们占用着，其余接口需要查库时，拿不到连接，只能等，所以集体超时。

查看数据库最大连接数和最大空闲连接数都配置的是10，而MySQL一次诊断中的updating都有16个，因此确认猜想是正确的。

## 总结

MySQL UPDATE 语句的 WHERE 条件使用的字段如果没有索引则会进行全表扫描，会对每条记录加锁，相当于锁表。

服务发生异常如果能确定重启能恢复则第一时间重启，因为有 2 个集群每个集群 2 个 pod，可以先重启其中 3 个 pod，保留一个定位问题。
