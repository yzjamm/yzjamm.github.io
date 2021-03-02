---
title: MySQL Enterprise Backup Point-in-time Recovery
date: 2018-12-05
comments: true
categories: [MySQL, StudyNotes]
tags: [MySQL Enterprise Backup, Point-in-time Recovery, Docker, Study Notes]
typora-copy-images-to: ..\assets\img
---
Both MySQL Enterprise Backup and Percona XtraBackup can provide InnoDB hot backup, incremental backup, point-in-time recovery. But Both tools need to run  "FLUSH TABLES WITH READ LOCK' to make sure backup job gets a consistent backup after locking the entire database. Even if we specify --only-innodb option, the job still need to lock the database for a little while,  except --no-locking option. When MySQL Enterprise Backup finishes, it will provide MySQL binlog position like below:

INFO: MySQL binlog position: filename mysqlslave01-bin.000004, position 235

Point-in-time recovery replying on this information as starting point to roll forward the database . But --no-locking option won't give this information when the job completes.

The concern is the command "FLUSH TABLES WITH READ LOCK" has to wait until long queries finish. And it will show state "Waiting for table flush" and block other DML sessions with "Waiting for global read lock".  

Percona XtraBackup has better control for the lock. As it mentioned "Backup locks is a lightweight alternative to FLUSH TABLES WITH READ LOCK available in Percona Server 5.6+. Percona XtraBackup uses them automatically to copy non-InnoDB data to avoid blocking DML queries that modify InnoDB tables."

This blog post title is point-in-time recovery. So we are not discussing lock issue here. Our strategy is running the backup job on the slave database. and use slave binary log files to do point-in-time. One parameter needs to be turned on - log-slave-updates. This option makes a slave write udpates whtat are received from a master server and performed by the slave's SQL thread to the slave's own binary log.

Let's do a testing.

## Prerequisites

- Download mysql-commercial-server-minimal-8.0.12-1.1.el7.x86_64.rpm




## Build Master/Slave in Docker Containers

Start From the Dockerfile	

```dockerfile
FROM oraclelinux
add mysqlserver.rpm /root
add mysqlshell.rpm /root
ARG MYSQLD_URL=/root/mysqlserver.rpm
ARG SHELL_URL=/root/mysqlshell.rpm

# Install server
RUN rpmkeys --import http://repo.mysql.com/RPM-GPG-KEY-mysql \
  && yum install -y $MYSQLD_URL \
  && yum install -y $SHELL_URL \
  && yum install -y libpwquality \
  && yum install -y hostname \
  && yum install -y less vim-minimal net-tools \
  && rm -rf /var/cache/yum/* \
  && rm -rf /root/*.rpm

CMD tail -f /dev/null
EXPOSE 3306

```



**Build Docker Image**

```powershell
docker build -t yzjamm/mysqlserver -f Dockerfile_mysqlserver.txt .
```



**Run Master/Slave Docker Containers**

```powershell
docker run --name=master --hostname=master -v C:\docker\:/data/backups --network=grnet -p 3336:3306 -itd yzjamm/mysqlserver
docker run --name=slave --hostname=slave -v C:\docker\:/data/backups --network=grnet -p 3337:3306 -itd yzjamm/mysqlserver

```



**Configure my.cnf and startup DB instances**

```powershell
#Master configuration
docker exec -it master bash
#add below lines into /etc/my.cnf
vi /etc/my.cnf
server-id=100
log_bin=/var/lib/mysql/mysql-bin-master.log
binlog-format = ROW
gtid-mode = ON
enforce-gtid-consistency = ON
#initialize database files
mysqld --initialize-insecure=on
#check error log
cat /var/log/mysqld.log
#startup master instance
mysqld &

#Slave configuration
docker exec -it slave bash
#add below lines into /etc/my.cnf
vi /etc/my.cnf
server-id=101
log_bin=/var/lib/mysql/mysql-bin-slave.log
binlog-format = ROW
gtid-mode = ON
enforce-gtid-consistency = ON
log-slave-updates = ON
#initialize database files
mysqld --initialize-insecure=on
#check error log
cat /var/log/mysqld.log
#startup slave instance
mysqld &

#Master/Slave configuration
docker exec -it master mysql
create user 'jack'@'%' identified WITH mysql_native_password by 'jack';
grant replication slave on *.* to 'jack'@'%';
mysql> show master status;
+------------------+----------+--------------+------------------+------------------------------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set                        |
+------------------+----------+--------------+------------------+------------------------------------------+
| mysql-bin.000002 |      680 |              |                  | 853ec1c9-f734-11e8-825e-0242ac120002:1-2 |
+------------------+----------+--------------+------------------+------------------------------------------+
1 row in set (0.00 sec)

docker exec -it slave mysql
mysql> CHANGE MASTER TO MASTER_HOST='xxx.xxx.xxx.xxx',MASTER_PORT=3336, MASTER_USER='jack', MASTER_PASSWORD='jack', MASTER_LOG_FILE='mysql-bin.000002', MASTER_LOG_POS=680;
start slave;
show slave status\G

```



**Run MySQL Enterprise Backup tool to backup slave**

