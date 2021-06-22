---
layout: post
title:  "mysql+orchestrator部署"
date:   2021-06-10 12:31:29 +0900
categories: MySQL
---

# mysql主从+orchestrator集群管理
## 环境准备
```
1、版本：
系统版本：CentOS7 
Mysql版本：MySQL 5.7.32
orchestrator版本: orchestrator-3.1.4    
orchestrator下载地址：https://github.com/openark/orchestrator/releases

2、MySQL环境：
Mysql一主两从： 是属于被管理的三个mysql数据库
主库： 主机名：host1 IP：192.168.1.11  端口：3306  server_id：1113306  读写
从库1：主机名：host2 IP：192.168.1.22  端口：3306  server_id：2223306  只读
从库2：主机名：host3 IP：192.168.1.33  端口：3306  server_id：3333306  只读

3、Orchestrator环境：
后端mysql库 用于存储orch的配置信息 这里三个节点是做raft高可用：
192.168.1.55  3306        server_id：111  主机名：server1
192.168.1.66  3306        server_id：222  主机名：server2
192.168.1.77  3306        server_id：333  主机名：server3
```
## 环境配置
需要在orchestrator所在服务器中配置hosts记录，将所管理的mysql服务器加入到hosts解析中，因为当集群加入orchestrator之后，mysql会查询当前主机的hostname并进行上报，不配置解析就找不到对应的主机。
```
[root@host1 ~]# vi /etc/hosts
192.168.1.11  host1
192.168.1.22  host2
192.168.1.22  host3
```
## MySQL主从安装（三个节点）
上传MySQL安装包，版本为：MySQL 5.7.32，上传到/usr/local目录下：
```
[root@host1~]# cd /usr/local/
[root@host1 local]# ls mysql*
mysql-5.7.32-linux-glibc2.12-x86_64.tar
[root@host1 local]# tar zxvf mysql-5.7.32-linux-glibc2.5-x86_64.tar.gz
```
解压完成之后创建软连接并进入该目录
```
[root@host1 local]# ln -s mysql-5.7.9-linux-glibc2.5-x86_64 mysql
[root@host1 local]# cd mysql
```
创建用户和组
```
[root@host1 mysql]# groupadd mysql
[root@host1 mysql]# useradd -r -g mysql mysql
```
修改目录权限
```
[root@host1 mysql]# chown -R mysql .
[root@host1 mysql]# chgrp -R mysql .
```
创建数据目录和日志文件目录等
```
[root@host1 mysql]# mkdir /mydata /mydata/data /mydata/log /mydata/tmp
[root@host1 mysql]# chown -R mysql:mysql /mydata
```
创建my.cnf文件，文件绝对路径为/etc/my.cnf：
```
[mysql]
port=3306
prompt=\\u@\\d\\r:\\m:\\s>
no-auto-rehash

[client]
port=3306
socket=/tmp/mysql.sock

[mysqld]
socket=/tmp/mysql.sock
basedir=/usr/local/mysql/
datadir=/mydata/data
tmpdir=/mydata/tmp
log-error=/mydata/log/mysql.err
log_timestamps=SYSTEM
slow_query_log_file=/mydata/log/mysql.slow
pid-file=/mydata/data/mysql.pid

#innodb
innodb_file_format=Barracuda
innodb_data_home_dir=/mydata/data/
innodb_log_group_home_dir=/mydata/data/
innodb_data_file_path=ibdata1:1G:autoextend

# TODO 根据机器实际内存50%配置
innodb_buffer_pool_size=4G
innodb_buffer_pool_instances=8  # innodb_buffer_pool_size/8, 最少为4
innodb_log_files_in_group=3
innodb_log_file_size=1G
innodb_log_buffer_size=16M
innodb_checksum_algorithm=crc32
innodb_default_row_format=DYNAMIC

innodb_flush_log_at_trx_commit=1
innodb_max_dirty_pages_pct=75
innodb_io_capacity=2000 # 根据磁盘IO情况，SSD给5000
innodb_thread_concurrency=0
innodb_read_io_threads=8
innodb_write_io_threads=8
innodb_open_files=4096
innodb_file_per_table=1
innodb_flush_method=O_DIRECT
innodb_flush_neighbors=0
innodb_change_buffering=all
lock_wait_timeout=60
innodb_print_all_deadlocks=ON
# 默认不回滚整个事务 当事务因为获得锁而超时时 交给应用处理 决定是否回滚或重试
innodb_rollback_on_timeout=OFF

# oltp类型的应用 应该设置的更小
innodb_lock_wait_timeout=50
innodb_buffer_pool_dump_at_shutdown=ON
innodb_buffer_pool_load_at_startup=ON
innodb_buffer_pool_dump_pct=100
# MySQL5.7默认不允许语法错误 默认值为ON 调整为OFF
innodb_strict_mode=OFF
innodb_buffer_pool_chunk_size=128M
max_heap_table_size=128M
# 默认隔离级别为RR
transaction-isolation=REPEATABLE-READ

#replication
skip-slave-start
relay-log=relay.log
# 默认开启group commit
# TODO 需要通过压测测试出最佳组合
binlog_group_commit_sync_delay=0
# 短路条件
binlog_group_commit_sync_no_delay_count=0
# 使用logical_clock
slave_parallel_type=logical_clock
slave_parallel_workers=8
# TODO 该参数的值必须比max_allowed_packet大
slave_pending_jobs_size_max=16M
binlog_order_commits=1
# 为了确保事务的顺序 一定需要设置slave_preserve_commit_order=1 否则从库上事务回放的顺序可能和主库上不一致
slave_preserve_commit_order=1
binlog_gtid_simple_recovery=1

#binlog
log-bin=mysql-bin
server_id=1113306  # 主从不一致，需调整，ip c、d段地址+两位随机数
binlog_cache_size=1M
max_binlog_cache_size=2G
max_binlog_size=1024M
binlog-format=ROW
# TODO 是否使用双1 需要设置
sync_binlog=1
log-slave-updates
read_only=ON 
expire_logs_days=10
binlog_rows_query_log_events=ON
# MySQL5.7默认值设置为ABORT_SERVER 在binlog损坏时直接关闭server 调整为IGNORE_ERROR 后续可以讨论是否在binlog损坏时直接关闭server
binlog_error_action=ABORT_SERVER

#server
default-storage-engine=INNODB
character-set-server=utf8mb4

# TODO 具体看业务情况
collation_server=utf8mb4_unicode_ci
# MySQL5.7文档中包含character-set-client参数 但启动时不能设置该参数 所以为了兼容性考虑设置为loose-character-set-client
loose-character-set-client=utf8mb4
open_files_limit=102400
log_slow_admin_statements=1
# log_slow_verbosity=microtime,query_plan,innodb
log_queries_not_using_indexes=1
long_query_time=0.5
slow_query_log=1
log_slow_slave_statements=1
query_cache_type=0
table_definition_cache=65536
table_open_cache=65536
thread_stack=512K
thread_cache_size=256
sort_buffer_size=1M
join_buffer_size=1M
read_buffer_size=1M
skip-name-resolve

# 根据业务情况看是否需要开启ssl
skip-ssl
max_connections=3000
max_connect_errors=65536
max_allowed_packet=16M
connect_timeout=8
net_read_timeout=30
net_write_timeout=60
slave_net_timeout=16
interactive_timeout=1800
back_log=1024

innodb_stats_on_metadata=OFF

#MySQL5.7默认打印1592错误到日志文件 调整为OFF默认不打印
log_statements_unsafe_for_binlog=OFF

#MySQL5.7默认打开performance_schema
# 测试数据表明PS打开对性能影响非常小
performance_schema=ON

gtid_mode=ON
enforce-gtid-consistency=ON

#replicate
relay_log_recovery=1
relay_log_info_repository=TABLE
master_info_repository=TABLE
slave_compressed_protocol=ON

#user MySQL5.7增加用户密码过期策略 调整为默认用户密码不过期
# TODO 实际根据等保要求设置
default_password_lifetime=0


# sql_mode调整为和MySQL5.6一致
sql_mode='STRICT_TRANS_TABLES,NO_ENGINE_SUBSTITUTION,NO_ZERO_DATE,NO_ZERO_IN_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER'

#semi sync replication settings
plugin_dir=/usr/local/mysql/lib/plugin
plugin_load="rpl_semi_sync_master=semisync_master.so;rpl_semi_sync_slave=semisync_slave.so"
rpl_semi_sync_master_enabled=OFF
rpl_semi_sync_slave_enabled=ON
rpl_semi_sync_master_timeout=5000000
rpl_semi_sync_master_wait_for_slave_count=2
rpl_semi_sync_master_wait_point=AFTER_SYNC
```