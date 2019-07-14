---
layout: post
title: BigQuery various table types
tags: [GCP]
category: [GCP]
author: vikrant
comments: true
--- 

Partitioned tables are a special kind of table in dataset that is divided into segments called partitions to easily query the data. Maily two type of partitioning exist in BQ:

- Ingestion-time based partitioning
- Column based partitioning. 

#### Ingestion-time based partitioning

- Based on time when the data is injected into the bq, tables get paritioned. 

- Command to create ingestion time based table.

```
$ bq mk -t --expiration 2592000 --time_partitioning_type=DAY --time_partitioning_expiration 259200 --description "This is my time-partitioned table" --label organization:development test_dataset_1.my_table_1 qtr:STRING,sales:INTEGER,year:STRING
Table 'test-project1:test_dataset_1.my_table_1' successfully created.
```

- Check the schema and expiration time of table. 

```
$ bq show test_dataset_1.my_table_1
Table test-project1:test_dataset_1.my_table_1

   Last modified         Schema         Total Rows   Total Bytes     Expiration            Time Partitioning         Clustered Fields            Labels           
 ----------------- ------------------- ------------ ------------- ----------------- ------------------------------- ------------------ -------------------------- 
  14 Jul 13:10:56   |- date: string     2            48            13 Aug 13:06:06   DAY (expirationMs: 259200000)                      organization:development  
                    |- sales: integer                                                                                                                             
                    |- year: string                                  
```

- Insert some data into table.

```

INSERT INTO test_dataset_1.my_table_1 (date,sales,year) VALUES ('2019-04-20',30000,'2019')

$ bq query --nouse_legacy_sql 'SELECT * FROM test_dataset_1.my_table_1'
Waiting on bqjob_r4723fbc39286830f_0000016befc29e7f_1 ... (0s) Current status: DONE   
+------------+-------+------+
|    date    | sales | year |
+------------+-------+------+
| 2019-04-20 | 30000 | 2019 |
| 2019-04-24 | 20000 | 2019 |
+------------+-------+------+
```

- In case of ingestion-time based partitioned tables a new pseudo column (_PARTITIONTIME) is inject into the table, you can't see this pseudo column but we can use this in our query to narrow down the result of our query.

```
$ bq query --nouse_legacy_sql 'select * from test_dataset_1.my_table_1 WHERE _PARTITIONTIME = TIMESTAMP("2019-07-14")'
Waiting on bqjob_r5c545e2e32011cf9_0000016befc8a766_1 ... (0s) Current status: DONE   
+------------+-------+------+
|    date    | sales | year |
+------------+-------+------+
| 2019-04-24 | 20000 | 2019 |
| 2019-04-20 | 30000 | 2019 |
+------------+-------+------+
```

- Simple SELECT * on table will not show this pseudo column if you want to see this column then explicitly state the name of pseudo column.


```
$ bq query --nouse_legacy_sql 'SELECT date,sales,_PARTITIONTIME as pt FROM `test_dataset_1.my_table_1`'
Waiting on bqjob_r3b1216902e5f9e6f_0000016befcc4ca5_1 ... (0s) Current status: DONE   
+------------+-------+---------------------+
|    date    | sales |         pt          |
+------------+-------+---------------------+
| 2019-04-24 | 20000 | 2019-07-14 00:00:00 |
| 2019-04-20 | 30000 | 2019-07-14 00:00:00 |
+------------+-------+---------------------+

Another way using legacy sql

$ bq query --use_legacy_sql 'SELECT * from [test_dataset_1.my_table_1$__PARTITIONS_SUMMARY__]'
Waiting on bqjob_r7226f8ffacc355f0_0000016bf01f05f9_1 ... (0s) Current status: DONE   
+------------+----------------+------------+--------------+---------------+---------------------+--------------------+-------------------------+
| project_id |   dataset_id   |  table_id  | partition_id | creation_time | creation_timestamp  | last_modified_time | last_modified_timestamp |
+------------+----------------+------------+--------------+---------------+---------------------+--------------------+-------------------------+
| test-project1 | test_dataset_1 | my_table_1 | 20190714     | 1563095507274 | 2019-07-14 09:11:47 |      1563095520896 |     2019-07-14 09:12:00 |
+------------+----------------+------------+--------------+---------------+---------------------+--------------------+-------------------------+
```

