---
layout: post
title: Big Query nested 
tags: [GCP]
category: [GCP]
author: vikrant
comments: true
--- 

*Disclaimer: This article assumes that you have some basic understanding about bq concepts.*

Recently, I started exploring google BQ (most popular service) for one of our internal project for log analysis. I came across concept of denormalized storage which is opposite the one proposed in relational DBs. Denormalized storage uses the nested structures, it provides advantage of reduced computation costs and other benefits include minimize joins. 

In this article, I am sharing information on the nested structs in BQ. Before diving into the world of nested structs some information about the basics of table creationg in BQ.

In bq table schema can be specified at the time of creation or while loading the data or let it automatically detect from the data which we are uploading to bq. In all three examples used in this article, I am providing schema at the time of table creation. While providing schema, you need to specify the name of column and type of column, mode of column is an optional item. Type of column helps to understand the type of data which you can store in column. STRING AND INTEGER are the most commonly used data types. 

#### Nested types:

- STRUCT (Kind of dictionary)  TYPE = RECORD, MODE = nullable or required 
- REPEATED (List) TYPE = ANY except RECORD, datatype MODE
- REPEATED STRUCT (each element is dictionary) TYPE = RECORD, MODE = repeated 

#### Nested Operators:

- STRUCT : Create dictionary in sql query
- UNNEST : Unpack the list from table
- ARRAY_AGG :  Create list of struct in sql query 

#### Example 1

- Creating table with all type of nested types. 

a) Name (string) type=string,mode=required
b) Age (integer) type=integer,mode=required
c) Gender (string) type=string,mode=nullable
d) Hobbies (array) type=string,mode=repeated
e) personaldetails(struct) type=record,mode=nullable
   nested field 1 : bloodgroup(string) type=string,mode=nullable
   nested field 2 : salary(integer) type=integer,mode=nullable
f) addresses(array) type=record,mode=repeated
   nested field 1 : address(string) type=string,mode=nullable
   nested field 2 : city(string) type=string,mode=nullable
   nested field 3 : state(string) type=string,mode=nullable
   nested field 4 : zip(string) type=string,mode=nullable

