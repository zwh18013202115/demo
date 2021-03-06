# 03_sparkLoad2DorisDB

#  Description

Demonstration loading data into DorisDB using the SparkLoad feature.

# Preparations

## Hadoop environment

Deploy Hadoop including YARN, HDFS and SPARK on Linux hosts.

This demo follows below component versions:  
- hadoop 2.7.7
- spark 2.3.3

### About JAVA_HOME variable

> On the top of hadoop/libexec/hadoop-config.sh, add below export: 

```
export JAVA_HOME=xxxxJAVA_HOME_PATH
```

## DorisDB Cluster

### fe configures

> configure below options in fe.conf

```
enable_spark_load=true
spark_home_default_dir=/usr/local/spark-2.3.3-bin-hadoop2.7/
yarn_client_path=/usr/local/hadoop-2.7.7/bin/yarn
```

### prepare a spark-2x.zip file

```
[root@master1 ~ ]# cd /usr/local/spark-2.3.3-bin-hadoop2.7/jars
[root@master1 jars]# zip -r spark-2x.zip ./*.jar
```

# Case1

Load CSV file on HDFS into DorisDB

## Mimic Data

Simulate csv file with 10000 lines, 2 cols and upload to hdfs

- call [gen_wide.py](../SparkDemo/src/main/py/gen_wide.py) in module [SparkDemo](../SparkDemo)


```
[root@master1 data]# python data_wide.py 10000 2 > demo3_data1.csv
[root@master1 data]# grep ^[0-9] demo3_data1.csv | wc -l
10000
[root@master1 data]# head demo3_data1.csv
1        10
9        5
8        8
8        3
9        11
8        4
5        12
2        7
3        3
6        5
[root@master1 ~]# hadoop fs -mkdir -p  /dorisDB-demo/data
[root@master1 data]# hadoop fs -put demo3_data1.csv /dorisDB-demo/data/

```

## Testing

DorisDB DDL

```
CREATE TABLE `doris_demo`.`demo3_spark_tb1` (
    `k1`  varchar(50) NULL  COMMENT "",
    `v1`  String      NULL  COMMENT ""
) ENGINE=OLAP
DUPLICATE KEY(`k1` )
COMMENT "OLAP"
DISTRIBUTED BY HASH(`v1` ) BUCKETS 3
PROPERTIES (
    "replication_num" = "1",
    "in_memory" = "false",
    "storage_format" = "DEFAULT"
);
```

Create spark1 resource in dorisDB:

```
-- add broker1
MySQL [(none)]> ALTER SYSTEM ADD BROKER broker1 "master1:8000";
Query OK, 0 rows affected (0.04 sec)

-- yarn HA cluster mode
CREATE EXTERNAL RESOURCE "spark1"
PROPERTIES
(
  "type" = "spark",
  "spark.master" = "yarn",
  "spark.submit.deployMode" = "cluster",
  "spark.hadoop.yarn.resourcemanager.ha.enabled" = "true",
  "spark.hadoop.yarn.resourcemanager.ha.rm-ids" = "rm1,rm2",
  "spark.hadoop.yarn.resourcemanager.hostname.rm1" = "master1",
  "spark.hadoop.yarn.resourcemanager.hostname.rm2" = "worker1",
  "spark.hadoop.fs.defaultFS" = "hdfs://mycluster/",
  "working_dir" = "hdfs://mycluster/tmp/doris",
  "broker" = "broker1"
);
```

submit spark load job:

```
USE doris_demo;
LOAD LABEL doris_demo.label1
(
    DATA INFILE("hdfs://mycluster/dorisDB-demo/data/demo3_data1.csv")
    INTO TABLE demo3_spark_tb1
    COLUMNS TERMINATED BY "\t"
    (k1,v1)
    SET
    (
        k1=k1,
        v1=v1
    )
)
WITH RESOURCE 'spark1'
(
    "spark.executor.memory" = "500m",
    "spark.shuffle.compress" = "true",
    "spark.driver.memory" = "1g"
)
PROPERTIES
(
    "timeout" = "3600",
    "max_filter_ratio" = "0.2"
);
```

Verification

```
MySQL [doris_demo]> select * from demo3_spark_tb1 limit 5;
+------+------+
| k1   | v1   |
+------+------+
| 1    | 10   |
| 1    | 12   |
| 1    | 10   |
| 1    | 12   |
| 1    | 10   |
+------+------+
5 rows in set (0.18 sec)

MySQL [doris_demo]> select count(1) from demo3_spark_tb1 limit 5;
+----------+
| count(1) |
+----------+
|    10000 |
+----------+
1 row in set (0.07 sec)

MySQL [doris_demo]> select count(distinct v1) v1 from demo3_spark_tb1 limit 5;
+------+
| v1   |
+------+
|   12 |
+------+
1 row in set (0.03 sec)

MySQL [doris_demo]> select count(distinct k1) k1 from demo3_spark_tb1 limit 5;
+------+
| k1   |
+------+
|   10 |
+------+
1 row in set (0.03 sec)
```


# Case2

Load parquet file into DorisDB via Spark-load 

> requirement

- External table in Hive
- External table in DorisDB


## Mimic Data

Convert CSV into parquet format in Spark REPL environment (Spark-shell)

