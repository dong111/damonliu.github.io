---
layout: post
category: db
title: MySQL 性能优化 - 参数配置
tags: [mysql]
---

##### 以下配置全部在MySQL5.5版本上进行
```mysql
show variables                          -- 查看配置的信息
show global status                      -- 查看当前字段的状态
set max_connections = 300               -- 修改配置信息
```

### 1 慢查询

##### 1.1 启用MySQL慢查询日志
```mysql
[mysqld]
slow_query_log = 1                                              -- 1/ON 开启 0/OFF 关闭
slow_query_log_file = /data/mysql/mysql-slow-queries。log       -- 日志文件
long_query_time = 5                                             -- 查询时间超过多少秒
log_queries_not_using_indexes                                   -- 查询未使用索引 不是必须是未使用索引
```

##### 1.2 查看慢查询配置
```mysql
mysql> show variables like '%slow%';
+---------------------------+--------------------------------------+
| Variable_name             | Value                                |
+---------------------------+--------------------------------------+
| log_slow_admin_statements | OFF                                  |    -- 主库状态
| log_slow_slave_statements | OFF                                  |    -- 从库状态
| slow_launch_time          | 2                                    |    -- 创建一个查询线程的花费时间秒
| slow_query_log            | ON                                   |    
| slow_query_log_file       | /data/mysql/mysql-slow-queries。log   |
+---------------------------+--------------------------------------+

mysql> show global status like '%slow%';
+---------------------+--------+
| Variable_name       | Value  |
+---------------------+--------+
| Slow_launch_threads | 13     |            -- 超过slow_launch_time时间创建的线程数
| Slow_queries        | 285697 |            -- 慢查询数量
+---------------------+--------+
```
MySQL有自带的命令mysqldumpslow可进行查询慢查询日志。

### 2 连接数

经常会遇见 "MySQL: ERROR 1040: Too many connections" 的情况，一种是访问量确实很高，MySQL服务器抗不住，这个时候就要考虑增加从服务器分散读压力，另外一种情况是MySQL配置文件中max_connections值过小。

```mysql
mysql> show variables like 'max_connections%';
+-----------------+-------+
| Variable_name   | Value |
+-----------------+-------+
| max_connections | 1000  |
+-----------------+-------+
```

MySQL服务器最大连接数是默认值151，可选范围(1-100000)，然后查询一下服务器响应的最大连接数:

```mysql
mysql> show global status like 'Max_used_connections';
+----------------------+-------+
| Variable_name        | Value |
+----------------------+-------+
| Max_used_connections | 204   |
+----------------------+-------+
```

理想的设置是:
Max\_used\_connections / max\_connections * 100% ≈ 85%
最大连接数占上限连接数的85％左右，如果发现比例在10%以下，MySQL服务器连接数上限设置的过高了。

### 3 查询缓存
MySQL没有shared\_pool缓存执行计划，但是提供了query cache缓存sql执行结果和文本，如果在生命周期内完全相同的sql再次运行，则连sql解析都免去了，直接返回查询缓存结果。MyISAM和InnoDB分别引入了key cache和buffer pool。

```mysql
mysql> show variables like '%query_cache%';
+------------------------------+----------+
| Variable_name                | Value    |
+------------------------------+----------+
| have_query_cache             | YES      |
| query_cache_limit            | 1048576  |
| query_cache_min_res_unit     | 4096     |
| query_cache_size             | 33554432 |
| query_cache_type             | ON       |
| query_cache_wlock_invalidate | OFF      |
+------------------------------+----------+
```
query\_cache\_limit：                超过此大小的查询将不缓存。<br/>
query\_cache\_min\_res\_unit：       缓存块的最小大小。<br/>
query\_cache\_size：                 查询缓存大小。<br/>
query\_cache\_type：                 缓存类型，决定缓存什么样的查询，示例中表示不缓存 select sql_no_cache 查询。<br/>
query\_cache\_wlock\_invalidate：    当有其他客户端正在对MyISAM表进行写操作时，如果查询在query cache中，是否返回cache结果还是等写操作完成再读表获取结果。<br/>
query\_cache\_min\_res\_unit:        的配置是一柄”双刃剑”，默认是4KB，设置值大对大数据查询有好处，但如果你的查询都是小数据查询，就容易造成内存碎片和浪费。<br/>

