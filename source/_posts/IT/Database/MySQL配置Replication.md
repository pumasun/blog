---
title: MySQL配置Replication
date: 2022-12-08 15:01:39
tags: ['IT', 'MySQL', 'Database', 'Replication', 'Manual']
description: '在实际生产环境中，为了保障数据的高可用，我们经常需要对数据进行多副本部署。如果你在使用MySQL，那么Replication将帮你实现这个需求。'
keywords: ['生产', 'MySQL', '副本', '高可用', 'Replication', 'MySQL', 'Database', 'DB']
top_img: img/MySQL配置Replication.png
category: ['IT', 'MySQL']
---
# MySQL配置Replication
## 官网
[MySQL 5.7](https://dev.mysql.com/doc/refman/5.7/en/replication.html)

## Master准备
### 1.配置my.cnf
``` SHELL
vim /etc/mysql/my.cnf
```
``` ini
[mysqld]
log-bin=mysql-bin  //[必须]启用二进制日志
server-id=1        //[必须]服务器唯一ID，默认是1，一般取IP最后一段
```
如果Master为另外一个服务器的Slave，则需要配置 `log-slave-updates=ON`

### 2.重启MySQL
### 3.创建Replication账户
``` SQL
mysql> CREATE USER 'USER_NAME'@'*' IDENTIFIED BY 'PASSWORD';
mysql> GRANT REPLICATION SLAVE ON *.* TO 'USER_NAME'@'%';
```
或
``` SQL
mysql> GRANT REPLICATION SLAVE ON *.* to 'USER_NAME'@'%' IDENTIFIED  BY 'PASSWORD';
```
 
### 4.将Master设定为ReadOnly
``` SQL
mysql> FLUSH TABLES WITH READ LOCK;
mysql> SET GLOBAL read_only = ON;
```
 
Replication配置完成后，使用以下语句恢复服务
``` SQL
mysql> SET GLOBAL read_only = OFF;
mysql> UNLOCK TABLES;
```

### 5.创建新的bin-log文件
``` SQL
Mysql>flush logs;
```
 
### 6.备份Master数据库已有数据
热备份全部数据库
``` shell
mysqldump --all-databases > fulldb.dump
```
 
热备份部分数据库
``` shell
mysqldump --databases db1 db2 > partdb.sql
```
 
冷备份全部数据
``` shell
mysqladmin shutdown
tar cf /tmp/db.tar ./data
service mysql start
```
 
### 7.查看Master状态
``` SQL
mysql> SHOW MASTER STATUS\G
```
 
输出示例
``` SQL
+------------------+----------+--------------------------------+
| File             | Position | Binlog_Do_DB | inlog_Ignore_DB |
+------------------+----------+--------------------------------+
| mysql-bin.000004 |      308 |              |                 |
+------------------+----------+--------------------------------+
1 row in set (0.00 sec)
```

`注：执行完此步骤后不要再操作主服务器MYSQL，防止主服务器状态值变化。`



## Slave准备
### 1.配置my.cnf
``` SHELL
vim /etc/mysql/my.cnf
```
``` INI
[mysqld]
log-bin=mysql-bin  //[必须]启用二进制日志
server-id=2        //[必须]服务器唯一ID，默认是1，一般取IP最后一段
```

### 2.导入备份数据
``` SHELL
mysql < fulldb.dump
```
 
或
``` SHELL
cd /var/lib/mysql
tar xvf dbdump.tar
```
### 3.重启MySQL
### 4.配置主从同步
``` SQL
mysql> CHANGE MASTER TO
    ->    MASTER_HOST='127.0.0.1',
    ->    MASTER_USER='USER_NAME',
    ->    MASTER_PASSWORD='PASSWORD',
    ->    MASTER_LOG_FILE='mysql-bin.000004',
    ->    MASTER_LOG_POS=308;
```

### 5.启动同步
``` SQL
mysql> START SLAVE;
```
### 6.检查同步状态
``` SQL
mysql> SHOW SLAVE STATUS\G
```

输出示例
``` SQL
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.245.140
                  Master_User: root
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000003
          Read_Master_Log_Pos: 5669
               Relay_Log_File: mysqld-relay-bin.000002
                Relay_Log_Pos: 5482
        Relay_Master_Log_File: mysql-bin.000003
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 5669
              Relay_Log_Space: 5639
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 140
1 row in set (0.00 sec)
```
`注：Slave_IO及Slave_SQL进程必须正常运行，即YES状态，否则都是错误的状态。`



## Q&A
### 1.同步终止
### 2.创建新的log-file
``` SQL
Mysql>flush logs;
```
 
### 3.如何让副本停止处理请求
可以用mysqladmint完全停止副本上的同步处理
``` SHELL
mysqladmin stop-slave
```
 
也可以接受数据元Bin数据更新但不执行
``` SHELL
mysql -e 'STOP SLAVE SQL_THREAD;'
```
 
可以使用mysqladmin恢复副本同步
``` SHELL
mysqladmin start-slave
```
 
### 4.如何指定备份的DB
#### a.Master
``` INI
Binlog_Do_DB	# 设定哪些数据库需要记录Binlog
Binlog_Ignore_DB	#设定哪些数据库不要记录Binlog
```
### b.Slave
``` INI
Replicate_Do_DB	 # 设定须要复制的数据库，多个DB用逗号分隔
Replicate_Ignore_DB	# 设定可以忽略的数据库
Replicate_Do_Table	# 设定须要复制的Table
Replicate_Ignore_Table	# 设定可以忽略的Table
Replicate_Wild_Do_Table	# 功能同Replicate_Do_Table，但可以带通配符来进行设置
Replicate_Wild_Ignore_Table	#功能同Replicate_Ignore_Table，可带通配符设置
```