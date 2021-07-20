---
layout: post
title:  "pt-online-schema-change工具"
subtitle: "pt-online-schema-change介绍"
date:   2021-07-20 18:31:29 +0900
categories: MySQL
author:  "张鑫"
tags:
  - pt-online-schema-change
---

### 说明

* 支持在线修改表结构，加锁时间短，对线上业务影响小
* 根据需要修改的源表结构创建一个临时使用的空表，然后在源表创建触发器，接下来开始慢慢地将数据从源表复制到新表中，在复制过程中，如果源表数据发生变化，则通过触发器同步到新表中，当数据完成复制后，调用rename table指令，互换源表和新表名字，换下来的废弃的表默认被删除
* 源表必须要有主键或者唯一索引（唯一索引不能存在空值）

### 详细执行流程

1. 相关环境参数检查，状态信息，检查是否有触发器，检查表是否有主键
2. 接着按照修改表的表定义，新建一个名为'_tb_new'不可见的临时表，对这个表执行alter添加字段，并校验是否执行成功。
3. 然后针对源表创建三个触发器，分别如下：
    * create trigger db_tb_del after delete on db.tb for each row delete ignore from db._tb_new where db._tb_new.id <=> OLD.id #删掉新表中db._tb_new.id <=> OLD.id的数据，否则忽略操作
    * create trigger db_tb_del after update on db.tb for each row replace into db._tb_new(id,...) values(new.id,...)  #源表执行update的时候，把对应的数据replace into的方式写入新表
    * create trigger db_tb_del after insert on db.tb for each row replace into db._tb_new(id,...) values(new.id,...)  #源表执行iinsert操作的时候，把对应的数据replace into的方式写入新表
4. 触发器创建好之后会执行insert low_priority ignore into db._tb_new(id,..) select id,... from tb lock in share mode语句复制源表数据到新表。
5. 复制完成之后执行语句：rename table db.tb to db._tb_old,db._tb_new to db.tb同时把源表修改为_tb_old格式，把新表_tb_new修改为源表名字的原子修改。
6. 接着，如果没有加不删除old表的选项，那么就会删除Old表，然后删除三个触发器。到这里就完成了在线表结构的修改 。整个过程只在rename表的时间会锁一下表，其他时候不锁表。

### 原表支持在线DML

* ALTER过程采用Copy Table To New Table的方式，新建一个表格，然后在原表上创建3个触发器：DELETE\UPDATE\INSERT触发器，拷贝数据到新表的过程中，如果原表数据发生变化，则会通过触发器更新到新表上。
* INSERT原表的时候，触发器根据其主键ID把新纪录INSERT到新表上；
* UPDATE原表的时候，触发器根据其主键ID判断新旧ID是否一致，如果一致则删除，然后在REPLACE INTO新纪录到新表
* DELETE原表的时候，触发器根据其主键ID直接删除行记录
* 如果数据修改的时候，还没有拷贝到新表，修改后再拷贝；如果是数据已经拷贝，原表发生修改，这时触发器同步修改数据，两种情况下都保证了数据的一致性；

### 整个操作流程锁情况

* 创建新表后，按照每一个chunk的大小拷贝数据到新表，每次SELECT都是share mode，带S锁，但是每个chunk都比较小，所以锁时间不大
* 最后数据拷贝结束，会有一个rename操作，这个操作过程中，是不支持DML操作的，但其速度很快，不会造成长时间锁表情况
* 该工具会设置该DDL操作的锁等待超时为1s，当出现异常的时候，会使ALTER操作异常，而不是其他业务操作异常，这样可以最大程度的不影响其他事务的进行

### 执行期间性能影响

* 总体而言，对数据库的锁影响降低到了最小，执行期间允许DML操作
* 但是注意，任何DDL SQL在这里，都是转换成copy table to new table的形式，这个过程中，会极大占用磁盘的IO跟CPU资源，同时跟主从延时带来一定的影响，还是那句老话，重复了解DDL的影响程度后，再选择合适时机执行。
* copy data过程中，如果主从延迟异常超过 max-lag则停止copy data，等待主从延迟恢复，默认为1min，可以通过--max-lag设置
* 检测到服务器负载异常，也会停止操作，可以通过 --max-load，--critical-load设置

### 该工具限制情况

* 表格必须带有主键或者唯一索引
* 存在复制过滤掉表格，ALTER操作
* copy data过程中，如果主从延迟异常超过 max-lag则停止copy data，等待主从延迟恢复，默认为1s，可以通过--max-lag设置
* 检测到服务器负载异常，也会停止操作，可以通过 --max-load，--critical-load设置
* 设置操作的锁等待超时为1s，当出现异常的时候，ALTER操作异常，而不是其他业务操作异常，这样可以最大程度的不影响其他事务的进行
* 默认情况下，存在 被外键引用的表格是不支持ALTER操作的，除非手动指定参数--alter-foreign-keys-method
* 不支持修改 Percona XtraDB Cluster （PXC）上节点的 myisam表格

### 模拟执行

