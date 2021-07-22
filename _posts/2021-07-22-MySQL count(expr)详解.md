---
layout: post
title:  "MySQL count(expr)详解"
subtitle: "MySQL select count(expr)介绍"
date:   2021-07-22 18:31:29 +0900
categories: MySQL
author:  "张鑫"
tags:
  - count(1)
  - count(*)
  - count(字段）
---

# MySQL count()介绍

### count()几种用法

count(*)、count(1)、count(字段)

### count(字段)和count(*)的查询结果有什么不同

* COUNT(字段)的统计结果中，如果行中该字段为NULL，会直接跳过
* COUNT(*)的统计结果中，会加上值为NULL的行数
* COUNT(字段)是进行全表扫描，然后判断指定字段的值是不是为NULL，不为NULL则累加

### count(1)和count(*)有什么区别，性能如何

count(1)和count(\*)在MySQL中效果是一样的，count(\*)是SQL92定义的标准统计行数的语法

**官方说明：**
> InnoDB handles SELECT COUNT(*) and SELECT COUNT(1) operations in the same way. There is no performance difference.

### count(\*)的优化

* MyISAM的锁是表级锁，所以同一张表上面的操作需要串行进行，所以，MyISAM做了一个简单的优化，那就是它可以把表的总行数单独记录下来，如果从一张表中使用COUNT(\*)进行查询的时候,直接返回这个记录下来的值，前提是不能有where条件；
* 在InnoDB中，使用COUNT(\*)查询行数的时候，不可避免的要进行扫表了。InnoDB中索引分为聚簇索引（主键索引）和非聚簇索引（非主键索引），聚簇索引的叶子节点中保存的是整行记录，而非聚簇索引的叶子节点中保存的是该行记录的主键的值，相比之下，非聚簇索引要比聚簇索引小很多，所以MySQL会优先选择最小的非聚簇索引来扫表。所以，当我们建表的时候，除了主键索引以外，创建一个非主键索引还是有必要的。