```powershell
docker exec slave mysqlbackup --user=root --password= --backup-dir=/data/backups/slave  --backup-image=jacktest --compress backup-to-image 
......
181210 15:51:33 MAIN    INFO: MySQL binlog position: filename mysql-bin-slave.000002, position 819
181210 15:51:33 MAIN    INFO: GTID_EXECUTED is 1dbc7f04-fc8d-11e8-b80a-0242ac120002:6-8

-------------------------------------------------------------
   Parameters Summary
-------------------------------------------------------------
   Start LSN                  : 19054592
   End LSN                    : 19092683
-------------------------------------------------------------

mysqlbackup completed OK!
```



**Create another MySQL database Container for testing point-in-time recovery**

```powershell
docker run --name=clonetest --hostname=clonetest -v C:\docker\:/data/backups --network=grnet -p 3338:3306 -itd yzjamm/mysqlserver

docker exec -i clonetest mysqlbackup --uncompress --backup-dir=/data/backups/clonetest --backup-image=/data/backups/slave/jacktest --force copy-back-and-apply-log

docker exec -it clonetest bash
#add below lines into /etc/my.cnf
vi /etc/my.cnf
server-id=103
log_bin=/var/lib/mysql/mysql-bin-clonetest.log
binlog-format = ROW
gtid-mode = ON
enforce-gtid-consistency = ON

#change owner of data files under /var/lib/mysql
chown -R mysql:mysql /var/lib/mysql

#startup database instance
mysqld &

```

**Create testing data in master**

```powershell
docker exec -it master mysql
mysql> create database test;
Query OK, 1 row affected (0.01 sec)
mysql> use test;
Database changed
mysql> create table test1 (name varchar(1));
Query OK, 0 rows affected (0.04 sec)
mysql> insert into test1 values('a');
Query OK, 1 row affected (0.01 sec)
```



**Check Slave binary log events and will see the test1 table is there**

```powershell
docker exec -it slave mysql
mysql> show binary logs;
+------------------------+-----------+
| Log_name               | File_size |
+------------------------+-----------+
| mysql-bin-slave.000001 |       178 |
| mysql-bin-slave.000002 |       819 |
+------------------------+-----------+
2 rows in set (0.00 sec)

mysql> show binlog events in 'mysql-bin-slave.000002';
+------------------------+-----+----------------+-----------+-------------+-------------------------------------------------------------------+
| Log_name               | Pos | Event_type     | Server_id | End_log_pos | Info                                                              |
+------------------------+-----+----------------+-----------+-------------+-------------------------------------------------------------------+
| mysql-bin-slave.000002 |   4 | Format_desc    |       101 |         124 | Server ver: 8.0.12-commercial, Binlog ver: 4                      |
| mysql-bin-slave.000002 | 124 | Previous_gtids |       101 |         155 |                                                                   |
| mysql-bin-slave.000002 | 155 | Gtid           |       100 |         235 | SET @@SESSION.GTID_NEXT= '1dbc7f04-fc8d-11e8-b80a-0242ac120002:6' |
| mysql-bin-slave.000002 | 235 | Query          |       100 |         341 | create database test /* xid=9 */                                  |
| mysql-bin-slave.000002 | 341 | Gtid           |       100 |         421 | SET @@SESSION.GTID_NEXT= '1dbc7f04-fc8d-11e8-b80a-0242ac120002:7' |
| mysql-bin-slave.000002 | 421 | Query          |       100 |         543 | use `test`; create table test1 (name varchar(1)) /* xid=10 */     |
| mysql-bin-slave.000002 | 543 | Gtid           |       100 |         625 | SET @@SESSION.GTID_NEXT= '1dbc7f04-fc8d-11e8-b80a-0242ac120002:8' |
| mysql-bin-slave.000002 | 625 | Query          |       100 |         695 | BEGIN                                                             |
| mysql-bin-slave.000002 | 695 | Table_map      |       100 |         750 | table_id: 87 (test.test1)                                         |
| mysql-bin-slave.000002 | 750 | Write_rows     |       100 |         788 | table_id: 87 flags: STMT_END_F                                    |
| mysql-bin-slave.000002 | 788 | Xid            |       100 |         819 | COMMIT /* xid=12 */                                               |
+------------------------+-----+----------------+-----------+-------------+-------------------------------------------------------------------+
11 rows in set (0.00 sec)
```

**Copy binary log file to shared folder**

```pow
[root@slave mysql]# cp /var/lib/mysql/mysql-bin-slave.000002 /data/backups
[root@slave mysql]#
```

**Point-in-time recovery: roll forward the binary log events onto clonetest database, find --start-position from slave backup job log or clonetest restore job log**

```powershell
docker exec -it clonetest bash
mysqlbinlog --start-position=819 /data/backups/mysql-bin-slave.000002  |   mysql -u root
mysql
mysql> select * from test.test1;
+------+
| name |
+------+
| a    |
+------+
1 rows in set (0.00 sec)

mysql>

```



**Below is the chart comparison between Percona XtraBackup and MySQL Enterprise Backup**



![1544110032821](/assets/img/1544110032821.png)
