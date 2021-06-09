# 06_flinkConnector_Json2DorisDB

## DDL

```
MySQL [doris_demo]> CREATE TABLE `doris_demo`.`demo2_flink_tb1` (
    ->   `NAME` VARCHAR(100) NOT NULL COMMENT "姓名",
    ->   `SCORE` INT(2) NOT NULL COMMENT "得分"
    -> ) ENGINE=OLAP
    -> DUPLICATE KEY(`NAME`)
    -> COMMENT "OLAP"
    -> DISTRIBUTED BY HASH(`NAME`) BUCKETS 3
    -> PROPERTIES (
    -> "replication_num" = "1",
    -> "in_memory" = "false",
    -> "storage_format" = "V2"
    -> );
Query OK, 0 rows affected (0.11 sec)

```

## 执行程序

IDEA里执行 FlinkDemo模块的[Demo2](../FlinkDemo/src/main/scala/com/dorisdb/flink/Demo2.scala)

## 验证数据持续导入

```
MySQL [doris_demo]> select * from demo2_flink_tb1 limit 5;
+--------+-------+
| NAME   | SCORE |
+--------+-------+
| lebron |    43 |
| lebron |    11 |
| lebron |    42 |
| lebron |    96 |
| kobe   |    29 |
+--------+-------+
5 rows in set (0.08 sec)

MySQL [doris_demo]> select count(1) from demo2_flink_tb1;
+----------+
| count(1) |
+----------+
|       18 |
+----------+
1 row in set (0.04 sec)

MySQL [doris_demo]> select sum(score) sc , name from demo2_flink_tb1 group by name;
+------+---------+
| sc   | name    |
+------+---------+
| 2067 | kobe    |
| 1825 | stephen |
| 2156 | lebron  |
+------+---------+
3 rows in set (0.03 sec)

MySQL [doris_demo]> select sum(score) sc , name from demo2_flink_tb1 group by name;
+------+---------+
| sc   | name    |
+------+---------+
| 2187 | lebron  |
| 2094 | kobe    |
| 1835 | stephen |
+------+---------+
3 rows in set (0.02 sec)

MySQL [doris_demo]> select sum(score) sc , name from demo2_flink_tb1 group by name;
+------+---------+
| sc   | name    |
+------+---------+
| 2094 | kobe    |
| 1835 | stephen |
| 2187 | lebron  |
+------+---------+
3 rows in set (0.01 sec)
```