You can find the json schema [here](https://github.com/ervikrant06/bigquery_examples/blob/master/table_1_schema.json)

```
$ bq mk --table test_dataset_1.test_table_1 table_1_schema.json
```

- Use the following command to verify the schema of table. 

```
$ bq query --nouse_legacy_sql 'SELECT * FROM test_dataset_1.INFORMATION_SCHEMA.COLUMNS WHERE TABLE_NAME = "test_table_1"'
Waiting on bqjob_r62be1ad6eb014d5d_0000016bee93a65b_1 ... (0s) Current status: DONE   
+---------------+----------------+--------------+-----------------+------------------+-------------+----------------------------------------------------------------------+--------------+-----------------------+-----------+-----------+--------------+-------------------+------------------------+-----------------------------+
| table_catalog |  table_schema  |  table_name  |   column_name   | ordinal_position | is_nullable |                              data_type                               | is_generated | generation_expression | is_stored | is_hidden | is_updatable | is_system_defined | is_partitioning_column | clustering_ordinal_position |
+---------------+----------------+--------------+-----------------+------------------+-------------+----------------------------------------------------------------------+--------------+-----------------------+-----------+-----------+--------------+-------------------+------------------------+-----------------------------+
| test-project1    | test_dataset_1 | test_table_1 | Name            |                1 | NO          | STRING                                                               | NEVER        | NULL                  | NULL      | NO        | NULL         | NO                | NO                     |                        NULL |
| test-project1    | test_dataset_1 | test_table_1 | Age             |                2 | NO          | INT64                                                                | NEVER        | NULL                  | NULL      | NO        | NULL         | NO                | NO                     |                        NULL |
| test-project1    | test_dataset_1 | test_table_1 | Gender          |                3 | YES         | STRING                                                               | NEVER        | NULL                  | NULL      | NO        | NULL         | NO                | NO                     |                        NULL |
| test-project1    | test_dataset_1 | test_table_1 | Hobbies         |                4 | NO          | ARRAY<STRING>                                                        | NEVER        | NULL                  | NULL      | NO        | NULL         | NO                | NO                     |                        NULL |
| test-project1    | test_dataset_1 | test_table_1 | personaldetails |                5 | YES         | STRUCT<bloodgroup STRING, salary INT64>                              | NEVER        | NULL                  | NULL      | NO        | NULL         | NO                | NO                     |                        NULL |
| test-project1    | test_dataset_1 | test_table_1 | addresses       |                6 | NO          | ARRAY<STRUCT<address STRING, city STRING, state STRING, zip STRING>> | NEVER        | NULL                  | NULL      | NO        | NULL         | NO                | NO                     |                        NULL |
+---------------+----------------+--------------+-----------------+------------------+-------------+----------------------------------------------------------------------+--------------+-----------------------+-----------+-----------+--------------+-------------------+------------------------+-----------------------------+
```

- Insert some data into table either using the CLI or GUI. 

```
bq query --nouse_legacy_sql 'INSERT INTO test_dataset_1.test_table_1 (Name,Age,Hobbies,personaldetails,addresses) 
VALUES ('Sunil',35,['basketball','cricket'],('O+',35000),[('Sector 17','Gurgaon','Haryana','122001')])'
```

- Verify the data inserted into the table. 

```
$ bq query --nouse_legacy_sql 'SELECT * FROM test_dataset_1.test_table_1'
Waiting on bqjob_r7b72affcbfc29a73_0000016beea5e393_1 ... (0s) Current status: DONE   
+-------+-----+--------+------------------------------+--------------------------------------+-----------------------------------------------------------------------------+
| Name  | Age | Gender |           Hobbies            |           personaldetails            |                                  addresses                                  |
+-------+-----+--------+------------------------------+--------------------------------------+-----------------------------------------------------------------------------+
| Sunil |  35 | NULL   | ["basketball","cricket"]     | {"bloodgroup":"O+","salary":"35000"} | [{"address":"Sector 17","city":"Gurgaon","state":"Haryana","zip":"122001"}] |
| Jon   |  40 | NULL   | ["basketball","programming"] | {"bloodgroup":"O+","salary":"30000"} | [{"address":"Sector 14","city":"Gurgaon","state":"Haryana","zip":"122001"}] |
| Tom   |  30 | NULL   | ["reading","programming"]    | {"bloodgroup":"B+","salary":"20000"} | [{"address":"Sector 18","city":"Gurgaon","state":"Haryana","zip":"122001"}] |
+-------+-----+--------+------------------------------+--------------------------------------+-----------------------------------------------------------------------------+
```

- Query the data from the table:

a) UNNEST operator to unpack the hobbies list from test_table_1

```
$ bq query --nouse_legacy_sql 'SELECT name,hobby from test_dataset_1.test_table_1, UNNEST(Hobbies) as hobby'
Waiting on bqjob_r3efdbe6b280ebb5c_0000016beea9f9bc_1 ... (0s) Current status: DONE   
+-------+-------------+
| name  |    hobby    |
+-------+-------------+
| Sunil | basketball  |
| Sunil | cricket     |
| Jon   | basketball  |
| Jon   | programming |
| Tom   | reading     |
| Tom   | programming |
+-------+-------------+
```

b) To unpack the list of STRUCT also use UNNEST operator and refer each element of struct using `.` notation in name.

```
$ bq query --nouse_legacy_sql 'SELECT name,hobby,personaldetails.salary,addressdetail.address from test_dataset_1.test_table_1, UNNEST(Hobbies) as hobby, UNNEST(addresses) as addressdetail'
Waiting on bqjob_r6b8e24c73906fc6e_0000016beeaf35b4_1 ... (0s) Current status: DONE   
+-------+-------------+--------+-----------+
| name  |    hobby    | salary |  address  |
+-------+-------------+--------+-----------+
| Tom   | reading     |  20000 | Sector 18 |
| Tom   | programming |  20000 | Sector 18 |
| Sunil | basketball  |  35000 | Sector 17 |
| Sunil | cricket     |  35000 | Sector 17 |
| Jon   | basketball  |  30000 | Sector 14 |
| Jon   | programming |  30000 | Sector 14 |
+-------+-------------+--------+-----------+
```

#### Example 2

- We have seen the basic usage of UNSTRUCT Operator in the previous example, in this example we will create another table and insert the query results into the newly created table. 