#### Column based partitioning

- Column based partitioning based on specific column. Must be of type DATE and TIMESTAMP. No _PARTITIONTIME pseudo-column. Two special partitioins are present __NULL__ and __UNPARTITIONED__.

- In the following command entrydate column is used to partition the data. Note the type of entrydate is date if you put any type apart from DATE and TIMESTAMP for the column which you gonna use for column partitioning table creation command will fail. 

```
$ bq mk -t --time_partitioning_field=entrydate --description "This is my date partitioned table" --label organization:development test_dataset_1.my_table_2 entrydate:DATE,sales:integer,year:STRING
Table 'test-project1:test_dataset_1.my_table_2' successfully created.
```

- Injected two rows into this table. 

```
$ bq show test_dataset_1.my_table_2
Table test-project1:test_dataset_1.my_table_2

   Last modified          Schema         Total Rows   Total Bytes   Expiration     Time Partitioning      Clustered Fields            Labels           
 ----------------- -------------------- ------------ ------------- ------------ ------------------------ ------------------ -------------------------- 
  14 Jul 15:57:43   |- entrydate: date   2            44                         DAY (field: entrydate)                      organization:development  
                    |- sales: integer                                                                                                                  
                    |- year: string                                                                                                      
INSERT INTO test_dataset_1.my_table_2 (entrydate,sales,year) VALUES ("2019-05-23",70000,"2019")

$ bq query --nouse_legacy_sql 'SELECT * FROM test_dataset_1.my_table_2'
Waiting on bqjob_r18571ce0a2803f53_0000016bf006ae5c_1 ... (0s) Current status: DONE   
+------------+-------+------+
| entrydate  | sales | year |
+------------+-------+------+
| 2019-04-23 | 70000 | 2019 |
| 2019-05-23 | 70000 | 2019 |
+------------+-------+------+
```

- To list all the partitions present in table, we have to use the legacy query.

```
$ bq query --use_legacy_sql 'SELECT * from [test_dataset_1.my_table_2$__PARTITIONS_SUMMARY__]'
Waiting on bqjob_r756ff0299392c7a3_0000016bf01f4b87_1 ... (0s) Current status: DONE   
+------------+----------------+------------+--------------+---------------+---------------------+--------------------+-------------------------+
| project_id |   dataset_id   |  table_id  | partition_id | creation_time | creation_timestamp  | last_modified_time | last_modified_timestamp |
+------------+----------------+------------+--------------+---------------+---------------------+--------------------+-------------------------+
| test-project1 | test_dataset_1 | my_table_2 | 20190423     | 1563100040838 | 2019-07-14 10:27:20 |      1563100041031 |     2019-07-14 10:27:21 |
| test-project1 | test_dataset_1 | my_table_2 | 20190523     | 1563100063653 | 2019-07-14 10:27:43 |      1563100063872 |     2019-07-14 10:27:43 |
+------------+----------------+------------+--------------+---------------+---------------------+--------------------+-------------------------+

$ bq query --nouse_legacy_sql 'SELECT * FROM test_dataset_1.my_table_2 where entrydate = DATE("2019-04-23")'
Waiting on bqjob_rf7ca4e87c0d2571_0000016bf01a7b39_1 ... (0s) Current status: DONE   
+------------+-------+------+
| entrydate  | sales | year |
+------------+-------+------+
| 2019-04-23 | 70000 | 2019 |
+------------+-------+------+
```

#### Clustered table

These tables are used to organize data based on columns when data is already segmented into paritioned tables. This helps to improve the performance of column based queries.  