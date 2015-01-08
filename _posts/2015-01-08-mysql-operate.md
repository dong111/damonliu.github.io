---
layout: post
category: db
title: MySQL 数据库常用操作
tags: [mysql]
---

* DCL(Data Control Language) 数据库控制语言。
* DDL(Data Definition Language) 数据库定义语言语句。
* DML(Data Manipulation Language) 数据操纵语言。

# 1 数据库有关

### 1.1 查看数据库状态
```mysql
mysql> status
--------------
mysql  Ver 14.14 Distrib 5.5.33, for Linux (x86_64) using readline 5.1

Connection id:          152034
Current database:
Current user:           root@localhost
SSL:                    Not in use
Current pager:          stdout
Using outfile:          ''
Using delimiter:        ;
Server version:         5.5.33 MySQL Community Server (GPL) by Remi
Protocol version:       10
Connection:             Localhost via UNIX socket
Server characterset:    utf8
Db     characterset:    utf8
Client characterset:    utf8
Conn.  characterset:    utf8
UNIX socket:            /var/lib/mysql/mysql.sock
Uptime:                 366 days 50 min 57 sec
```

### 1.2 设置数据的编码
```mysql
[client]
default-character-set=utf8

[mysql]
default-character-set=utf8

[mysqld]
collation-server=utf8_general_ci
init-connect='SET NAMES utf8'
character-set-server=utf8

mysql> ALTER DATABASE `mysql` DEFAULT CHARACTER SET = UTF8;        // 修改一个数据库的编码
```

### 1.3 授权
```mysql
mysql> GRANT ALL ON *.* TO root@'%' IDENTIFIED BY 'root' WITH GRANT OPTION; // 可以在所有的IP地址访问
mysql> GRANT SELECT ON `damon`.`user` TO damon@localhost IDENTIFIED BY 'damon'; 
// damon用户只有在本机登录查询damon用户user表的权限
```

### 1.4 查看权限
```mysql
mysql> SELECT DISTINCT CONCAT('User: ''',user,'''@''',host,''';') AS `grant` FROM `mysql`.`user`; // 查看所有用户访问权限
+---------------------------+
| grant                     |
+---------------------------+
| User: 'root'@'%';         |
| User: 'root'@'localhost'; |
+---------------------------+

mysql> SHOW GRANTS FOR 'root'@'%';   // 查看具体的权限
+------------------------------------------------------------------＋
| GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY PASSWORD  
     '*81F5E21E35407D884A6CD4A731AEBFB6AF209E1B' WITH GRANT OPTION |
+------------------------------------------------------------------+
```

### 1.5 修改数据密码
```mysql
mysql> UPDATE `mysql`.`user` SET `password` = PASSWORD('root') WHERE `user` = 'root';
mysql> FLUSH PRIVILEGES;

mysql> mysqladmin -u `root` -p `root` PASSWORD;

mysql> SET PASSWORD FOR root = PASSWORD('root');
```

# 2 表有关

### 2.1 查看表状态
```mysql
mysql> SHOW TABLE STATUS LIKE 'user' \G
           Name: user
         Engine: MyISAM
        Version: 10
     Row_format: Dynamic
           Rows: 1
 Avg_row_length: 108
    Data_length: 380
Max_data_length: 281474976710655
   Index_length: 2048
      Data_free: 272
 Auto_increment: NULL
    Create_time: 2014-08-08 07:54:17
    Update_time: 2014-08-08 08:06:38
     Check_time: NULL
      Collation: utf8_bin
       Checksum: NULL
 Create_options: 
        Comment: Users and global privileges

mysql> ALTER TABLE `user` engine = InnoDB;                      // 修改表的存储引擎
```


### TODO
