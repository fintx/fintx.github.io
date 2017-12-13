---
title: MySQL线上死锁问题分析一例
date: 2017-07-23 16:18:04
categories: Database
tags: [database,mysql,deadlock]
comments: true
description: "一次真实线上故障的总结"
---

### 1 问题描述

#### 0)表结构:
```text
CREATE TABLE `t_acctsn_custno_pkid` (
  `CustNo` char(20) COLLATE utf8_bin NOT NULL,
  `CustType` char(2) COLLATE utf8_bin NOT NULL,
  `AcctSN` char(8) COLLATE utf8_bin NOT NULL,
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;
```

#### 1)表数据

|CustNo|CustType|AcctSN|id|
|-------|--|-|-|
|0000001|00|0|1|
|0000002|00|0|2|

#### 2)问题程序示意

```java
int concurrency = 5;
for (int c = 0; c < concurrency; c++) {
	new Thread(new Runnable() {
		@Override
		public void run() {
			while (true) {
				Connection conn = null;
				try {
					conn = DBUtil.getTxnConnection();
					PreparedStatement ps = DBUtil.getPreparedStatement(conn,
							"select acctSN+1 from t_acctsn_custno_pkid where custNo = ? for update");
					ps.setString(1, "0000001");
					ResultSet rs = ps.executeQuery();
					DBUtil.closePreparedStatement(ps);
					
					ps = DBUtil.getPreparedStatement(conn,
							"update  t_acctsn_custno_pkid  set acctSN=?  where custNo = ? ");
					ps.setString(1, acctSN);
					ps.setString(2, "0000001");
					ps.executeUpdate();
					DBUtil.closePreparedStatement(ps);
					
					DBUtil.closeTxnConnection(conn, true);
				} catch (Exception e) {
					e.printStackTrace();
					if (null != conn) {
						DBUtil.closeTxnConnection(conn, false);
					}
				}

			}
		}

	}).start();

	new Thread(new Runnable() {
		@Override
		public void run() {
			while (true) {
				Connection conn = null;
				try {
					conn = DBUtil.getTxnConnection();
					PreparedStatement ps = DBUtil.getPreparedStatement(conn,
							"select acctSN+1 from t_acctsn_custno_pkid where custNo = ? for update");
					ps.setString(1, "0000002");
					ResultSet rs = ps.executeQuery();
					DBUtil.closePreparedStatement(ps);
					
					ps = DBUtil.getPreparedStatement(conn,
							" update  t_acctsn_custno_pkid  set acctSN=?  where custNo = ? ");
					ps.setString(1, acctSN);
					ps.setString(2, "0000002");
					ps.executeUpdate();
					DBUtil.closePreparedStatement(ps);
					
					DBUtil.closeTxnConnection(conn, true);
				} catch (Exception e) {
					e.printStackTrace();
					if (null != conn) {
						DBUtil.closeTxnConnection(conn, false);
					}
				}

			}
		}

	}).start();
```

#### 3) MySQL死锁日志
```text
------------------------
LATEST DETECTED DEADLOCK
------------------------
2015-12-31 15:34:23 7fb7cdabb700
*** (1) TRANSACTION:
TRANSACTION 336633, ACTIVE 0 sec fetching rows
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 360, 2 row lock(s)
MySQL thread id 102345, OS thread handle 0x7fb7cdc41700, query id 1133481 172.16.5.15 root Sending data
select acctSN+1 from t_acctsn_custno_pkid where custNo = '0000001' for update
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 810 page no 3 n bits 72 index `PRIMARY` of table `checkaccount`.`t_acctsn_custno_pkid` trx id 336633 lock_mode X locks rec but not gap waiting
Record lock, heap no 3 PHYSICAL RECORD: n_fields 6; compact format; info bits 0
 0: len 8; hex 8000000000000002; asc         ;;
 1: len 6; hex 0000000522f0; asc     " ;;
 2: len 7; hex 31000001721bc8; asc 1   r  ;;
 3: len 20; hex 3030303030303220202020202020202020202020; asc 0000002             ;;
 4: len 2; hex 3030; asc 00;;
 5: len 8; hex 3135343434202020; asc 15444   ;;

*** (2) TRANSACTION:
TRANSACTION 336632, ACTIVE 0 sec starting index read
mysql tables in use 1, locked 1
3 lock struct(s), heap size 360, 2 row lock(s)
MySQL thread id 102340, OS thread handle 0x7fb7cdabb700, query id 1133491 172.16.5.15 root updating
update  t_acctsn_custno_pkid  set acctSN='15445'  where custNo = '0000002'
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 810 page no 3 n bits 72 index `PRIMARY` of table `checkaccount`.`t_acctsn_custno_pkid` trx id 336632 lock_mode X locks rec but not gap
Record lock, heap no 3 PHYSICAL RECORD: n_fields 6; compact format; info bits 0
 0: len 8; hex 8000000000000002; asc         ;;
 1: len 6; hex 0000000522f0; asc     " ;;
 2: len 7; hex 31000001721bc8; asc 1   r  ;;
 3: len 20; hex 3030303030303220202020202020202020202020; asc 0000002             ;;
 4: len 2; hex 3030; asc 00;;
 5: len 8; hex 3135343434202020; asc 15444   ;;

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 810 page no 3 n bits 72 index `PRIMARY` of table `checkaccount`.`t_acctsn_custno_pkid` trx id 336632 lock_mode X locks rec but not gap waiting
Record lock, heap no 2 PHYSICAL RECORD: n_fields 6; compact format; info bits 0
 0: len 8; hex 8000000000000001; asc         ;;
 1: len 6; hex 0000000522f1; asc     " ;;
 2: len 7; hex 32000001b423cc; asc 2    # ;;
 3: len 20; hex 3030303030303120202020202020202020202020; asc 0000001             ;;
 4: len 2; hex 3030; asc 00;;
 5: len 8; hex 3537323520202020; asc 5725    ;;

*** WE ROLL BACK TRANSACTION (2)
```

