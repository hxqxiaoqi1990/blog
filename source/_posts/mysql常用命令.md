---
title: mysql常用命令 
tags: [mysql]
categories: 数据库
date: 2019-12-19 10:49:00
---


# 介绍
记录mysql常用命令

# 常用命令
**binlog查询配置**

```bash
# 查询binlog数量
mysql> show binary logs;

# 查询binlog配置
mysql> show global variables like '%log%';
```

**慢查询设置**

``` bash
# 开启慢查询日志
mysql> set global slow_query_log=1;

# 定义时间SQL查询的超时时间
mysql> set global long_query_time = 0.005;

# 查看慢查询信息
mysql> show global variables;
mysql> show global variables like 'slow_query_log_file';
mysql> show global variables like 'long_query_time';
mysql> show global variables like 'slow_query_log';

# 查看慢查询
cat /var/log/mysql/slow.log
```

**慢查询分析**

```bash
# 按时间截取日志
sed -n   '/# Time: 200220/,$'p slow.log > test.log

# 获取执行时间最长的 10个 TOP SQL。
mysqldumpslow -s t -t 10 test10.log > slow_t_top_sql.txt

# 获取平均查询时间最长的 10 个 TOP SQL。
mysqldumpslow -s  at -t 10 test10.log > slow_at_top_sql.txt

# 获取锁定时间最长的 10个 TOP SQL。
mysqldumpslow -s l -t 10 test10.log > slow_l_top_sql.txt

# 获取平均锁定时间最长的 10个 TOP SQL。
mysqldumpslow -s al -t 10 test10.log > slow_l_top_sql.txt

# 获取返回记录最多的 10个 TOP SQL。
mysqldumpslow -s r -t 10 test10.log > slow_r_top_sql.txt

# 获取平均返回记录最多的 10个 TOP SQL。
mysqldumpslow -s ar -t 10 test10.log > slow_r_top_sql.txt

# 获取执行次数最多的 10个 TOP SQL。
mysqldumpslow -s c -t 10 test10.log > slow_r_top_sql.txt
```

**查看事务**

1、查看数据库当前的进程，看一下有无正在执行的慢SQL记录线程。

```js
mysql> show  processlist;
```

2、查看当前的事务

当前运行的所有事务

```js
mysql> SELECT * FROM information_schema.INNODB_TRX;
```

当前出现的锁

```js
mysql> SELECT * FROM information_schema.INNODB_LOCKs;
```

锁等待的对应关系

```js
mysql> SELECT * FROM information_schema.INNODB_LOCK_waits;
```

解释：看事务表INNODB_TRX，里面是否有正在锁定的事务线程，看看ID是否在show processlist里面的sleep线程中，如果是，就证明这个sleep的线程事务一直没有commit或者rollback而是卡住了，我们需要手动kill掉。

搜索的结果是在事务表发现了很多任务，这时候最好都kill掉。

3、批量删除事务表中的事务

我这里用的方法是：通过information_schema.processlist表中的连接信息生成需要处理掉的MySQL连接的语句临时文件，然后执行临时文件中生成的指令。

```js
mysql>  select concat('KILL ',id,';') from information_schema.processlist where user='cms_bokong';
+------------------------+
| concat('KILL ',id,';') |
+------------------------+
| KILL 10508;            |
| KILL 10521;            |
| KILL 10297;            |
+------------------------+
18 rows in set (0.00 sec)
```

当然结果不可能只有3个，这里我只是举例子。参考链接上是建议导出到一个文本，然后执行文本。而我是直接copy到记事本处理掉 ‘|’，粘贴到命令行执行了。都可以。

kill掉以后再执行`SELECT * FROM information_schema.INNODB_TRX;` 就是空了。

这时候系统就正常了

**查看命令**

```bash
# 查看以deprecated开头的表名
mysql> SELECT table_name from information_schema.columns where table_name like 'deprecated\_%' group by table_name;
```

