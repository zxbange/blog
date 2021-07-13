---
layout: post
title:  "MySQL Innodb锁"
subtitle: "MySQL Innodb锁的介绍"
date:   2021-07-06 22:31:29 +0900
categories: MySQL
author:  "张鑫"
tags:
  - Innodb锁
---

# MySQL Innodb锁

### lock与latch
lock与latch都可以被称为“锁”,但含义不同:
* latch一般成为闩锁,分为mutex(互斥量)和rwlock(读写锁),目的是保证并发线程操作临界资源的正确性,并且通常没有死锁检测;
* lock的对象是事务,用来锁定的是数据库中的对象,如表、页、行。并且一般lock的对象仅在事务commit或rollback后进行释放(不同事务隔离级别释放的时间可能不同),同时有死锁检测机制。

对于lock,用户可以通过命令SHOW ENGINE INNODB STATUS及information_schema架构下的表INNODB_TRX、INNODB_LOCKS、INNODB_LOCK_WAITS来观察锁。

### Innodb锁类型
两种标准行级锁:
* 共享锁(S lock),允许事务读一行数据。
* 排他锁(X lock),允许事务删除或更新一行数据。

|  | X | S |
| ----| ---- | --- |
| X  | 不兼容 | 不兼容 |
| S  | 不兼容 | 兼容  |

Innodb支持多粒度锁定,这种锁定允许事务在行级上的锁和表级上的锁。为了支持在不同粒度上进行加锁操作,Innodb支持一种额外锁方式,**意向锁(Intention Lock)**。

**数据库层次结构**:
数据库A->表->页->行
如果需要对页上的记录r上X锁,那么分别需要对数据库A、表、页上意向锁IX,最后对记录r上X锁。若任何一个部分导致等待,那么该操作需要等待粗粒度锁的完成。
例如在对记录r加X锁前,已经有事务对表1进行了S表锁,那么表1上已经存在了S锁,之后事务需要对记录r在表1上加IX,由于不兼容,该事务需要等待表锁操作完成。

Innodb存储引擎支持意向锁设计比较简单,其意向锁即为表级别的锁。设计目的主要是为了在一个事务中揭示下一行将被请求的锁类型。支持两种意向锁:
* 意向共享锁(IS Lock),事务想要获取一张表中某几行的共享锁
* 意向排他锁(IX Lock),事务想要获取一张表中某几行的排他锁

因为Innodb支持的是行级锁,所以意向锁不会阻塞全表扫以外的任何请求。

|  | IS | IX | S | X |
| -------- | -------- | -------- | --------| -------- |
| IS   | 兼容    | 兼容   | 兼容   | 不兼容   |
| IX   | 兼容    | 兼容   | 不兼容   | 不兼容   |
| S    | 兼容    | 不兼容  | 兼容   | 不兼容   |
| X    | 不兼容   | 不兼容  | 不兼容   | 不兼容   |

通过表information_schema.INNODB_TRX查看当前事务情况:
```sql
mysql> select * from information_schema.INNODB_TRX\G;
*************************** 1. row ***************************
                    trx_id: 2407444
                 trx_state: RUNNING
               trx_started: 2021-07-06 23:14:01
     trx_requested_lock_id: NULL
          trx_wait_started: NULL
                trx_weight: 4
       trx_mysql_thread_id: 5
                 trx_query: NULL
       trx_operation_state: NULL
         trx_tables_in_use: 0
         trx_tables_locked: 2
          trx_lock_structs: 4
     trx_lock_memory_bytes: 1136
           trx_rows_locked: 2
         trx_rows_modified: 0
   trx_concurrency_tickets: 0
       trx_isolation_level: REPEATABLE READ
         trx_unique_checks: 1
    trx_foreign_key_checks: 1
trx_last_foreign_key_error: NULL
 trx_adaptive_hash_latched: 0
 trx_adaptive_hash_timeout: 0
          trx_is_read_only: 0
trx_autocommit_non_locking: 0
*************************** 2. row ***************************
                    trx_id: 421479419620976
                 trx_state: LOCK WAIT
               trx_started: 2021-07-06 23:14:37
     trx_requested_lock_id: 421479419620976:191:5:11
          trx_wait_started: 2021-07-06 23:14:37
                trx_weight: 2
       trx_mysql_thread_id: 4
                 trx_query: select * from t1 where id=10 lock in share mode
       trx_operation_state: starting index read
         trx_tables_in_use: 1
         trx_tables_locked: 1
          trx_lock_structs: 2
     trx_lock_memory_bytes: 1136
           trx_rows_locked: 1
         trx_rows_modified: 0
   trx_concurrency_tickets: 0
       trx_isolation_level: REPEATABLE READ
         trx_unique_checks: 1
    trx_foreign_key_checks: 1
trx_last_foreign_key_error: NULL
 trx_adaptive_hash_latched: 0
 trx_adaptive_hash_timeout: 0
          trx_is_read_only: 0
trx_autocommit_non_locking: 0
2 rows in set (0.00 sec)
```

