# 07_flinkConnector_Sql2DorisDB

## DDL

```
MySQL [doris_demo]> CREATE TABLE `doris_demo`.`demo2_flink_tb1` (
    ->   `NAME` VARCHAR(100) NOT NULL COMMENT "ε§ε",
    ->   `SCORE` INT(2) NOT NULL COMMENT "εΎε"
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

## Performing

Run [Sql2DorisDB](../FlinkDemo/src/main/scala/com/dorisdb/flink/Sql2DorisDB.scala) directly in IDEA

## Verification

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
|      231 |
+----------+
1 row in set (0.02 sec)

MySQL [doris_demo]> select sum(score) sc , name from demo2_flink_tb1 group by name;
+------+---------+
| sc   | name    |
+------+---------+
| 3922 | lebron  |
| 3538 | kobe    |
| 3496 | stephen |
+------+---------+
3 rows in set (0.02 sec)

MySQL [doris_demo]> select sum(score) sc , name from demo2_flink_tb1 group by name;
+------+---------+
| sc   | name    |
+------+---------+
| 3627 | kobe    |
| 3592 | stephen |
| 3946 | lebron  |
+------+---------+
3 rows in set (0.02 sec)
```