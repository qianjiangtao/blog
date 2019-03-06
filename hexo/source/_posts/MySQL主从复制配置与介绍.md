---
title: mysql主从复制配置与介绍
date: 2019-03-06 09:45:31
categories:
- mysql
tags:
- mysql主从复制
---

![](https://gitee.com/qianjiangtao/my-image/raw/master/blog/2019-03-06-01.jpg)

<!--more-->

### 一，MySQL安装配置

*1.下载 mysql 源安装包* 

```shell
$ curl -LO http://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm
```

*2.安装 mysql 源* 

```shell
$ sudo yum localinstall mysql57-community-release-el7-11.noarch.rpm
```

**注意**：执行过程中如果报错如下



![test](https://gitee.com/qianjiangtao/my-image/raw/master/mysql/20160825153731032)



则通过修改python版本, 修改yum配置文件，将python版本指向以前的旧版本 

```shell
# 修改yum配置文件，将python版本指向以前的旧版本
# vi /usr/bin/yum
#!/usr/bin/python2.7

# 修改urlgrabber-ext-down文件，更改python版本
# vi /usr/libexec/urlgrabber-ext-down
#!/usr/bin/python2.7
```



**检查 yum 源是否安装成功** 

```shell
$ sudo yum repolist enabled|grep "mysql.*-community.*"

mysql-connectors-community/x86_64       MySQL Connectors Community            95
mysql-tools-community/x86_64            MySQL Tools Community                 84
mysql57-community/x86_64                MySQL 5.7 Community Server           327
```

3.*安装server*

```shell
$ sudo yum install mysql-community-server
```



*4.启动mysql服务*

-  安装服务

  ```shell
  $ sudo systemctl enable mysqld
  ```

- 启动服务

  ```shell
  $ sudo systemctl start mysqld
  ```

- 查看服务状态

  ```shell
  $ sudo systemctl status mysqld
  ```

  ```shell
  ● mysqld.service - MySQL Server
     Loaded: loaded (/usr/lib/systemd/system/mysqld.service; enabled; vendor preset: disabled)
     Active: active (running) since 一 2019-03-04 16:36:09 CST; 17s ago
       Docs: man:mysqld(8)
             http://dev.mysql.com/doc/refman/en/using-systemd.html
    Process: 3126 ExecStart=/usr/sbin/mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid $MYSQLD_OPTS (code=exited, status=0/SUCCESS)
    Process: 3053 ExecStartPre=/usr/bin/mysqld_pre_systemd (code=exited, status=0/SUCCESS)
   Main PID: 3129 (mysqld)
      Tasks: 27
     CGroup: /system.slice/mysqld.service
             └─3129 /usr/sbin/mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid
  3月 04 16:36:02 lvmama02 systemd[1]: Starting MySQL Server...
  3月 04 16:36:09 lvmama02 systemd[1]: Started MySQL Server.
  ```

*5.修改默认密码*

- `MySQL 5.7 `启动后，在 `/var/log/mysqld.log` 文件中给 root 生成了一个默认密码。通过下面的方式找到 root 默认密码，然后登录 mysql 进行修改

  ```shell
  $ grep 'password' /var/log/mysqld.log 
  2019-03-04T08:36:04.854935Z 1 [Note] A temporary password is generated for root@localhost: pdU7wuS/YhSG
  ```

- 登录mysql修改密码

  ```shell
  #注意：MySQL 5.7 默认安装了密码安全检查插件（validate_password），默认密码检查策略要求密码必须包
  #含：大小写字母、数字和特殊符号，并且长度不能少于 8 位。
  $ ALTER USER 'root'@'localhost' IDENTIFIED BY 'xxxxxx' 
  ```

**注意：**

​	测试环境我们可以设置禁用密码校验，毕竟密码太难记了,可以通过如下方式禁用:

```shell
$ sudo vi /etc/my.cnf
# 添加如下配置
# 禁用密码校验策略
validate_password = off

# 重新启动mysql 重新修改密码
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'root'
```



*6.添加远程登录用户*

​	MySQL 默认只允许 root 帐户在本地登录，如果要在其它机器上连接 MySQL，必须修改 root 允许远程连接，或者添加一个允许远程连接的帐户，为了安全起见，本例添加一个新的帐户：

```mysql
mysql> GRANT ALL PRIVILEGES ON *.* TO 'admin'@'%' IDENTIFIED BY 'admin' WITH GRANT OPTION;
Query OK, 0 rows affected, 1 warning (0.00 sec) 
```



*7.配置默认编码为 utf8* 

- MySQL 默认为 latin1, 一般修改为 UTF-8 

  ```shell
  $ vi /etc/my.cnf
  [mysqld]
  # 在myslqd下添加如下键值对
  character_set_server=utf8
  init_connect='SET NAMES utf8'
  ```

  

- 重启测试查看字符集

  ```mysql
  mysql> show variables like 'character%'
  +--------------------------+----------------------------+
  | Variable_name            | Value                      |
  +--------------------------+----------------------------+
  | character_set_client     | utf8                       |
  | character_set_connection | utf8                       |
  | character_set_database   | utf8                       |
  | character_set_filesystem | binary                     |
  | character_set_results    | utf8                       |
  | character_set_server     | utf8                       |
  | character_set_system     | utf8                       |
  | character_sets_dir       | /usr/share/mysql/charsets/ |
  +--------------------------+----------------------------+
  ```

*8.开启端口 (如果防火墙没关闭则需要做如下操作)*

```shell
$ sudo firewall-cmd --zone=public --add-port=3306/tcp --permanent
FirewallD is not running
$ sudo firewall-cmd --reload
FirewallD is not running
```



### 二，MySQL主从配置



- **配置主库**

  1，主库 mysql11 上开启设置

  ```shell
  # 添加如下配置
  # 参数必须唯一, 本例主库设置为 11 ，从库设置为 12
  server_id=101
  log_bin=/var/log/mysql/mysql-bin
  ```

  2，二进制日志文件的目录不是默认的，需要新建一下 

  ```shell
  # 创建文件夹
  $ sudo mkdir /var/log/mysql
  # 分配权限
  $ sudo chown mysql:mysql /var/log/mysql
  ```

  3，重启主库的 MySQL 服务 

  ```shell
  $ sudo systemctl restart mysqld
  ```

  4，测试

  ```mysql
  mysql> show master status
  +------------------+----------+--------------+------------------+-------------------+
  | File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
  +------------------+----------+--------------+------------------+-------------------+
  | mysql-bin.000001 |      154 |              |                  |                   |
  +------------------+----------+--------------+------------------+-------------------+
  ```

  从结果看到， File 字段有值，并且前面与配置文件一致，说明配置正确。后面的 `000001` 说明是第一次，如果 MySQL 重启服务，这个值会递增为 `mysql-bin.000002` 

  

- **配置从库**

  1，从库 mysql11 上配置 

  ```shell
  # 从库配置
  server_id=12
  log_bin=/var/log/mysql/mysql-bin.log
  relay_log=/var/log/mysql/mysql-relay-bin.log
  #库设为只读的
  read_only=1
  ```

  2，从库设置的二进制日志文件的目录不是默认的，需要新建一下 

  ```shell
  $ sudo mkdir /var/log/mysql
  # 分配权限
  $ sudo chown mysql:mysql /var/log/mysql
  ```

  3，重启从库的 MySQL 服务 

  ```shell
  $ sudo systemctl restart mysqld
  ```

  4，设置从库的复制参数 

  ```mysql
  mysql> CHANGE MASTER TO MASTER_HOST='192.168.187.11',
      -> MASTER_USER='admin',
      -> MASTER_PASSWORD='admin',
      #此选项初始化设置时需要跟主库中的一致。设置好后，如果主
      #库发生重启等，不需再次设置，从库会跟着更新
      -> MASTER_LOG_FILE='mysql-bin.000001',
      # master Position的值
      -> MASTER_LOG_POS=154; 
  ```

  5.查看从库状态

  ```mysql
  mysql> show slave status \G
  *************************** 1. row ***************************
                 Slave_IO_State: 
                    Master_Host: 192.168.187.11
                    Master_User: admin
                    Master_Port: 3306
                  Connect_Retry: 60
                Master_Log_File: mysql-bin.000001
            Read_Master_Log_Pos: 154
                 Relay_Log_File: mysql-relay-bin.000001
                  Relay_Log_Pos: 4
          Relay_Master_Log_File: mysql-bin.000001
               Slave_IO_Running: No
              Slave_SQL_Running: No
  *************************** 略 ***************************
  ```

  6，从 `Slave_IO_State`, `Slave_IO_Running: No`, `Slave_SQL_Running: No` 表明当前从库的复制服务还没有启动,启动从库

  ```mysql
  mysql> start slave;
  Query OK, 0 rows affected (0.03 sec)
  ```

  再次查看`show slave status \G`

  ```mysql
  *************************** 1. row ***************************
                 Slave_IO_State: Waiting for master to send event
                    Master_Host: 192.168.187.11
                    Master_User: admin
                    Master_Port: 3306
                  Connect_Retry: 60
                Master_Log_File: mysql-bin.000001
            Read_Master_Log_Pos: 154
                 Relay_Log_File: mysql-relay-bin.000002
                  Relay_Log_Pos: 320
          Relay_Master_Log_File: mysql-bin.000001
               Slave_IO_Running: Yes
              Slave_SQL_Running: Yes
  *************************** 略 ***************************
  ```

  7，测试在主库中新建一个库，查看从库是否同步复制主库数据

  ```mysql
  # ---------mater--------
  mysql> show databases;
  +--------------------+
  | Database           |
  +--------------------+
  | information_schema |
  | mysql              |
  | performance_schema |
  | sys                |
  +--------------------+
  4 rows in set (0.02 sec)
  
  mysql> create database test;
  Query OK, 1 row affected (0.02 sec)
  
  # ---------slave--------
  mysql> show databases;
  +--------------------+
  | Database           |
  +--------------------+
  | information_schema |
  | mysql              |
  | performance_schema |
  | sys                |
  | test               |
  +--------------------+
  5 rows in set (0.01 sec)
  ```



### 三，MySQL主从配置原理



#### 1.`mysql`支持的复制格式

- 基于语句复制(**STATEMENT**)

  - （**优点**）基于statement复制的优点很明显，简单的记录执行语句同步到从库执行同样的语句，占用磁盘空间小，网络传输快，并且通过`mysqlbinlog`工具容易读懂其中的内容 。
  - （**缺点**）并不是所有语句都能复制的比如：`insert into table1(create_time) values(now())`，取的是数据当前时间，不同的数据可能时间不一致，另外像存储过程和触发器也可能存在问题。

- 基于行复制(**ROW**）

  - （**优点**）`从MySQL5.1开始支持基于行的复制 `，最大的好处是可以正确地复制每一行数据。一些语句可以被更加有效地复制，另外就是几乎没有基于行的复制模式无法处理的场景，对于所有的`SQL`构造、触发器、存储过程等都能正确执行。
  - （**缺点**）主要的缺点就是二进制日志可能会很大，比如：`update table1 set name='admin' where id<1000，`基于行复制可能需要复制1000条记录，而基于语句复制只有一条语句，另外一个缺点就是不直观，所以，你不能使用`mysqlbinlog`来查看二进制日志。 

- 混合类型的复制(**MIXED**)

  混合复制是借用语句复制和行复制的有点进行整合，MIXED也是`MySQL`默认使用的二进制日志记录方式，但MIXED格式默认采用基于语句的复制，一旦发现基于语句的无法精确的复制时，就会采用基于行的复制。比如用到`UUID()`、`USER()`、`CURRENT_USER()`、`ROW_COUNT()`等无法确定的函数。 

  

#### 2.mysql主从复制作用

-  数据分布。
-  主从分摊负载。
-  高可用性和故障切换。
-  数据备份。
-  利用从服务器做查询。



#### 3.mysql主从复制原理

- **binlog Events**

​	我们知道`binlog`日志用于记录所有对`MySQL`的操作的变更，而这每一个变更都会对应的事件，也就是Event。index文件记录了所有的`binlog`位置 每个`binlog`会有`heade， event，rotate`三个event，`binlog`的结构如下。 

![1551783742620](https://gitee.com/qianjiangtao/my-image/raw/master/mysql/1551783742620.png)



- 常见event如下：

  - Format_desc：一个全新的binlog日志文件event信息

  - Rotate ：日志分割时结束event。

  - Table_map：表，列等元数据的event。

  - Query：查询，就是DDL这类的Event，如果binlog格式为STATEMENT格式，增删改都属于Qeury event。

  - Write_rows：Binlog为ROW格式时的插入event。

  - Update_rows：Binlog为ROW格式时的更新event。

  - Delete_rows：Binlog为ROW格式时的删除event。

    

**我们也可以通过`binlog` 看到这些事件，通过mysql提供的工具查看binlog日志，如下:**

![1551785276014](https://gitee.com/qianjiangtao/my-image/raw/master/mysql/1551785276014.png)



- **主从复制流程**

  ![1551836047352](https://gitee.com/qianjiangtao/my-image/raw/master/mysql/1551836047352.png)

  - 当从库发出 `start slave`命令时，从库会创建I/O线程和`SQL thread`（SQL线程）
  - 从库的IO和主库的dump线程建立连接 并监听`binlog`二进制日志事件
  - 从库根据change master to 语句提供的file名和position号，IO线程向主库发起`binlog`的请求 
  - 主库dump线程根据从库的请求，将本地`binlog`以events的方式发给从库IO线程 
  - 从库IO线程接收`binlog evnets`，并存放到本地relay-log中，传送过来的信息，会记录到master.info中。 
  - 从库SQL线程应用relay-log，并且把应用过的记录到relay-log.info,默认情况下，已经应用过的relay会自动被清理purge。 



### 四，MySQL只从配置缺陷

​	

​	MySQL的复制（replication）功能配置简单，深受开发人员的喜欢，基于复制的读写分离方案也非常流行。而MySQL数据库高可用大多也是基于复制技术，但是MySQL复制本身依然存在部分缺陷，最为主要的问题如下： 

- 数据丢失问题（consistency）
- 数据同步延迟问题（delay）
- 扩展性问题（scalability）



> 从MySQL 5.7的lossless semi-sync replication已经解决了主从数据丢失的问题，MySQL 5.7的multi-thread slave也很大程度地解决了数据同步延迟的问题，MySQL 5.7的Group replication也很大程度地解决了扩展性问题。另外，MySQL 5.7.22 backlog了MySQL 8.0中的基于WriteSet的并行复制，可以说完全解决了主从数据延迟的问题。可以看出，MySQL正在朝着一个非常好的方向发展 