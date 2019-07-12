---
title: Apache Flink 漫谈系列 - 流表对偶(duality)性
date: 2019-01-01 18:18:18
updated: 2019-01-01 18:18:18
categories: Apache Flink 漫谈
tags: Flink
---
# 实际问题
很多大数据计算产品，都对用户提供了SQL API，比如Hive, Spark, Flink等，那么SQL作为传统关系数据库的查询语言，是应用在批查询场景的。Hive和Spark本质上都是Batch的计算模式(在《Apache Flink 漫谈系列 - 概述》我们介绍过Spark是Micro Batching模式)，提供SQL API很容易被人理解，但是Flink是纯流（Native Streaming）的计算模式, 流与批在数据集和计算过程上有很大的区别.

<!-- more --> 

如图所示：

![](EF81F039-86CC-4799-8DCF-F3B0144B76EC.png)

* 批查询场景的特点 - 有限数据集，一次查询返回一个计算结果就结束查询

* 流查询场景的特点 - 无限数据集，一次查询不断修正计算结果，查询永远不结束

我们发现批与流的查询场景在数据集合和计算过程上都有很大的不同，那么基于Native Streaming模式的Apache Flink为啥也能为用户提供SQL API呢？

# 流与批的语义关系
我们知道SQL都是作用于关系表的，在传统数据库中进行查询时候，SQL所要查询的表在触发查询时候数据是不会变化的，也就是说在查询那一刻，表是一张静态表，相当于是一个有限的批数据，这样也说明SQL是源于对批计算的查询的，那么要回答Apache Flink为啥也能为用户提供SQL API，我们首先要理解流与批在语义层面的关系。我们以一个具体示例说明，如下图：
![](3794F522-3458-4386-A101-9458FD2D8DBF.png)
上图展现的是一个携带时间戳和用户名的点击事件流，我们先对这些事件流进行流式统计，同时在最后的流事件上触发批计算。流计算中每接收一个数据都会触发一次计算，我们以2018/4/30 22:37:45 Mary到来那一时间切片看，无论是在流还是批上计算结果都是6。也就是说在相同的数据源，相同的查询逻辑下，流和批的计算结果是相同的。相同的SQL在流和批这两种模式下，最终结果是一致的，那么流与批在语义上是完全相同的。

# 流与表的关系
流与批在语义上是一致的，SQL是作用于表的，那么要回答Apache Flink为啥也能为用户提供SQL API的问题，就变成了流与表是否具有等价性，也就是本篇要重点介绍的为什么流表具有对偶(duality)性？如下图所示，一张表可以看做为流吗？同样流可以看做是一张表吗？如果可以需要怎样的条件和变通？
![](41690DE4-8705-4305-BF74-AC47F8C870B3.png)

# MySQL主备复制
在介绍流与表的关系之前我们先聊聊MySQL的主备复制，binlog是MySQL实现主备复制的核心手段，简单来说MySQL主备复制实现分成三个步骤：
* Master将改变(change logs)以二进制日志事件(binary log events)形式记录到二进制日志(binlog)中；
* Slave将Master的binary log events拷贝到它的中继日志(relay log)；
* Slave重做中继日志中的事件，将改变反映到数据；

具体如下图所示:
   ![](FF4F58BD-6C4F-4065-A0B5-EC66B3C31BD0.png)
   
# binlog
接下来我们从binlog模式，binlog格式以及通过查看binlog的具体内容来详尽介绍binlog与表的关系。
## binlog模式
上面介绍的MySQL主备复制的核心手段是利用binlog实现的，那边binlog会记录那些内容呢？binlog记录了数据库所有的增、删、更新等操作。MySQL支持三种方式记录binlog:
* statement-based logging - Events contain SQL statements that produce data changes (inserts, updates, deletes);
* row-based logging - Events describe changes to individual rows;
* mixed-base logging - 该模式默认是statement-based，当遇到如下情况会自动切换到row-based:
    * NDB存储引擎，DML操作以row格式记录;
    * 使用UUID()、USER()、CURRENT_USER()、FOUND_ROWS()等不确定函数;
    * 使用Insert Delay语句;
    * 使用用户自定义函数(UDF);
    * 使用临时表;
    