```sql
[root@localhost ~]# pt-online-schema-change --user=root --password=root -P 3306 --charset=utf8 --alter "add column amount bigint(10)" D=employees,t=salaries --print --dry-run
Operation, tries, wait:
  analyze_table, 10, 1
  copy_rows, 10, 0.25
  create_triggers, 10, 1
  drop_triggers, 10, 1
  swap_tables, 10, 1
  update_foreign_keys, 10, 1
Starting a dry run.  `employees`.`salaries` will not be altered.  Specify --execute instead of --dry-run to alter the table.
Creating new table...
CREATE TABLE `employees`.`_salaries_new` (
  `emp_no` int(11) NOT NULL,
  `salary` int(11) NOT NULL,
  `from_date` date NOT NULL,
  `to_date` date NOT NULL,
  PRIMARY KEY (`emp_no`,`from_date`),
    CONSTRAINT `_salaries_ibfk_1` FOREIGN KEY (`emp_no`) REFERENCES `employees` (`emp_no`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
Created new table employees._salaries_new OK.
Altering new table...
ALTER TABLE `employees`.`_salaries_new` add column amount bigint(10)
Altered `employees`.`_salaries_new` OK.
Not creating triggers because this is a dry run.
-----------------------------------------------------------
Skipped trigger creation: 
Event : DELETE 
Name  : pt_osc_employees_salaries_del 
SQL   : CREATE TRIGGER `pt_osc_employees_salaries_del` AFTER DELETE ON `employees`.`salaries` FOR EACH ROW BEGIN DECLARE CONTINUE HANDLER FOR 1146 begin end; DELETE IGNORE FROM `employees`.`_salaries_new` WHERE `employees`.`_salaries_new`.`emp_no` <=> OLD.`emp_no` AND `employees`.`_salaries_new`.`from_date` <=> OLD.`from_date`; END  
Suffix: del 
Time  : AFTER 
-----------------------------------------------------------
-----------------------------------------------------------
Skipped trigger creation: 
Event : UPDATE 
Name  : pt_osc_employees_salaries_upd 
SQL   : CREATE TRIGGER `pt_osc_employees_salaries_upd` AFTER UPDATE ON `employees`.`salaries` FOR EACH ROW BEGIN DECLARE CONTINUE HANDLER FOR 1146 begin end; DELETE IGNORE FROM `employees`.`_salaries_new` WHERE !(OLD.`emp_no` <=> NEW.`emp_no` AND OLD.`from_date` <=> NEW.`from_date`) AND `employees`.`_salaries_new`.`emp_no` <=> OLD.`emp_no` AND `employees`.`_salaries_new`.`from_date` <=> OLD.`from_date`; REPLACE INTO `employees`.`_salaries_new` (`emp_no`, `salary`, `from_date`, `to_date`) VALUES (NEW.`emp_no`, NEW.`salary`, NEW.`from_date`, NEW.`to_date`); END  
Suffix: upd 
Time  : AFTER 
-----------------------------------------------------------
-----------------------------------------------------------
Skipped trigger creation: 
Event : INSERT 
Name  : pt_osc_employees_salaries_ins 
SQL   : CREATE TRIGGER `pt_osc_employees_salaries_ins` AFTER INSERT ON `employees`.`salaries` FOR EACH ROW BEGIN DECLARE CONTINUE HANDLER FOR 1146 begin end; REPLACE INTO `employees`.`_salaries_new` (`emp_no`, `salary`, `from_date`, `to_date`) VALUES (NEW.`emp_no`, NEW.`salary`, NEW.`from_date`, NEW.`to_date`);END  
Suffix: ins 
Time  : AFTER 
-----------------------------------------------------------
Not copying rows because this is a dry run.
INSERT LOW_PRIORITY IGNORE INTO `employees`.`_salaries_new` (`emp_no`, `salary`, `from_date`, `to_date`) SELECT `emp_no`, `salary`, `from_date`, `to_date` FROM `employees`.`salaries` FORCE INDEX(`PRIMARY`) WHERE ((`emp_no` > ?) OR (`emp_no` = ? AND `from_date` >= ?)) AND ((`emp_no` < ?) OR (`emp_no` = ? AND `from_date` <= ?)) LOCK IN SHARE MODE /*pt-online-schema-change 20410 copy nibble*/
SELECT /*!40001 SQL_NO_CACHE */ `emp_no`, `emp_no`, `from_date` FROM `employees`.`salaries` FORCE INDEX(`PRIMARY`) WHERE ((`emp_no` > ?) OR (`emp_no` = ? AND `from_date` >= ?)) ORDER BY `emp_no`, `from_date` LIMIT ?, 2 /*next chunk boundary*/
Not swapping tables because this is a dry run.
Not dropping old table because this is a dry run.
Not dropping triggers because this is a dry run.
DROP TRIGGER IF EXISTS `employees`.`pt_osc_employees_salaries_del`
DROP TRIGGER IF EXISTS `employees`.`pt_osc_employees_salaries_upd`
DROP TRIGGER IF EXISTS `employees`.`pt_osc_employees_salaries_ins`
2021-03-29T19:29:28 Dropping new table...
DROP TABLE IF EXISTS `employees`.`_salaries_new`;
2021-03-29T19:29:28 Dropped new table OK.
Dry run complete.  `employees`.`salaries` was not altered.
```