#### 4) MySQL数据库隔离级别
RC(READ COMMITTED)

### 2 知识准备
- 1 一条当前读的SQL语句，InnoDB与MySQL Server的交互，是一条一条进行的，因此，加锁也是一条一条进行的。先对一条满足条件的记录加锁，返回给MySQL Server，做一些DML操作；然后在读取下一条加锁，直至读取完毕。
- 2 RC级别下，由于custNo列上没有索引，因此只能走主键索引，进行全部扫描。主键索引上所有的记录，都被加上了X锁。无论记录是否满足条件，全部被加上X锁。既不是加表锁，也不是在满足条件的记录上加行锁。
在实际的实现中，MySQL有一些改进，在MySQL Server过滤条件，发现不满足后，会调用unlock_row方法，把不满足条件的记录放锁 (违背了2PL的约束)。这样做，保证了最后只会持有满足条件记录上的锁，但是每条记录的加锁操作还是不能省略的。

### 3 死锁分析
#### 0)死锁日志分析
- (1) TRANSACTION (select acctSN+1 from t_acctsn_custno_pkid where custNo = '0000001' for update) 共需3个锁(LOCK WAIT 3 lock struct(s)),其中1个表级意向写锁 IX ,2个行锁(2 row lock(s))。发生死锁时在尝试获取主键索引上(index `PRIMARY`) 某一行的X锁(table `checkaccount`.`t_acctsn_custno_pkid` trx id 336633 lock_mode X locks rec)
这一行的内容为 1 0000002 00 15444 
( 0: len 8; hex 8000000000000002; asc         ;;
 1: len 6; hex 0000000522f0; asc     " ;;
 2: len 7; hex 31000001721bc8; asc 1   r  ;;
 3: len 20; hex 3030303030303220202020202020202020202020; asc 0000002             ;;
 4: len 2; hex 3030; asc 00;;
 5: len 8; hex 3135343434202020; asc 15444   ;;
注意asc后的内容为行中每列的内容的ASCII码)
- (2) TRANSACTION (update  t_acctsn_custno_pkid  set acctSN='15445'  where custNo = '0000002') 共需3个锁(LOCK WAIT 3 lock struct(s)),其中1个表级意向写锁 IX ,2个行锁(2 row lock(s))。发生死锁时在已经持有主键索引上(index `PRIMARY`) 某一行的X锁(t table `checkaccount`.`t_acctsn_custno_pkid` trx id 336632 lock_mode X locks rec)。这一行的内容为  1 0000002 00 15444。并且发生死锁时在尝试获取主键索引上(index `PRIMARY`) 某一行的X锁table `checkaccount`.`t_acctsn_custno_pkid` trx id 336632 lock_mode X locks rec)。这一行的内容为 2 0000001 00 5725.
注1：死锁日志中(1) TRANSACTION中并不显示已持有的锁信息
注2：死锁日志分析中红字部分与表数据中的id列不对应，经过多次实验(2: len 7; hex 31000001721bc8; asc 1   r  ;;)这一列asc 后的内容并不显示不固定，有时是数字有时是字母有时是符号，推测与id有关但是显示的并不是id的值，估计与id列类型(`id` bigint(20) NOT NULL AUTO_INCREMENT,)有关系。

#### 1)事物时序分析

Transaction1| Transaction2
-|-
 |（select acctSN+1 from t_acctsn_custno_pkid where custNo = '0000002' for update）<br>1 对主键索引上所有行一行一行的加X锁<br>2释放custNo 为 '0000001'的行上的X锁保留custNo 为 '0000002'的行上的X锁
（select acctSN+1 from t_acctsn_custno_pkid where custNo = '0000001' for update）<br>3对主键索引上所有行一行一行的加X锁，首先先对主键索引上custno为0000001的行加X锁<br>4 尝试获取主键索引上custno为0000002上的X锁 | 
 |（update  t_acctsn_custno_pkid  set acctSN='15445'  where custNo = '0000002'）<br>5对主键索引上所有行一行一行的加X锁，尝试获取主键索引上custno为0000001上的X锁

注3：根据前面的知识储备，及对死锁日志的分析，在上表的时序下满足了死锁产生的条件。

### 4 解决方法
根据以上分析，绕了一大圈解决方法其实很简单 在custNo上加索引即可，其实custNo作为查询条件是应该有索引的，上线的同学疏忽了，造成了比较严重的线上故障，引以为戒。