## binlog格式
我们以row-based 模式为例介绍一下[binlog的存储格式](https://dev.MySQL.com/doc/internals/en/event-structure.html) ，所有的 binary log events都是字节序列，由两部分组成：
* event header
* event data

关于event header和event data 的格式在数据库的不同版本略有不同，但共同的地方如下：
```
+=====================================+
| event  | timestamp         0 : 4    |
| header +----------------------------+
|        | type_code         4 : 1    |
|        +----------------------------+
|        | server_id         5 : 4    |
|        +----------------------------+
|        | event_length      9 : 4    |
|        +----------------------------+
|        |不同版本不一样(省略)          |
+=====================================+
| event  | fixed part                 |
| data   +----------------------------+
|        | variable part              |
+=====================================+
```
这里有个值得我们注意的地方就是在binlog的header中有一个属性是timestamp，这个属性是标识了change发生的先后顺序，在备库进行复制时候会严格按照时间顺序进行log的重放。

## binlog的生成
我们以对MySQL进行实际操作的方式，直观的介绍一下binlog的生成，binlog是二进制存储的，下面我们会利用工具查看binlog的文本内容。

* 查看一下binlog是否打开：
```
show variables like 'log_bin'
    -&gt; ;
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| log_bin       | ON    |
+---------------+-------+
1 row in set (0.00 sec)
```
* 查看一下binlog的模式(我需要row-base模式)：
```
show variables like 'binlog_format';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| binlog_format | ROW   |
+---------------+-------+
1 row in set (0.00 sec)
```
* 清除现有的binlog
```
MySQL&gt; reset master;
Query OK, 0 rows affected (0.00 sec)创建一张我们做实验的表MySQL&gt; create table tab(
    -&gt;    id INT NOT NULL AUTO_INCREMENT,
    -&gt;    user VARCHAR(100) NOT NULL,
    -&gt;    clicks INT NOT NULL,
    -&gt;    PRIMARY KEY (id)
    -&gt; );
Query OK, 0 rows affected (0.10 sec)

MySQL&gt; show tables;
+-------------------+
| Tables_in_Apache Flinkdb |
+-------------------+
| tab         |
+-------------------+
1 row in set (0.00 sec)
```

* 进行DML操作
```
MySQL&gt; insert into tab(user, clicks) values ('Mary', 1);
Query OK, 1 row affected (0.03 sec)

MySQL&gt; insert into tab(user, clicks) values ('Bob', 1);
Query OK, 1 row affected (0.08 sec)

MySQL&gt; update tab set clicks=2 where user='Mary'
    -&gt; ;
Query OK, 1 row affected (0.06 sec)
Rows matched: 1  Changed: 1  Warnings: 0

MySQL&gt; insert into tab(user, clicks) values ('Llz', 1);
Query OK, 1 row affected (0.08 sec)

MySQL&gt; update tab set clicks=2 where user='Bob';
Query OK, 1 row affected (0.01 sec)
Rows matched: 1  Changed: 1  Warnings: 0

MySQL&gt; update tab set clicks=3 where user='Mary';
Query OK, 1 row affected (0.05 sec)
Rows matched: 1  Changed: 1  Warnings: 0

MySQL&gt; select * from tab;
+----+------+--------+
| id | user | clicks |
+----+------+--------+
|  1 | Mary |      3 |
|  2 | Bob  |      2 |
|  3 | Llz  |      1 |
+----+------+--------+
3 rows in set (0.00 sec)
```
* 查看正在操作的binlog
```
MySQL&gt; show master status\G
*************************** 1. row ***************************
             File: binlog.000001
         Position: 2547
     Binlog_Do_DB: 
 Binlog_Ignore_DB: 
Executed_Gtid_Set: 
1 row in set (0.00 sec)
```
上面 binlog.000001 文件是我们正在操作的binlog。

* 查看binlog.000001文件的操作记录
```
MySQL&gt; show binlog events in 'binlog.000001';
+---------------+------+----------------+-----------+-------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Log_name      | Pos  | Event_type     | Server_id | End_log_pos | Info                                                                                                                                                                |
+---------------+------+----------------+-----------+-------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| binlog.000001 |    4 | Format_desc    |         1 |         124 | Server ver: 8.0.11, Binlog ver: 4                                                                                                                                   |
| binlog.000001 |  124 | Previous_gtids |         1 |         155 |                                                                                                                                                                     |
| binlog.000001 |  155 | Anonymous_Gtid |         1 |         228 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                                                                                                                                |
| binlog.000001 |  228 | Query          |         1 |         368 | use `Apache Flinkdb`; DROP TABLE `tab` /* generated by server */ /* xid=22 */                                                                                        |
| binlog.000001 |  368 | Anonymous_Gtid |         1 |         443 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                                                                                                                                |
| binlog.000001 |  443 | Query          |         1 |         670 | use `Apache Flinkdb`; create table tab(
   id INT NOT NULL AUTO_INCREMENT,
   user VARCHAR(100) NOT NULL,
   clicks INT NOT NULL,
   PRIMARY KEY (id)
) /* xid=23 */ |
| binlog.000001 |  670 | Anonymous_Gtid |         1 |         745 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                                                                                                                                |
| binlog.000001 |  745 | Query          |         1 |         823 | BEGIN                                                                                                                                                               |
| binlog.000001 |  823 | Table_map      |         1 |         890 | table_id: 96 (Apache Flinkdb.tab)                                                                                                                                    |
| binlog.000001 |  890 | Write_rows     |         1 |         940 | table_id: 96 flags: STMT_END_F                                                                                                                                      |
| binlog.000001 |  940 | Xid            |         1 |         971 | COMMIT /* xid=25 */                                                                                                                                                 |
| binlog.000001 |  971 | Anonymous_Gtid |         1 |        1046 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                                                                                                                                |
| binlog.000001 | 1046 | Query          |         1 |        1124 | BEGIN                                                                                                                                                               |
| binlog.000001 | 1124 | Table_map      |         1 |        1191 | table_id: 96 (Apache Flinkdb.tab)                                                                                                                                    |
| binlog.000001 | 1191 | Write_rows     |         1 |        1240 | table_id: 96 flags: STMT_END_F                                                                                                                                      |
| binlog.000001 | 1240 | Xid            |         1 |        1271 | COMMIT /* xid=26 */                                                                                                                                                 |
| binlog.000001 | 1271 | Anonymous_Gtid |         1 |        1346 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                                                                                                                                |
| binlog.000001 | 1346 | Query          |         1 |        1433 | BEGIN                                                                                                                                                               |
| binlog.000001 | 1433 | Table_map      |         1 |        1500 | table_id: 96 (Apache Flinkdb.tab)                                                                                                                                    |
| binlog.000001 | 1500 | Update_rows    |         1 |        1566 | table_id: 96 flags: STMT_END_F                                                                                                                                      |
| binlog.000001 | 1566 | Xid            |         1 |        1597 | COMMIT /* xid=27 */                                                                                                                                                 |
| binlog.000001 | 1597 | Anonymous_Gtid |         1 |        1672 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                                                                                                                                |
| binlog.000001 | 1672 | Query          |         1 |        1750 | BEGIN                                                                                                                                                               |
| binlog.000001 | 1750 | Table_map      |         1 |        1817 | table_id: 96 (Apache Flinkdb.tab)                                                                                                                                    |
| binlog.000001 | 1817 | Write_rows     |         1 |        1866 | table_id: 96 flags: STMT_END_F                                                                                                                                      |
| binlog.000001 | 1866 | Xid            |         1 |        1897 | COMMIT /* xid=28 */                                                                                                                                                 |
| binlog.000001 | 1897 | Anonymous_Gtid |         1 |        1972 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                                                                                                                                |
| binlog.000001 | 1972 | Query          |         1 |        2059 | BEGIN                                                                                                                                                               |
| binlog.000001 | 2059 | Table_map      |         1 |        2126 | table_id: 96 (Apache Flinkdb.tab)                                                                                                                                    |
| binlog.000001 | 2126 | Update_rows    |         1 |        2190 | table_id: 96 flags: STMT_END_F                                                                                                                                      |
| binlog.000001 | 2190 | Xid            |         1 |        2221 | COMMIT /* xid=29 */                                                                                                                                                 |
| binlog.000001 | 2221 | Anonymous_Gtid |         1 |        2296 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                                                                                                                                |
| binlog.000001 | 2296 | Query          |         1 |        2383 | BEGIN                                                                                                                                                               |
| binlog.000001 | 2383 | Table_map      |         1 |        2450 | table_id: 96 (Apache Flinkdb.tab)                                                                                                                                    |
| binlog.000001 | 2450 | Update_rows    |         1 |        2516 | table_id: 96 flags: STMT_END_F                                                                                                                                      |
| binlog.000001 | 2516 | Xid            |         1 |        2547 | COMMIT /* xid=30 */                                                                                                                                                 |
+---------------+------+----------------+-----------+-------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------+
36 rows in set (0.00 sec)
```
上面我们进行了3次insert和3次update，那么在binlog中我们看到了三条Write_rows和三条Update_rows，并且在记录顺序和操作顺序保持一致，接下来我们看看Write_rows和Update_rows的具体timestamp和data明文。

* 导出明文
```
sudo MySQLbinlog --start-datetime='2018-04-29 00:00:03' --stop-datetime='2018-05-02 00:30:00' --base64-output=decode-rows -v /usr/local/MySQL/data/binlog.000001 &gt; ~/binlog.txt
```

打开binlog.txt 内容如下:
```
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=1*/;
/*!50003 SET @OLD_COMPLETION_TYPE=@@COMPLETION_TYPE,COMPLETION_TYPE=0*/;
DELIMITER /*!*/;
# at 4
#180430 22:29:33 server id 1  end_log_pos 124 CRC32 0xff61797c  Start: binlog v 4, server v 8.0.11 created 180430 22:29:33 at startup
# Warning: this binlog is either in use or was not closed properly.
ROLLBACK/*!*/;
# at 124
#180430 22:29:33 server id 1  end_log_pos 155 CRC32 0x629ae755  Previous-GTIDs
# [empty]
# at 155
#180430 22:32:11 server id 1  end_log_pos 228 CRC32 0xbde49fca  Anonymous_GTID  last_committed=0        sequence_number=1       rbr_only=no     original_committed_timestamp=1525098731207902   immediate_commit_timestamp=1525098731207902     transaction_length=213
# original_commit_timestamp=1525098731207902 (2018-04-30 22:32:11.207902 CST)
# immediate_commit_timestamp=1525098731207902 (2018-04-30 22:32:11.207902 CST)
/*!80001 SET @@session.original_commit_timestamp=1525098731207902*//*!*/;
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 228
#180430 22:32:11 server id 1  end_log_pos 368 CRC32 0xe5f330e7  Query   thread_id=9     exec_time=0     error_code=0    Xid = 22
use `Apache Flinkdb`/*!*/;
SET TIMESTAMP=1525098731/*!*/;
SET @@session.pseudo_thread_id=9/*!*/;
SET @@session.foreign_key_checks=1, @@session.sql_auto_is_null=0, @@session.unique_checks=1, @@session.autocommit=1/*!*/;
SET @@session.sql_mode=1168113696/*!*/;
SET @@session.auto_increment_increment=1, @@session.auto_increment_offset=1/*!*/;
/*!\C utf8mb4 *//*!*/;
SET @@session.character_set_client=255,@@session.collation_connection=255,@@session.collation_server=255/*!*/;
SET @@session.lc_time_names=0/*!*/;
SET @@session.collation_database=DEFAULT/*!*/;
/*!80005 SET @@session.default_collation_for_utf8mb4=255*//*!*/;
DROP TABLE `tab` /* generated by server */
/*!*/;
# at 368
#180430 22:32:21 server id 1  end_log_pos 443 CRC32 0x50e5acb7  Anonymous_GTID  last_committed=1        sequence_number=2       rbr_only=no     original_committed_timestamp=1525098741628960   immediate_commit_timestamp=1525098741628960     transaction_length=302
# original_commit_timestamp=1525098741628960 (2018-04-30 22:32:21.628960 CST)
# immediate_commit_timestamp=1525098741628960 (2018-04-30 22:32:21.628960 CST)
/*!80001 SET @@session.original_commit_timestamp=1525098741628960*//*!*/;
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 443
#180430 22:32:21 server id 1  end_log_pos 670 CRC32 0xe1353dd6  Query   thread_id=9     exec_time=0     error_code=0    Xid = 23
SET TIMESTAMP=1525098741/*!*/;
create table tab(
   id INT NOT NULL AUTO_INCREMENT,
   user VARCHAR(100) NOT NULL,
   clicks INT NOT NULL,
   PRIMARY KEY (id)
)
/*!*/;
# at 670
#180430 22:36:53 server id 1  end_log_pos 745 CRC32 0xcf436fbb  Anonymous_GTID  last_committed=2        sequence_number=3       rbr_only=yes    original_committed_timestamp=1525099013988373   immediate_commit_timestamp=1525099013988373     transaction_length=301
/*!50718 SET TRANSACTION ISOLATION LEVEL READ COMMITTED*//*!*/;
# original_commit_timestamp=1525099013988373 (2018-04-30 22:36:53.988373 CST)
# immediate_commit_timestamp=1525099013988373 (2018-04-30 22:36:53.988373 CST)
/*!80001 SET @@session.original_commit_timestamp=1525099013988373*//*!*/;
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 745
#180430 22:36:53 server id 1  end_log_pos 823 CRC32 0x71c64dd2  Query   thread_id=9     exec_time=0     error_code=0
SET TIMESTAMP=1525099013/*!*/;
BEGIN
/*!*/;
# at 823
#180430 22:36:53 server id 1  end_log_pos 890 CRC32 0x63792f6b  Table_map: `Apache Flinkdb`.`tab` mapped to number 96
# at 890
#180430 22:36:53 server id 1  end_log_pos 940 CRC32 0xf2dade22  Write_rows: table id 96 flags: STMT_END_F
### INSERT INTO `Apache Flinkdb`.`tab`
### SET
###   @1=1
###   @2='Mary'
###   @3=1
# at 940
#180430 22:36:53 server id 1  end_log_pos 971 CRC32 0x7db3e61e  Xid = 25
COMMIT/*!*/;
# at 971
#180430 22:37:06 server id 1  end_log_pos 1046 CRC32 0xd05dd12c         Anonymous_GTID  last_committed=3        sequence_number=4       rbr_only=yes    original_committed_timestamp=1525099026328547   immediate_commit_timestamp=1525099026328547     transaction_length=300
/*!50718 SET TRANSACTION ISOLATION LEVEL READ COMMITTED*//*!*/;
# original_commit_timestamp=1525099026328547 (2018-04-30 22:37:06.328547 CST)
# immediate_commit_timestamp=1525099026328547 (2018-04-30 22:37:06.328547 CST)
/*!80001 SET @@session.original_commit_timestamp=1525099026328547*//*!*/;
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 1046
#180430 22:37:06 server id 1  end_log_pos 1124 CRC32 0x80f259e0         Query   thread_id=9     exec_time=0     error_code=0
SET TIMESTAMP=1525099026/*!*/;
BEGIN
/*!*/;
# at 1124
#180430 22:37:06 server id 1  end_log_pos 1191 CRC32 0x255903ba         Table_map: `Apache Flinkdb`.`tab` mapped to number 96
# at 1191
#180430 22:37:06 server id 1  end_log_pos 1240 CRC32 0xe76bfc79         Write_rows: table id 96 flags: STMT_END_F
### INSERT INTO `Apache Flinkdb`.`tab`
### SET
###   @1=2
###   @2='Bob'
###   @3=1
# at 1240
#180430 22:37:06 server id 1  end_log_pos 1271 CRC32 0x83cddfef         Xid = 26
COMMIT/*!*/;
# at 1271
#180430 22:37:15 server id 1  end_log_pos 1346 CRC32 0x7095baee         Anonymous_GTID  last_committed=4        sequence_number=5       rbr_only=yes    original_committed_timestamp=1525099035811597   immediate_commit_timestamp=1525099035811597     transaction_length=326
/*!50718 SET TRANSACTION ISOLATION LEVEL READ COMMITTED*//*!*/;
# original_commit_timestamp=1525099035811597 (2018-04-30 22:37:15.811597 CST)
# immediate_commit_timestamp=1525099035811597 (2018-04-30 22:37:15.811597 CST)
/*!80001 SET @@session.original_commit_timestamp=1525099035811597*//*!*/;
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 1346
#180430 22:37:15 server id 1  end_log_pos 1433 CRC32 0x70ef97e2         Query   thread_id=9     exec_time=0     error_code=0
SET TIMESTAMP=1525099035/*!*/;
BEGIN
/*!*/;
# at 1433
#180430 22:37:15 server id 1  end_log_pos 1500 CRC32 0x75f1f399         Table_map: `Apache Flinkdb`.`tab` mapped to number 96
# at 1500
#180430 22:37:15 server id 1  end_log_pos 1566 CRC32 0x256bd4b8         Update_rows: table id 96 flags: STMT_END_F
### UPDATE `Apache Flinkdb`.`tab`
### WHERE
###   @1=1
###   @2='Mary'
###   @3=1
### SET
###   @1=1
###   @2='Mary'
###   @3=2
# at 1566
#180430 22:37:15 server id 1  end_log_pos 1597 CRC32 0x93c86579         Xid = 27
COMMIT/*!*/;
# at 1597
#180430 22:37:27 server id 1  end_log_pos 1672 CRC32 0xe8bd63e7         Anonymous_GTID  last_committed=5        sequence_number=6       rbr_only=yes    original_committed_timestamp=1525099047219517   immediate_commit_timestamp=1525099047219517     transaction_length=300
/*!50718 SET TRANSACTION ISOLATION LEVEL READ COMMITTED*//*!*/;
# original_commit_timestamp=1525099047219517 (2018-04-30 22:37:27.219517 CST)
# immediate_commit_timestamp=1525099047219517 (2018-04-30 22:37:27.219517 CST)
/*!80001 SET @@session.original_commit_timestamp=1525099047219517*//*!*/;
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 1672
#180430 22:37:27 server id 1  end_log_pos 1750 CRC32 0x5356c3c7         Query   thread_id=9     exec_time=0     error_code=0
SET TIMESTAMP=1525099047/*!*/;
BEGIN
/*!*/;
# at 1750
#180430 22:37:27 server id 1  end_log_pos 1817 CRC32 0x37e6b1ce         Table_map: `Apache Flinkdb`.`tab` mapped to number 96
# at 1817
#180430 22:37:27 server id 1  end_log_pos 1866 CRC32 0x6ab1bbe6         Write_rows: table id 96 flags: STMT_END_F
### INSERT INTO `Apache Flinkdb`.`tab`
### SET
###   @1=3
###   @2='Llz'
###   @3=1
# at 1866
#180430 22:37:27 server id 1  end_log_pos 1897 CRC32 0x3b62b153         Xid = 28
COMMIT/*!*/;
# at 1897
#180430 22:37:36 server id 1  end_log_pos 1972 CRC32 0x603134c1         Anonymous_GTID  last_committed=6        sequence_number=7       rbr_only=yes    original_committed_timestamp=1525099056866022   immediate_commit_timestamp=1525099056866022     transaction_length=324
/*!50718 SET TRANSACTION ISOLATION LEVEL READ COMMITTED*//*!*/;
# original_commit_timestamp=1525099056866022 (2018-04-30 22:37:36.866022 CST)
# immediate_commit_timestamp=1525099056866022 (2018-04-30 22:37:36.866022 CST)
/*!80001 SET @@session.original_commit_timestamp=1525099056866022*//*!*/;
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 1972
#180430 22:37:36 server id 1  end_log_pos 2059 CRC32 0xe17df4e4         Query   thread_id=9     exec_time=0     error_code=0
SET TIMESTAMP=1525099056/*!*/;
BEGIN
/*!*/;
# at 2059
#180430 22:37:36 server id 1  end_log_pos 2126 CRC32 0x53888b05         Table_map: `Apache Flinkdb`.`tab` mapped to number 96
# at 2126
#180430 22:37:36 server id 1  end_log_pos 2190 CRC32 0x85f34996         Update_rows: table id 96 flags: STMT_END_F
### UPDATE `Apache Flinkdb`.`tab`
### WHERE
###   @1=2
###   @2='Bob'
###   @3=1
### SET
###   @1=2
###   @2='Bob'
###   @3=2
# at 2190
#180430 22:37:36 server id 1  end_log_pos 2221 CRC32 0x877f1e23         Xid = 29
COMMIT/*!*/;
# at 2221
#180430 22:37:45 server id 1  end_log_pos 2296 CRC32 0xfbc7e868         Anonymous_GTID  last_committed=7        sequence_number=8       rbr_only=yes    original_committed_timestamp=1525099065089940   immediate_commit_timestamp=1525099065089940     transaction_length=326
/*!50718 SET TRANSACTION ISOLATION LEVEL READ COMMITTED*//*!*/;
# original_commit_timestamp=1525099065089940 (2018-04-30 22:37:45.089940 CST)
# immediate_commit_timestamp=1525099065089940 (2018-04-30 22:37:45.089940 CST)
/*!80001 SET @@session.original_commit_timestamp=1525099065089940*//*!*/;
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 2296
#180430 22:37:45 server id 1  end_log_pos 2383 CRC32 0x8a514364         Query   thread_id=9     exec_time=0     error_code=0
SET TIMESTAMP=1525099065/*!*/;
BEGIN
/*!*/;
# at 2383
#180430 22:37:45 server id 1  end_log_pos 2450 CRC32 0xdf18ca60         Table_map: `Apache Flinkdb`.`tab` mapped to number 96
# at 2450
#180430 22:37:45 server id 1  end_log_pos 2516 CRC32 0xd50de69f         Update_rows: table id 96 flags: STMT_END_F
### UPDATE `Apache Flinkdb`.`tab`
### WHERE
###   @1=1
###   @2='Mary'
###   @3=2
### SET
###   @1=1
###   @2='Mary'
###   @3=3
# at 2516
#180430 22:37:45 server id 1  end_log_pos 2547 CRC32 0x94f89393         Xid = 30
COMMIT/*!*/;
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by MySQLbinlog */ /*!*/;
DELIMITER ;
# End of log file
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;
```
* 梳理操作和binlog的记录关系
<div class="table-contianer"><table><tbody><tr><td style="width: 200px;">DML</td><td style="width: 216.667px;">binlog-header(timestamp)</td><td style="width: 201.111px;">data</td></tr><tr><td style="width: 200px;">insert into blink_tab(user, clicks) values ('Mary', 1);</td><td style="width: 216.667px;"><p>1525099013</p><p>(2018/4/30 22:36:53)</p></td><td style="width: 201.111px;">## INSERT INTO `blinkdb`.`blink_tab`<br />### SET<br />### @1=1<br />### @2='Mary'<br />### @3=1</td></tr><tr><td style="width: 200px;">insert into blink_tab(user, clicks) values ('Bob', 1);</td><td style="width: 216.667px;"><p>1525099026</p><p>(2018/4/30 22:37:06)</p></td><td style="width: 201.111px;">### INSERT INTO `blinkdb`.`blink_tab`<br />### SET<br />### @1=2<br />### @2='Bob'<br />### @3=1&nbsp;</td></tr><tr><td style="width: 200px;">&nbsp;update blink_tab set clicks=2 where user='Mary';</td><td style="width: 216.667px;"><p>&nbsp;1525099035</p><p>(2018/4/30 22:37:15)</p></td><td style="width: 201.111px;">&nbsp;### UPDATE `blinkdb`.`blink_tab`<br />### WHERE<br />### @1=1<br />### @2='Mary'<br />### @3=1<br />### SET<br />### @1=1<br />### @2='Mary'<br />### @3=2</td></tr><tr><td style="width: 200px;">&nbsp;insert into blink_tab(user, clicks) values ('Llz', 1);</td><td style="width: 216.667px;"><p>1525099047</p><p>(2018/4/30 22:37:27)&nbsp;</p></td><td style="width: 201.111px;">### INSERT INTO `blinkdb`.`blink_tab`<br />### SET<br />### @1=3<br />### @2='Llz'<br />### @3=1&nbsp;</td></tr><tr><td style="width: 200px;">&nbsp;update blink_tab set clicks=2 where user='Bob';</td><td style="width: 216.667px;"><p>&nbsp;1525099056</p><p>(2018/4/30 22:37:36)</p></td><td style="width: 201.111px;">### UPDATE `blinkdb`.`blink_tab`<br />### WHERE<br />### @1=2<br />### @2='Bob'<br />### @3=1<br />### SET<br />### @1=2<br />### @2='Bob'<br />### @3=2&nbsp;</td></tr><tr><td style="width: 200px;">&nbsp;update blink_tab set clicks=3 where user='Mary';</td><td style="width: 216.667px;"><p>1525099065</p><p>(2018/4/30 22:37:45)&nbsp;</p></td><td style="width: 201.111px;">### UPDATE `blinkdb`.`blink_tab`<br />### WHERE<br />### @1=1<br />### @2='Mary'<br />### @3=2<br />### SET<br />### @1=1<br />### @2='Mary'<br />### @3=3&nbsp;</td></tr></tbody></table></div>

* 简化一下binlog
![](DC38424C-75A9-4DE9-9485-32F5F3D0F55F.png)

* replay binlog会得到如下表数据（按timestamp顺序)
![](2052C4F1-2EE4-41D1-8A8D-165A40827462.png)

