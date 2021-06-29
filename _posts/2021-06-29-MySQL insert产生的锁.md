---
layout: post
title:  "MySQL insert锁"
subtitle: "介绍insert几种情况导致的锁"
date:   2021-06-29 22:31:29 +0900
categories: MySQL
author:  "张鑫"
tags:
  - insert锁
---

# insert...select

### 插入不同的表
实验表:
```sql
CREATE TABLE `t` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `c` int(11) DEFAULT NULL,
  `d` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `c` (`c`)
) ENGINE=InnoDB;

insert into t values(null, 1,1);
insert into t values(null, 2,2);
insert into t values(null, 3,3);
insert into t values(null, 4,4);

create table t2 like t
```
可重复读隔离级别下,binlog_format=statement 时执行:
```sql
insert into t2(c,d) select c,d from t;
```
需要对表 t 的所有行和间隙加锁,因为如果不加锁,此时另外一个session对表t做操作,就会导致binlog记录两个session的记录顺序不同,语句应用到备库产生数据不一致。

### insert 循环写入
下面SQL,要往表 t2 中插入一行数据,这一行的 c 值是表 t 中 c 值的最大值加 1:
```
insert into t2(c,d)  (select c+1, d from t force index(c) order by c desc limit 1);
```
加锁范围是表 t 索引 c 上的 (3,4]和 (4,supremum]这两个 next-key lock,以及主键索引上 id=4 这一行。扫描行数为1。

如下SQL:
```
insert into t(c,d)  (select c+1, d from t force index(c) order by c desc limit 1);
```
扫描行数为5,这里是用到内存临时表,因为这类一边遍历数据,一边更新数据的情况,如果读出来的数据直接写回原表,就可能在遍历过程中,读到刚刚插入的记录,新插入的记录如果参与计算逻辑,就跟语义不符。

优化方式可以手动创建内存临时表:
```sql
create temporary table temp_t(c int,d int) engine=memory;
insert into temp_t  (select c+1, d from t force index(c) order by c desc limit 1);
insert into t select * from temp_t;
drop table temp_t;
```

### insert唯一键冲突
**锁冲突**
![](/myblog/img/insert_unique_block.jpg)
* 在可重复读(repeatable read)隔离级别下执行的。session B 要执行的 insert 语句进入了锁等待状态。
* session A 执行的 insert 语句,发生唯一键冲突的时候,并不只是简单地报错返回,还在冲突的索引上加了锁。我们前面说过,一个 next-key lock 就是由它右边界的值定义的。这时候,session A 持有索引 c 上的 (5,10]共享 next-key lock(读锁)。

**死锁**
![](/myblog/img/insert_deadlock.jpg)
 session A 执行 rollback 语句回滚的时候,session C 几乎同时发现死锁并返回
 执行逻辑如下:
* 在 T1 时刻,启动 session A,并执行 insert 语句,此时在索引 c 的 c=5 上加了记录锁。这个索引是唯一索引,因此退化为记录锁。
* 在 T2 时刻,session B 要执行相同的 insert 语句,发现了唯一键冲突,加上读锁;同样地,session C 也在索引 c 上,c=5 这一个记录上,加了读锁。
* T3 时刻,session A 回滚。这时候,session B 和 session C 都试图继续执行插入操作,都要加上写锁。两个 session 都要等待对方的行锁,所以就出现了死锁。

### insert into ... on duplicate key update
上面唯一键冲突的语句,改下如下之后:
```sql
insert into t values(11,10,10) on duplicate key update d=100; 
```
给索引 c 上 (5,10] 加一个排他的 next-key lock(写锁)。
insert into ... on duplicate key update 这个语义的逻辑是,插入一行数据,如果碰到唯一键约束,就执行后面的更新语句。