```
scala> sc.setLogLevel("ERROR")

scala> val df = spark.read.option("delimiter","\t").csv("hdfs://mycluster/dorisDB-demo/data/demo3_data1.csv").toDF("k1","v1")
df: org.apache.spark.sql.DataFrame = [k1: string, v1: string]

scala> df.show(5, false)
+---+---+
|k1 |v1 |
+---+---+
|1  |10 |
|9  |5  |
|8  |8  |
|8  |3  |
|9  |11 |
+---+---+
only showing top 5 rows

scala> df.coalesce(1).write.parquet("hdfs://mycluster/dorisDB-demo/data/demo3_data1.parquet")

scala> spark.read.parquet("hdfs://mycluster/dorisDB-demo/data/demo3_data1.parquet").show(5)
+---+---+
| k1| v1|
+---+---+
|  1| 10|
|  9|  5|
|  8|  8|
|  8|  3|
|  9| 11|
+---+---+
only showing top 5 rows     
```  

## Testing

### Hive DDL

```
CREATE EXTERNAL TABLE `t1`(
 `k1` string,
 `v1` string)
ROW FORMAT SERDE
 'org.apache.hadoop.hive.ql.io.parquet.serde.ParquetHiveSerDe'
STORED AS INPUTFORMAT
 'org.apache.hadoop.hive.ql.io.parquet.MapredParquetInputFormat'
OUTPUTFORMAT
 'org.apache.hadoop.hive.ql.io.parquet.MapredParquetOutputFormat'
LOCATION
 'hdfs://mycluster/dorisDB-demo/data/demo3_data1.parquet'
TBLPROPERTIES ( 'parquet.compression'='snappy');
```

### DorisDB DDL

```
CREATE EXTERNAL RESOURCE "hive0"
PROPERTIES (
  "type" = "hive", 
  "hive.metastore.uris" = "thrift://master1:9083"
);


MySQL [doris_demo]> create table demo3_spark_tb2 like demo3_spark_tb1;
Query OK, 0 rows affected (0.07 sec)


MySQL [doris_demo]> CREATE EXTERNAL TABLE hive_t1
    ->     (
    ->          k1 string,
    ->          v1 string
    ->      )
    ->  ENGINE=hive
    ->  properties (
    ->     "resource" = "hive0",
    ->     "database" = "default",
    ->     "table" = "t1");
Query OK, 0 rows affected (0.03 sec)

```

### Spark load 

Load data from external hive table into DorisDB inner table, using spark1 resource

```
USE doris_demo;
LOAD LABEL doris_demo.label2
(
    DATA FROM TABLE hive_t1
    INTO TABLE demo3_spark_tb2
)
WITH RESOURCE 'spark1'
(
    "spark.executor.memory" = "2g",
    "spark.shuffle.compress" = "true",
    "spark.driver.memory" = "1g"
)
PROPERTIES
(
    "timeout" = "3600",
    "max_filter_ratio" = "0.2"
);
```

show load

```
*************************** 9. row ***************************
         JobId: 14039
         Label: label2
         State: FINISHED
      Progress: ETL:100%; LOAD:100%
          Type: SPARK
       EtlInfo: unselected.rows=0; dpp.abnorm.ALL=0; dpp.norm.ALL=10000
      TaskInfo: cluster:spark1; timeout(s):3600; max_filter_ratio:0.2
      ErrorMsg: NULL
    CreateTime: 2021-05-31 21:05:45
  EtlStartTime: 2021-05-31 21:06:12
 EtlFinishTime: 2021-05-31 21:06:46
 LoadStartTime: 2021-05-31 21:06:46
LoadFinishTime: 2021-05-31 21:06:49
           URL: http://worker1:20888/proxy/application_1622453682723_0025/
    JobDetails: {"Unfinished backends":{"00000000-0000-0000-0000-000000000000":[]},"ScannedRows":9999,"TaskNumber":1,"All backends":{"00000000-0000-0000-0000-000000000000":[-1]},"FileNumber":0,"FileSize":0}
9 rows in set (0.00 sec)
```

### Verification

```
MySQL [doris_demo]> select * from demo3_spark_tb2 limit 5;
+------+------+
| k1   | v1   |
+------+------+
| 1    | 3    |
| 1    | 2    |
| 1    | 3    |
| 1    | 2    |
| 1    | 6    |
+------+------+
5 rows in set (0.06 sec)

MySQL [doris_demo]> select count(1) from demo3_spark_tb2 limit 5;
+----------+
| count(1) |
+----------+
|    10000 |
+----------+
1 row in set (0.03 sec)

MySQL [doris_demo]> select count(distinct k1) k1 from demo3_spark_tb2 limit 5;
+------+
| k1   |
+------+
|   10 |
+------+
1 row in set (0.02 sec)

MySQL [doris_demo]> select count(distinct v1) v1 from demo3_spark_tb2 limit 5;
+------+
| v1   |
+------+
|   12 |
+------+
1 row in set (0.02 sec)
```

# NOTE
spark-submit logs can be found under below path on a fe-leader node :

```
log/spark_launcher_log/
```


# License

DorisDB/demo is under the Apache 2.0 license. See the [LICENSE](../LICENSE) file for details.