* 表与binlog的关系简单示意如下
![](F736DF56-0C25-4DD9-84D8-200F5DC4EC38.png)
# 流表对偶(duality)性
前面我花费了一些时间介绍了MySQL主备复制机制和binlog的数据格式，binlog中携带时间戳，我们将所有表的操作都按时间进行记录下来形成binlog，而对binlog的event进行重放的过程就是流数据处理的过程，重放的结果恰恰又形成了一张表。也就是表的操作会形成携带时间的事件流，对流的处理又会形成一张不断变化的表，表和流具有等价性，可以互转。随着时间推移，DML操作不断进行，那么表的内容也不断变化，具体如下：

![](299C46FC-E45C-4CCE-AC60-4EFDA2E101E9.png)

如上图所示内容，流和表具备相同的特征：
* 表 - Schema，Data，DML操作时间
* 流 - Schema，Data, Data处理时间

我们发现，虽然大多数表上面没有明确的显示出DML操作时间，但本质上数据库系统里面是有数据操作时间信息的，这个和流上数据的处理时间（processing time)/产生时间（event-time)相对应。流与表具备相同的特征，可以信息无损的相互转换，我称之为流表对偶(duality)性。

上面我们描述的表，在流上称之为动态表(Dynamic Table)，原因是在流上面任何一个事件的到来都是对表上数据的一次更新(包括插入和删除),表的内容是不断的变化的，任何一个时刻流的状态和表的快照一一对应。流与动态表(Dynamic Table)在时间维度上面具有等价性，这种等价性我们称之为流和动态表(Dynamic Table)的对偶(duality)性。

# 小结
本篇主要介绍Apache Flink作为一个流计算平台为什么可以为用户提供SQL API。其根本原因是如果将流上的数据看做是结构化的数据，流任务的核心是将一个具有时间属性的结构化数据变成同样具有时间属性的另一个结构化数据，而表的数据变化过程binlog恰恰就是一份具有时间属性的流数据，流与表具有信息无损的相互转换的特性，这种流表对偶性也决定了Apache Flink可以采用SQL作为流任务的开发语言。