You can find the json schema [here](https://github.com/ervikrant06/bigquery_examples/blob/master/table_2_schema.json)

create table using schema. 

```
$ bq mk --table test_dataset_1.test_table_2 table_2_schema.json
Table 'test-project1:test_dataset_1.test_table_2' successfully created.
```

- Issue the query on test_table_1 and inject the results into test_table_2.

```
$ bq query --nouse_legacy_sql 'INSERT INTO test_dataset_1.test_table_2 (name,hobby,salary,address) 
> SELECT name,hobby,personaldetails.salary,addressdetail.address 
> from test_dataset_1.test_table_1, UNNEST(Hobbies) as hobby, UNNEST(addresses) as addressdetail'
Waiting on bqjob_r53602639f277faaf_0000016beebc96a2_1 ... (0s) Current status: DONE   
Number of affected rows: 6
```

- Let's query the content of test_table_2 we can see that REPEATED, STRUCT, REPEATED STRUCT types are flattened into this table because of the query which we used.  

```
$ bq query --nouse_legacy_sql 'SELECT * FROM test_dataset_1.test_table_2'
Waiting on bqjob_r7c8f6a75a6295c90_0000016beebeaa6d_1 ... (0s) Current status: DONE   
+-------+-------------+--------+-----------+
| Name  |    hobby    | salary |  address  |
+-------+-------------+--------+-----------+
| Tom   | programming |  20000 | Sector 18 |
| Jon   | basketball  |  30000 | Sector 14 |
| Tom   | reading     |  20000 | Sector 18 |
| Jon   | programming |  30000 | Sector 14 |
| Sunil | basketball  |  35000 | Sector 17 |
| Sunil | cricket     |  35000 | Sector 17 |
+-------+-------------+--------+-----------+
```

#### Example 3

- In the previous two examples, we have seen the usage of UNSTRUCT operator to flatten the REPEATED (list) type. In this example, we will see the usage of other two operators (ARRAY_AGG and STRUCT). We will create list (hobbies) and list of structs (personaldetails). 

You can find the json schema [here](https://github.com/ervikrant06/bigquery_examples/blob/master/table_3_schema.json)

create table using schema. 

```
$ bq mk --table test_dataset_1.test_table_3 table_3_schema.json
Table 'test-project1:test_dataset_1.test_table_3' successfully created.
```
- Query the test_table_2 which we created in previous example remember this table contains the flattened data, we will use ARRAY_AGG and STRUCT operators to combine the data; Name is used to group the rows. It may look like a very useful example but this is suffice for the scope of article. We are injecting the query results into test_table_3 which we created in last step for this example. 

```
$ bq query --nouse_legacy_sql 'INSERT INTO test_dataset_1.test_table_3
> SELECT name, ARRAY_AGG(STRUCT(salary,address)) AS personaldetails, ARRAY_AGG(hobby) AS hobbies 
> FROM test_dataset_1.test_table_2
> GROUP BY Name'
Waiting on bqjob_r4de0f27fe5359153_0000016beec8f85f_1 ... (0s) Current status: DONE   
Number of affected rows: 3
```

- Query the results from test_table_3, personaldetails is an array of structs and hobbies is an array.

```
$ bq query --nouse_legacy_sql 'SELECT * FROM test_dataset_1.test_table_3'
Waiting on bqjob_r11100b1e0007f32a_0000016beec95ccd_1 ... (0s) Current status: DONE   
+-------+-------------------------------------------------------------------------------------+------------------------------+
| Name  |                                   personaldetails                                   |           hobbies            |
+-------+-------------------------------------------------------------------------------------+------------------------------+
| Jon   | [{"salary":"30000","address":"Sector 14"},{"salary":"30000","address":"Sector 14"}] | ["basketball","programming"] |
| Sunil | [{"salary":"35000","address":"Sector 17"},{"salary":"35000","address":"Sector 17"}] | ["basketball","cricket"]     |
| Tom   | [{"salary":"20000","address":"Sector 18"},{"salary":"20000","address":"Sector 18"}] | ["programming","reading"]    |
+-------+-------------------------------------------------------------------------------------+------------------------------+
```

Hope you enjoyed reading this article. 