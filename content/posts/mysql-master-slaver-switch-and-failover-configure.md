---
title: "MySQL主从复制及FailOver切换配置"
date: 2021-10-08T00:35:07+08:00
draft: false
tags: ["MySQL"]
categories: ["MySQL","DB"]
---


### MySQL主从复制原理

  master服务器将数据的改变都记录到二进制binlog日志中，只要master上的数据发生改变，则将其改变写入二进制日志；salve服务器会在一定时间间隔内对master二进制日志进行探测其是否发生改变，如果发生改变，则开始一个I/O Thread请求master二进制事件，同时主节点为每个I/O线程启动一个dump线程，用于向其发送二进制事件，并保存至从节点本地的中继日志中，从节点将启动SQL线程从中继日志中读取二进制日志，在本地重放，使得其数据和主节点的保持一致，最后I/O Thread和SQL Thread将进入睡眠状态，等待下一次被唤醒。

具体原理参考[MySQL 主从同步(1) - 概念和原理介绍 以及 主从/主主模式 部署记录](https://www.cnblogs.com/kevingrace/p/6256603.html)

### MySQL主从复制配置

MySQL服务器信息：

主库: 系统 Win10 20H2,数据库MySQL 8.0.21,地址 172.23.64.1:3306

从库: 系统 Ubuntu20.04.1 LTS,数据库MySQL 8.0.21,地址172.23.79.79:3306

同步的数据库名 test
保证系统间端口可以互相访问

#### MySQL主库配置

修改MySQL服务配置文件my.ini
win10下目录为‪C:\ProgramData\MySQL\MySQL Server 8.0\my.ini

``` shell
# 服务器ID，需要保证每个服务器的ID唯一
server-id=1

# 需要同步的数据库名，此处为test
binlog-do-db=test

# 需要忽略的数据库名，此处为mysql
binlog-ignore-db=mysql

# 指定binlog二进制日志同步策略
# sync_binlog=0，当事务提交之后，MySQL不做fsync之类的磁盘同步指令刷新binlog_cache中的信息到磁盘，而让Filesystem自行决定什么时候来做同步，或者cache满了之后才同步到磁盘。
# sync_binlog=n，当每进行n次事务提交之后，MySQL将进行一次fsync之类的磁盘同步指令来将binlog_cache中的数据强制写入磁盘。
sync_binlog = 1


# 跳过现有的采用checksum的事件，此处需要主从服务器保持一致
binlog_checksum = none

# bin-log日志文件格式，设置为MIXED可以防止主键重复
binlog_format = mixed
```

重启MySQL服务后，执行`show master status;`结果如下:

``` SQL
+----------------------------+----------+--------------+------------------+-------------------+
| File                       | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+----------------------------+----------+--------------+------------------+-------------------+
| DESKTOP-NEON79U-bin.000496 |      156 |              |                  |                   |
+----------------------------+----------+--------------+------------------+-------------------+
```

#### MySQL从库配置

首先同步主库表结构及数据至从库

修改MySQL服务配置文件
Ubuntu20.04中目录为:/etc/mysql/mysql.conf.d/mysqld.cnf

``` shell
# 服务器ID，保证唯一
server-id=2

# 要复制的数据库名
replicate-do-db=test

# 要忽略的数据库名(其实上一个参数已经指定要复制的数据库名了，这一条可以不要)
replicate-ignore-db=mysql

# 忽略所有错误
slave-skip-errors = all

# 跳过现有的采用checksum的事件，此处需要主从服务器保持一致
slave_sql_verify_checksum = none
```

重启服务后，进入MySQL命令行

`stop slave;` 关闭slave


`change  master to master_host='172.23.64.1',master_user='user'master_password='xxxxx';`

配置主服务器信息
master_host为主服务器地址
master_user为可远程登陆的用户名
master_password为密码

在其他教程里，还指定了master_log_file为binglog文件名，就是主机执行`show master status;`得到的File值，如上文:DESKTOP-NEON79U-bin.000496

以及master_log_pos，主机执行`show master status;`得到的Position 值，如上文:156
在我的尝试中，不输入这两个值，子系统会自动配置，指定后反而会导致Last_IO_Error: Relay log write failure: could not queue event from master的错误

`start slave;`重启slave

之后可以输入`show slave status\G;`来查看结果

```
mysql> show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for source to send event
                  Master_Host: 172.23.64.1
                  Master_User: byco
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: DESKTOP-NEON79U-bin.000496
          Read_Master_Log_Pos: 156
               Relay_Log_File: ubuntubyco-relay-bin.000027
                Relay_Log_Pos: 391
        Relay_Master_Log_File: DESKTOP-NEON79U-bin.000496
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: test
          Replicate_Ignore_DB: mysql
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 156
              Relay_Log_Space: 840
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
             Master_Server_Id: 1
                  Master_UUID: 288897e6-dd45-11ea-86f8-00ff6eb77a84
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Replica has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set:
            Executed_Gtid_Set:
                Auto_Position: 0
         Replicate_Rewrite_DB:
                 Channel_Name:
           Master_TLS_Version:
       Master_public_key_path:
        Get_master_public_key: 0
            Network_Namespace:
1 row in set, 1 warning (0.00 sec)
```

其中
Slave_IO_Running和Slave_SQL_Running值为Yes则说明主从复制已经生效

如果Slave_IO_Running=connecting，则连接出错，有可能是ip地址，防火墙的原因
Slave_IO_Running=no，我遇到的原因包括mysql配置文件中slave_sql_verify_checksum不一致，以及在子系统指定主系统链接信息时指定了master_log_file以及master_log_pos。

关于报错原因可以从上文的以下几项看到：
Last_IO_Errno
Last_IO_Error
Last_SQL_Errno
Last_SQL_Error

以及可以查询mysql配置文件下log_error配置项对应的error文件来排查问题

至此MySQL主从复制基本已经可以使用了

### FailOver切换配置

参考文档[Configuring Server Failover for Connections Using JDBC](https://dev.mysql.com/doc/connector-j/8.0/en/connector-j-config-failover.html)

通过配置多个数据源，当jdbc无法连接第一个数据源时，会切换到第二个数据源

``` url
jdbc:mysql://[primary host][:port],[secondary host 1][:port][,[secondary host 2][:port]]...[/[database]]»
[?propertyName1=propertyValue1[&propertyName2=propertyValue2]...]
```

本文使用的示例JDBC-URL为

``` url
jdbc:mysql://172.23.64.1:3306,172.23.79.79:3306/test?useUnicode=true&characterEncoding=UTF8&rewriteBatchedStatements=true&autoReconnect=true&serverTimezone=UTC&useSSL=false&failOverReadOnly=false
```

其中与FailOver切换相关的配置项有failOverReadOnly=false
当该值为false时，切换后会将新的数据源视为可读写数据库，
若为true，则只能是只读数据库

### 参考文档

[MySQL 主从同步(1) - 概念和原理介绍 以及 主从/主主模式 部署记录](https://www.cnblogs.com/kevingrace/p/6256603.html)

[MySQL主从同步（复制）](https://www.cnblogs.com/kylinlin/p/5258719.html)

[Linux环境下MySQL开启远程访问权限](https://cloud.tencent.com/developer/article/1640595)

[MySQL8.0允许远程登陆](https://www.jianshu.com/p/435307de1c29)

[MySQL :: MySQL 8.0 Reference Manual :: 17 Replication](https://dev.mysql.com/doc/refman/8.0/en/replication.html)

[9.1 Configuring Server Failover for Connections Using JDBC](https://dev.mysql.com/doc/connector-j/8.0/en/connector-j-config-failover.html)