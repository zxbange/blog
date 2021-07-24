---
layout: post
title:  "Cardinality值"
subtitle: "InnoDB存储引擎的Cardinality统计"
date:   2021-07-24 14:31:29 +0900
categories: MySQL
author:  "张鑫"
tags:
  - Cardinality
---

# InnoDB存储引擎Cardinality统计

对于一张表添加索引时,需要考虑Cardinality值,表示索引中不重复记录数量的预估值,通过SHOW INDEX FROM TABL可查看该值。在实际应用中,Cardinality/n_rows_in_table的值应尽可能接近1。如果非常小,就需要考虑这个索引创建的必要性。

InnoDB存储引擎内部对更新Cardinality信息的策略为:
* 上次统计完成后,表中1/16的数据已发生过变化。
* stat_nodified_counter > 2 000 000 000。

对于某一行数据频繁更新操作,表数据实际并没有增加,则第一种更新策略无法使用,所以InnoDB存储引擎内部有一个计数器stat_modified_counter,表示变化次数,大于2 000 000 000,则更新Cardinality值。

统计规则,通过采样的方式,默认对8个叶子节点(Leaf Page)进行采用。

1. 取得B+树索引中叶子节点的数量,记为A。
2. 随机取得B+树索引中的8个叶子节点。统计每个页不同记录的个数,即为P1,P2,··· P8。
3. 根据采样信息给出Cardinality的预估值:Cardinality=(P1+P2+···P8)*A/8

这将导致每次得到的值可能是不同的,如使用show index from table会触发MySQL数据库对于Cardinality值的统计。所以可能导致两次统计结果不同。

参数对Cardinality统计进行设置:

| 参数 | 说明 |
| -------- | -------- |
| Innodb_stats_persistent    | 是否将命令ANALYZE TABLE计算得到的Cardinality值存放到磁盘。若是,可减少重新计算每个索引的Cardinality值,如数据库重启。此外,用户可以通过命令CREATE TABLE和ALTER TABLE的选项STATS_PERSISTENT来对每张表进行控制。默认值:OFF|
| Innodb_stats_on_metadata    | 当通过命令SHOW TABLE STATUS、SHOW INDEX及访问INFORMATION_SCHEMA架构下的表TABLES和STATISTICS时,是否需要重新计算索引的Cardinality值。默认值:OFF|
| Innodb_stats_persisteng_sample_pages    | 若innodb_stats_persistent设置为ON,该参数ANALYZE TABLE更新Cardinality值时每次采样页的数量。默认值:20     |
| Innodb_stats_transient_sample_pages    | 该参数表示每次采样页的数量。默认值:8     |