通过information_schema.INNODB_LOCKS表查看锁情况:
```sql
mysql> select * from information_schema.INNODB_LOCKS\G;
*************************** 1. row ***************************
    lock_id: 421479419620976:191:5:11
lock_trx_id: 421479419620976
  lock_mode: S
  lock_type: RECORD
 lock_table: `zx`.`t1`
 lock_index: PRIMARY
 lock_space: 191
  lock_page: 5
   lock_rec: 11
  lock_data: 10
*************************** 2. row ***************************
    lock_id: 2407444:191:5:11
lock_trx_id: 2407444
  lock_mode: X
  lock_type: RECORD
 lock_table: `zx`.`t1`
 lock_index: PRIMARY
 lock_space: 191
  lock_page: 5
   lock_rec: 11
  lock_data: 10
2 rows in set, 1 warning (0.00 sec)
```
通过表information_schema.INNODB_LOCK_WAITS可直观查看当前事务的等待:
```sql
mysql> select * from information_schema.INNODB_LOCK_WAITS\G;
*************************** 1. row ***************************
requesting_trx_id: 421479419620976
requested_lock_id: 421479419620976:191:5:11
  blocking_trx_id: 2407444
 blocking_lock_id: 2407444:191:5:11
1 row in set, 1 warning (0.00 sec)
```

### 一致性非锁定读
InnoDB存储引擎通过行多版本控制的方式读取当前执行时间数据库中行的数据。如果正在读取的行在做delete或者update操作,Innodb存储引擎会去读取行的一个快照数据。
* RC隔离级别,对于快照数据,总是读取被锁定行的最新一份快照数据;
* RR隔离级别,对于快站数据,总是读取事务开始时的行数据版本。

### 一致性锁定读
这种方式不会读取行数据的快照,而是直接给行去加锁。
* select ... for update对读取的行记录加一个X锁,其他事务不能对已锁定的行加任何锁。
* select ... lock in share mode对读取的行记录加一个S锁,其他事务可以向被锁定的行加S锁,但不能加X锁。

### 外键和锁
对于外键值的插入和更新,首先查父表记录,但是不是使用一致性非锁定读,使用select ... lock in share mode方式给父表加一个S锁,保证数据的一致。但是如果父表上加了X锁,就会导致子表操作被阻塞。

## 锁的算法

### 行锁的3种算法
* Record Lock:单个行记录上的锁;
* Gap Lock:间隙锁,锁定一个范围,但不包含记录本身;
* Next-Key Lock:Gap Lock+Record Lock。

Record Lock总是会去锁住索引记录,如果InnoDB存储引擎表在建立的时候没有设置任何一个索引,那么会使用隐式的主键进行锁定。

* 原则 1:加锁的基本单位是 next-key lock。next-key lock 是前开后闭区间。
* 原则 2:查找过程中访问到的对象才会加锁。
* 优化 1:索引上的等值查询,给唯一索引加锁的时候,next-key lock 退化为行锁。
* 优化 2:索引上的等值查询,向右遍历时且最后一个值不满足等值条件的时候,next-key lock 退化为间隙锁。
* 一个 bug:唯一索引上的范围查询会访问到不满足条件的第一个值为止。(目前已修复)

显示关闭Gap Lock:
* 将事务隔离级别设置为READ COMMITTED;
* 将参数innodb_locks_unsafe_for_binlog设置为1。

## 锁的问题
### 脏读
在READ-UNCOMMITTED隔离级别下,会产生脏读,即事务T2对数据的更新并未提交,但是事务T1可以读到。在READ-COMMITTED隔离级别下可避免。
### 幻读
在READ-COMMITTED隔离级别下,会产生欢度,即事务T2对数据的更新已提交,但是事务T1可以读到。在READ-REPEATABLE隔离级别下可避免。
### 丢失更新
* 事务T1将记录r更新为v1,但是事务T1未提交;
* 事务T2将记录r更新为v2,事务T2未提交;
* 事务T1提交;
* 事务T2提交。
串行化可解决这类问题

### 阻塞
innodb_lock_wait_timeout:动态参数,控制等待时间(默认是50秒);
innodb_rollback_on_timeout:静态参数,设定是否等待超时时对进行中的事务回滚(默认OFF不回滚)。
















