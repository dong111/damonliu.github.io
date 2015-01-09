---
layout: post
category: db
title: MySQL 数据库常用操作和配置
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

mysql> alter database `mysql` default character set = UTF8;        -- 修改一个数据库的编码
```

### 1.3 授权
```mysql
mysql> grant all on *.* to root@'%' identified by 'root' with grant option; -- 可以在所有的IP地址访问
mysql> grant select on `damon`.`user` TO damon@localhost identified by 'damon'; 
-- damon用户只有在本机登录查询damon数据库user表的权限
```

### 1.4 查看权限
```mysql
mysql> select distinct concat('User: ''',user,'''@''',host,''';') as `grant` from `mysql`.`user`;
 -- 查看所有用户访问权限
+---------------------------+
| grant                     |
+---------------------------+
| User: 'root'@'%';         |
| User: 'root'@'localhost'; |
+---------------------------+

mysql> show grants for 'root'@'%';   -- 查看具体的权限
+------------------------------------------------------------------＋
| GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY PASSWORD  
     '*81F5E21E35407D884A6CD4A731AEBFB6AF209E1B' WITH GRANT OPTION |
+------------------------------------------------------------------+
```

### 1.5 修改数据密码
```mysql
mysql> update `mysql`.`user` set `password` = PASSWORD('root') where `user` = 'root';
mysql> flush privileges;

mysql> mysqladmin -uroot -p 'root' PASSWORD;

mysql> set password for root = PASSWORD('root');
```

### 1.6 导入导出
```mysql
shell> mysqldump -uroot -proot `mysql` > /data/mysql.sql              -- 导出mysql整个数据库
shell> mysqldump -uroot -proot `mysql` `user` > /data/mysql_user.sql  -- 导出mysql数据库的user表
-- 常用参数 -d 删除数据，只导出结构 -t 只导出数据 --add-drop-table 添加删除表的语句

mysql> select * into outfile '/data/1.txt' from `user` where `name` = 'root'; -- 导出到文件

mysql> source mysql.sql                                               -- 导入数据
mysql> load data local infile '/data/1.txt' into table `user` fields terminated by '\t';
-- 导入文件数据 字段的分隔符为 '\t' 一个制表符  注意文件里面的分割也要是制表符
```

对于MyISAM存储引擎的表，可以通过以下方式快速的导入大量的数据：

```mysql
mysql> alter table `user` disable keys;
mysql> loading the data
mysql> alter table `user` enable keys
```
 DISABLE KEYS 和 ENABLE KEYS 用来打开或关闭MyISAM表非唯一索引的更新，可以提高速度，**<span class="text-danger">注意：对InnoDB表无效。</span>**

Innodb存储引擎，导入大数据：

* 按照主键顺序导入
* 关闭唯一索引                    set unique_checks = 0;
* 关闭自动提交                    set autocommit = 0;

导入完成之后不要忘记设置回来。



# 2 表有关

### 2.1 查看表状态
```mysql
mysql> show table status like 'user' \G
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
```


### 2.2 表复制
```mysql
mysql> create table `user3` select * from `user`;       -- 复制表结构和数据，但是没有复制 primary key 等

mysql> create table `user1` like `user`;
mysql> create table `user2` select * from `user` where 1 = 2
-- 这两个都是复制旧表的结构，不添加数据，会把主键设置也拷贝过来

mysql> insert into `user1` select * from `user`;        -- 拷贝数据，表结构要一样
mysql> insert into user1(`name`, `password`) select `name`, `password` from user; -- 表结构不一样

mysql> show create table `user` \G;                     -- 可以查看见表语句，拷贝出来创建新表
```

### 2.3 表修改
```mysql
mysql> alter table `user` engine = InnoDB;                                              -- 修改表的存储引擎
mysql> alter table `user` change clomun `id` `user_id` int(11) not null auto_increment, drop primary key,
    -> add primary key using btree(`user_id`);                     -- 修改列名称，主键的话删除索引重新建立
mysql> alter table `user` modify column `id` int(11) not null auto_increment;           -- 修改列属性
mysql> alter table `user` drop column `password`;                                       -- 删除列
mysql> alter table `users` add column `password` varchar(65) not null after `name`;     -- 添加新列
```

### 2.4 索引
```mysql
mysql> show index from `user`;                                       -- 查看索引
mysql> show keys from `user`;

mysql> alter table `user` add index index_id (`id`);                 -- 添加普通索引
mysql> create index index_id on `user` (`id`);
mysql> alter table `user` add unique (`email`);                      -- 添加唯一索引
mysql> create unique index index_email on `user` (`email`);
mysql> alter table `user` add primary key (`user`, `email`);         -- 添加 primary key 索引

mysql> drop index index_id on `user`;                                -- 删除索引
mysql> alter table `user` drop index `index_email';
mysql> alter table `user` drop primary key;

```

### 2.5 查看表的大小
表的一些信息存储在mysql的内置数据库information_schema的TABLES表中。

```mysql
mysql> select TABLE_NAME, DATA_LENGTH+INDEX_LENGTH, TABLE_ROWS 
    -> from TABLES where TABLE_SCHEMA='mysql' and TABLE_NAME = 'user'       -- 通过数据的大小加上索引的大小

mysql> select concat(round(sum(DATA_LENGTH/1024/1024), 2),'MB') as data from TABLES; -- 查看所有数据的大小

mysql> select concat(round(sum(DATA_LENGTH/1024/1024), 2),'MB') as data
    -> from TABLES where TABLE_SCHEMA = 'mysql';                            -- 查看mysql数据库的大小

mysql> selectconcat(round(sum(DATA_LENGTH/1024/1024),2),'MB') as data
    -> from TABLES wheretable_schema = 'mysql'
    -> and table_name = 'user';                                             -- 查看指定数据库的表的大小   
```


# 3 其它

MySQL在Linux下数据库名、表名、列名、别名大小写规则是这样的：
  
* 数据库名与表名是严格区分大小写的；
* 表的别名是严格区分大小写的；
* 列名与列的别名在所有的情况下均是忽略大小写的；
* 变量名也是严格区分大小写的；

MySQL在Windows下是不区分大小写的:

```mysql
lower_case_table_names = 0  -- 0 区分大小写，存储的是实际值 1 不区分大小写，存储的是小写 2 不区分大小写，存储的是实际值
```


* 第一范式：数据库表的每一列都是不可分割的原子数据项
* 第二范式：要求实体的属性完全依赖于主关键字
* 第三范式：任何非主属性不依赖于其它非主属性