```mysql
mysql> show global status like 'qcache%';
+-------------------------+----------+
| Variable_name           | Value    |
+-------------------------+----------+
| Qcache_free_blocks      | 4505     |
| Qcache_free_memory      | 15813888 |
| Qcache_hits             | 3674248  |
| Qcache_inserts          | 6271377  |
| Qcache_lowmem_prunes    | 930565   |
| Qcache_not_cached       | 84576581 |
| Qcache_queries_in_cache | 10031    |
| Qcache_total_blocks     | 24583    |
+-------------------------+----------+
```
Qcache\_free\_blocks：缓存中相邻内存块的个数，数目大说明可能有碎片，FLUSH QUERY CACHE会对缓存中的碎片进行整理，从而得到一个空闲块。<br/>
Qcache\_free\_memory：缓存中的空闲内存。<br/>
Qcache\_hits：        每次查询在缓存中命中时就增大。<br/>
Qcache\_inserts：     每次插入一个查询时就增大。命中次数除以插入次数就是不中比率。<br/>
Qcache\_lowmem\_prunes：缓存出现内存不足并且必须要进行清理以便为更多查询提供空间的次数。这个数字最好长时间来看；如果这个数字在不断增长，就表示可能碎片非常严重，或者内存 很少。（上面的 free\_blocks和free\_memory可以告诉您属于哪种情况）。<br/>
Qcache\_not\_cached： 不适合进行缓存的查询的数量，通常是由于这些查询不是 SELECT 语句或者用了now()之类的函数。<br/>
Qcache\_queries\_in\_cache：当前缓存的查询（和响应）的数量。<br/>
Qcache\_total\_blocks：缓存中块的数量。<br/>

查询缓存碎片率 = Qcache\_free\_blocks / Qcache\_total\_blocks * 100%<br/>
如果查询缓存碎片率超过20%，可以用FLUSH QUERY CACHE整理缓存碎片，或者试试减小query\_cache\_min\_res\_unit，如果你的查询都是小数据量的话。<br/>
查询缓存利用率 = (query\_cache\_size - Qcache\_free\_memory) / query\_cache\_size * 100%<br/>
查询缓存利用率在25%以下的话说明query\_cache\_size设置的过大，可适当减小；查询缓存利用率在80％以上而且Qcache\_lowmem\_prunes > 50的话说明query\_cache\_size可能有点小，要不就是碎片太多。<br/>
查询缓存命中率 = (Qcache\_hits - Qcache\_inserts) / Qcache\_hits * 100%<br/>


### 4 索引关键字缓存

##### 4.1 MyISAM 引擎
key\_buffer\_size是对MyISAM表性能影响最大的一个参数，下面一台以MyISAM为主要存储引擎服务器的配置:

```mysql
mysql> show variables like 'key_buffer_size';
+-----------------+---------+
| Variable_name   | Value   |
+-----------------+---------+
| key_buffer_size | 8388608 |           -- 索引的缓存大小 bytes
+-----------------+---------+
```

分配了8MB内存给key\_buffer\_size，我们再看一下key\_buffer\_size的使用情况:

```mysql
mysql> show global status like 'Key_read%';
+-------------------+----------+
| Variable_name     | Value    |
+-------------------+----------+
| Key_read_requests | 50000    |
| Key_reads         | 5        |
+-------------------+----------+
2 rows in set (0。09 sec)
```

一共有50000个索引读取请求，有5个请求在内存中没有找到直接从硬盘读取索引，计算索引未命中缓存的概率：
key\_cache\_miss\_rate ＝ Key\_reads / Key\_read\_requests * 100%
比如上面的数据，key\_cache\_miss\_rate为0.01%，10000个索引读取请求才有一个直接读硬盘，已经很BT了，key\_cache\_miss\_rate在0.1%以下都很好（每1000个请求有一个直接读硬盘），如果key\_cache\_miss\_rate在0.01%以下的话，key\_buffer\_size分配的过多，可以适当减少。

MySQL服务器还提供了Key\_blocks\_*参数：

```mysql
mysql> show global status like 'key_blocks_u%';
+-------------------+--------+
| Variable_name     | Value  |
+-------------------+--------+
| Key_blocks_unused | 319663 |
| Key_blocks_used   | 37906  |
+-------------------+--------+
```

Key\_blocks\_unused 表示未使用的缓存簇(blocks)数，Key\_blocks\_used表示曾经用到的最大的blocks数，理想的设置：Key\_blocks\_used / (Key\_blocks\_unused + Key\_blocks\_used) * 100% ≈ 80%

MyISAM 可以创建多个key\_buffer，并指定索引使用的key\_buffer

```mysql
mysql> key buffer -set global hot_cache.key_buffer_size=2*1024*1024*1024 
mysql> key buffer -cache index index_name in hot_cache
```

也可以在数据库启动时提前加载缓存，也可以通过配置文件自动把索引映射到key cache

```mysql
key_buffer_size = 4G
hot_cache.key_buffer_size = 2G
cold_cache.key_buffer_size = 2G
init_file=/data/mysqld_init.sql
mysqld_init.sql内容如下
CACHE INDEX db1.t1, db1.t2, db2.t3 IN hot_cache
CACHE INDEX db1.t4, db2.t5, db2.t6 IN cold_cache
```


##### 4.2 InnoDB 引擎
```mysql
mysql> show variables like '%innodb_buffer%';
+------------------------------+-----------+
| Variable_name                | Value     |
+------------------------------+-----------+
| innodb_buffer_pool_instances | 1         |        -- 子buffer pool数量，buffer pool至少为1G时才能生效
| innodb_buffer_pool_size      | 134217728 |        -- buffer pool大小 bytes
+------------------------------+-----------+

mysql> show global status like '%innodb_buffer_pool%';
+---------------------------------------+-------------+
| Variable_name                         | Value       |
+---------------------------------------+-------------+
| Innodb_buffer_pool_pages_data         | 7944        | -- 当前buffer pool缓存的数据大小，包括脏数据
| Innodb_buffer_pool_bytes_data         | 130154496   | -- 当前buffer pool缓存的数据大小，包括脏数据 
| Innodb_buffer_pool_pages_dirty        | 42          | -- 缓存脏数据页数量
| Innodb_buffer_pool_bytes_dirty        | 688128      | -- 缓存的脏数据大小
| Innodb_buffer_pool_pages_flushed      | 75134914    | -- 刷新页请求数量
| Innodb_buffer_pool_pages_free         | 0           | -- 空闲页数量
| Innodb_buffer_pool_pages_total        | 8191        | -- 缓存池总页数
| Innodb_buffer_pool_read_ahead_rnd     | 0           |
| Innodb_buffer_pool_read_ahead         | 95192790    |
| Innodb_buffer_pool_read_ahead_evicted | 11789798    |
| Innodb_buffer_pool_read_requests      | 48471864300 |
| Innodb_buffer_pool_reads              | 250182943   |
| Innodb_buffer_pool_wait_free          | 0           |
| Innodb_buffer_pool_write_requests     | 862377551   |
+---------------------------------------+-------------+
```


### 5 临时表
```mysql
mysql> show global status like 'created_tmp%';
+-------------------------+----------+
| Variable_name           | Value    |
+-------------------------+----------+
| Created_tmp_disk_tables | 4177263  |
| Created_tmp_files       | 30       |
| Created_tmp_tables      | 28880451 |
+-------------------------+----------+
```
每次创建临时表，Created\_tmp\_tables增加，如果是在磁盘上创建临时表，Created\_tmp\_disk\_tables也增加，Created\_tmp\_files表示MySQL服务创建的临时文件文件数，比较理想的配置是：
Created\_tmp\_disk\_tables / Created\_tmp\_tables * 100% <= 25%

```mysql
mysql> show variables where Variable_name in ('tmp_table_size', 'max_heap_table_size');
+---------------------+----------+
| Variable_name       | Value    |
+---------------------+----------+
| max_heap_table_size | 16777216 |
| tmp_table_size      | 16777216 |
+---------------------+----------+
```
只有256MB以下的临时表才能全部放内存，超过的就会用到硬盘临时表。

### 6 Open Table情况
```mysql
mysql> show global status like 'open%tables%';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| Open_tables   | 512   |       -- 打开表的数量
| Opened_tables | 14545 |       -- 表示打开过的表数量
+---------------+-------+
```
如果Opened\_tables数量过大，说明配置中table\_open\_cache值可能太小，我们查询一下服务器table_cache值：

```mysql
mysql> show variables like 'table_open_cache%';
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| table_open_cache | 512   |
+------------------+-------+
```
比较合适的值为：
Open\_tables / Opened\_tables   * 100% >= 85%
Open\_tables / table\_open\_cache * 100% <= 95%

### 7 进程使用情况
```mysql
mysql> show global status like 'Thread%';
+-------------------+--------+
| Variable_name     | Value  |
+-------------------+--------+
| Threads_cached    | 8      |
| Threads_connected | 38     |
| Threads_created   | 111206 |
| Threads_running   | 1      |
+-------------------+--------+
```
如果我们在MySQL服务器配置文件中设置了thread\_cache\_size，当客户端断开之后，服务器处理此客户的线程将会缓存起来以响应下一个客户 而不是销毁（前提是缓存数未达上限）。Threads\_created表示创建过的线程数，如果发现Threads\_created值过大的话，表明 MySQL服务器一直在创建线程，这也是比较耗资源，可以适当增加配置文件中thread\_cache\_size值，查询服务器thread\_cache\_size配置：

```mysql
mysql> show variables like 'thread_cache_size';
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| thread_cache_size | 8     |
+-------------------+-------+
```

### 8 排序使用情况
```mysql
mysql> show global status like 'sort%';
+-------------------+----------+
| Variable_name     | Value    |
+-------------------+----------+
| Sort_merge_passes | 3        |
| Sort_range        | 608266   |
| Sort_rows         | 20752276 |
| Sort_scan         | 6116860  |
+-------------------+----------+
```
Sort\_merge\_passes 包括两步。MySQL 首先会尝试在内存中做排序，使用的内存大小由系统变量Sort\_buffer\_size 决定，如果它的大小不够把所有的记录都读到内存中，MySQL 就会把每次在内存中排序的结果存到临时文件中，等MySQL 找到所有记录之后，再把临时文件中的记录做一次排序。这再次排序就会增加Sort\_merge\_passes。实际上，MySQL会用另一个临时文件来存再次排序的结果，所以通常会看到Sort\_merge\_passes增加的数值是建临时文件数的两倍。因为用到了临时文件，所以速度可能会比较慢，增加Sort\_buffer\_size 会减少Sort\_merge\_passes 和 创建临时文件的次数，但盲目的增加Sort\_buffer\_size 并不一定能提高速度。

### 9 文件打开数(open_files)
```mysql
mysql> show global status like 'open_files';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| Open_files    | 9     |       -- 打开文件数
+---------------+-------+

mysql> show variables like 'open_files_limit';
+------------------+--------+
| Variable_name    | Value  |
+------------------+--------+
| open_files_limit | 306154 |   -- 比较合适的设置：Open_files / open_files_limit * 100% <= 75％
+------------------+--------+
```


### 10 表锁情况
```mysql
mysql> show global status like 'table_locks%';
+-----------------------+-----------+
| Variable_name         | Value     |
+-----------------------+-----------+
| Table_locks_immediate | 176460059 |           -- 即释放表锁数
| Table_locks_waited    | 59650     |           -- 需要等待的表锁数
+-----------------------+-----------+
```
比较合适的值为：Table\_locks\_immediate / Table\_locks\_waited > 5000。

### 11 表扫描情况
```mysql
mysql> show global status like 'handler_read%';
+-----------------------+--------------+
| Variable_name         | Value        |
+-----------------------+--------------+
| Handler_read_first    | 14688150     |
| Handler_read_key      | 1410907010   |
| Handler_read_last     | 130          |
| Handler_read_next     | 46733612527  |
| Handler_read_prev     | 1            |
| Handler_read_rnd      | 1075430049   |
| Handler_read_rnd_next | 171056223252 |
+-----------------------+--------------+

mysql> show global status like 'com_select';
+---------------+----------+
| Variable_name | Value    |
+---------------+----------+
| Com_select    | 90888200 |
+---------------+----------+
```
计算表扫描率：
表扫描率 ＝ Handler\_read\_rnd\_next / Com\_select
如果表扫描率超过4000，说明进行了太多表扫描，很有可能索引没有建好，增加read\_buffer\_size值会有一些好处，但最好不要超过8MB。


