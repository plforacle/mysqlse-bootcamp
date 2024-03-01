# Database Design

## Introduction
In this lab you will execute some SQL operations, practice using JSON and use MySQL Document Store.

Estimated Lab Time: 30 minutes

### Objectives
In this lab, you will:
* Work with SQL and JSON
* Understand the functioning of MySQL Document Store 
* Practice some CRUD operations.

> **Note:** 
  * Server: mysql1
  * Tasks 2 and 3 can be completed from app-srv

## Task 1: Work with SQL
1. Open an SSH client to app-srv
	```
    <span style="color:green">shell></span><copy> ssh -i id_rsa_app-srv opc@<public_ip></copy>
    ```

2. Connect to <span style="color:green">shell-mysql1</span>

    ```
    <span style="color:green">shell-app-srv$</span><copy> ssh -i $HOME/sshkeys/id_rsa_mysql1 opc@mysql1 </copy>
    ```
3. Now the lab start: connect to your <span style="color:red">mysql-advanced</span> with admin user
    ```
    <span style="color:green">shell></span><copy>mysql -uadmin -p -P3307 -hmysql1</copy>
    ```
4. Create a new table poi
     ```
    <span style="color:blue">mysql></span><copy>use world;</copy>
    ```
     ```
    <span style="color:blue">mysql></span><copy>CREATE TABLE if not exists poi (x Int, y INT, z INT);</copy>
    ```
5. Add to the table a new column for id used for large integer values
     ```
    <span style="color:blue">mysql></span><copy>alter table poi add id bigint;</copy>
    ```
    ```
    <span style="color:blue">mysql></span><copy>ALTER TABLE poi ADD PRIMARY KEY (id);</copy>
    ```
6. Create a copy of your city table
    ```
    <span style="color:blue">mysql></span><copy>create table city_part as select * from city;</copy>
    ```
7. How many records does it contain?
    ```
    <span style="color:blue">mysql></span><copy>SELECT count(*) FROM city_part;</copy>
    ```
8. How many records city table contain?
    ```
    <span style="color:blue">mysql></span><copy>SELECT count(*) FROM city;</copy>
    ```
9. Verify the difference of the two table creation (there is a big one!)
    ```
    <span style="color:blue">mysql></span><copy>show create table city\G</copy>
    ```
    ```
    <span style="color:blue">mysql></span><copy>show create table city_part\G</copy>
    ```
10. Create an index on new table
    ```
    <span style="color:blue">mysql></span><copy>CREATE INDEX myidindex ON city_part (ID);</copy>
    ```
11. Check table statistics. What is the Cardinality (=unique records) of primary key?
    ```
    <span style="color:blue">mysql></span><copy>SELECT * FROM INFORMATION_SCHEMA.STATISTICS WHERE table_name ='city' and table_schema='world'\G</copy>
    ```
12. Create a new index
    ```
    <span style="color:blue">mysql></span><copy>CREATE INDEX myccindex ON city_part (CountryCode);</copy>
    ```
13. Delete some columns (Population and CountryCode)
    ```
    <span style="color:blue">mysql></span><copy>ALTER TABLE city_part DROP COLUMN Population;</copy>
    ```
    ```
    <span style="color:blue">mysql></span><copy>ALTER TABLE city_part DROP COLUMN CountryCode;</copy>
    ```
14. Optimize the table
    ```
    <span style="color:blue">mysql></span><copy>OPTIMIZE TABLE city_part;</copy>
    ```
    > **Note:** warning is expected: https://dev.mysql.com/doc/refman/8.0/en/optimize-table.html

15. Update table statistics
    ```
    <span style="color:blue">mysql></span><copy>ANALYZE TABLE city_part;</copy>
    ```
16. Exit MySQL
    ```
    <span style="color:blue">mysql></span><copy>\q</copy>
    ```
17. Create partitions:
    * Check the files for the city_part table on your disk
        ```
        <span style="color:green">shell-mysql1></span><copy>ls -l /mysql/data/world</copy>
        ```
    * Connect to <span style="color:green">shell-mysql1</span>

        ```
        <span style="color:green">shell></span><copy>mysql -uadmin -p -P3307 -hmysql1</copy>
        ```
    * Partition your table into 5 segments based on hash
        ```
        <span style="color:blue">mysql></span><copy>ALTER TABLE world.city_part PARTITION BY HASH (id) PARTITIONS 5;</copy>
        ```
    * Exit MySQL and Check the file of the city_part table on your disk
        ```
        <span style="color:blue">mysql></span><copy>\q</copy>
        ```
        ```
        <span style="color:green">shell-mysql1></span><copy>ls -l /mysql/data/world</copy>
        ```


## Task 2: Work with JSON
1. Create a database for JSON tests
    ```
    <span style="color:blue">mysql></span><copy>CREATE DATABASE json_test;</copy>
    ```
    ```
    <span style="color:blue">mysql></span><copy>USE json_test;</copy>
    ```
2. Create a JSON table
    ```
    <span style="color:blue">mysql></span><copy>CREATE TABLE jtest (id bigint NOT NULL AUTO_INCREMENT, doc JSON, PRIMARY KEY (id));</copy>
    ```
    ```
    <span style="color:blue">mysql></span><copy>DESC jtest;</copy>
    ```
3. Add data to this table 

    > **Note:** The second insert adds 2 records in one step

    ```
    <span style="color:blue">mysql></span><copy>INSERT INTO jtest(doc) VALUE('{"A": "hello", "b": "test", "c": {"hello": 1}}');</copy>
    ```
    ```
    <span style="color:blue">mysql></span><copy>INSERT INTO jtest(doc) VALUE('{"b": "hello"}'),('{"c": "help"}');</copy>
    ```
    ```
    <span style="color:blue">mysql></span><copy>SELECT * FROM jtest;</copy>
    ```
4. Retrieve json documents with these commands 

    > **Note** the shortcut “->” or “->>”


    ```
    <span style="color:blue">mysql></span><copy>SELECT json_extract (doc, "$.b") FROM jtest;</copy>
    ```
    ```
    <span style="color:blue">mysql></span><copy>SELECT doc->"$.b" FROM jtest;</copy>
    ```
    ```
    <span style="color:blue">mysql></span><copy>SELECT doc->>"$.b" from jtest;</copy>
    ```
5.  Create an index on the virtual column
    * You can search a record and check with “explain” that there is no usable index on our table
        ```
        <span style="color:blue">mysql></span><copy>select * from jtest where doc->>"$.b"='test';</copy>
        ```
        ```
        <span style="color:blue">mysql></span><copy>EXPLAIN select * from jtest where doc->>"$.b"='test';</copy>
        ```
        ![MYSQLEE](images/explain-jtest-query.png "explain query")
    * We add now an index on column “b”
        ```
        <span style="color:blue">mysql></span><copy>alter table jtest add column gencol CHAR(50) AS (doc->>"$.b");</copy>
        ```
        ```
        <span style="color:blue">mysql></span><copy>CREATE INDEX myvirtindex ON jtest(gencol);</copy>
        ```
    * And now we repeat the explain and see that now we automatically use the index
        ```
        <span style="color:blue">mysql></span><copy>EXPLAIN select * from jtest where doc->>"$.b"='test';</copy>
        ```
        ![MYSQLEE](images/explain-jtest-query-with-index.png "explain jtest query with index")



## Task 3: MySQL Document Store
> **Note:** We use the port for Xdev (<span style="color:red">33070</span>) instead of usual classic protocol port (<span style="color:red">3307</span>)


1. Please connect to MySQL Database via X Protocol
    ```
    <span style="color:green">shell></span><copy>mysqlsh -uadmin -hmysql1 -P 33070 -p</copy>
    ```
2. Create and use a test schema. 
    (We use javascript mode, but python is available also)

    ```
    <span style="color:blue">My</span><span style="color: orange">SQL </span><span style="background-color:yellow">JS</span>><copy>session.createSchema('docstore')</copy>
    ```
    ```
    <span style="color:blue">My</span><span style="color: orange">SQL </span><span style="background-color:yellow">JS</span>><copy>\use docstore</copy>
    ```
3. Now create and populate a small collection

    ```
    <span style="color:blue">My</span><span style="color: orange">SQL </span><span style="background-color:yellow">JS</span>><copy>db.createCollection('posts');</copy>
    ```

    ```
    <span style="color:blue">My</span><span style="color: orange">SQL </span><span style="background-color:yellow">JS</span>><copy>db.posts.add({"title":"MySQL 8.0 rocks", "text":"My first post!", "code": "42"})</copy>
    ```

    ```
    <span style="color:blue">My</span><span style="color: orange">SQL </span><span style="background-color:yellow">JS</span>><copy>db.posts.add({"title":"Polyglot database", "text":"Developing both SQL and NoSQL applications"})</copy>
    ```

4. Checking the built-in JSON validation
    ```
    <span style="color:blue">My</span><span style="color: orange">SQL </span><span style="background-color:yellow">JS</span>><copy>db.posts.add("This is not a valid JSON document")</copy>
    ```
5. Inspect the posts collection you have just created
    ```
    <span style="color:blue">My</span><span style="color: orange">SQL </span><span style="background-color:yellow">JS</span>><copy>db.posts.find()</copy>
    ```

    What can you notice? Did the system add something to content by itself?

    ```
    <span style="color:blue">My</span><span style="color: orange">SQL </span><span style="background-color:yellow">JS</span>><copy>db.posts.find().limit(1)</copy>
    ```

6. Modify existing elements of the collection
    ```
    <span style="color:blue">My</span><span style="color: orange">SQL </span><span style="background-color:yellow">JS</span>><copy>db.posts.modify("title = 'MySQL 8.0 rocks'").set("title", " MySQL 8.0 rocks!!!")</copy>
    ```
    ```
    <span style="color:blue">My</span><span style="color: orange">SQL </span><span style="background-color:yellow">JS</span>><copy>db.posts.find()</copy>
    ```
7. Check that that a collection is just a table with 2 columns: Index and JSON Document
    ```
    <span style="color:blue">My</span><span style="color: orange">SQL </span><span style="background-color:yellow">JS</span>><copy>session.sql("desc posts")</copy>
    ```
    ```
    <span style="color:blue">My</span><span style="color: orange">SQL </span><span style="background-color:yellow">JS</span>><copy>session.sql("show create table posts")</copy>
    ```
    ```
    <span style="color:blue">My</span><span style="color: orange">SQL </span><span style="background-color:yellow">JS</span>><copy>session.sql("select * from posts")</copy>
    ```
8. Therefore, it is possible to add indexes on specific JSON elements of the collection
    ```
    <span style="color:blue">My</span><span style="color: orange">SQL </span><span style="background-color:yellow">JS</span>><copy>db.posts.createIndex('myIndex', {fields: [{field:"$.title", type: "TEXT(20)"}]} )</copy>
    ```
    ```
    <span style="color:blue">My</span><span style="color: orange">SQL </span><span style="background-color:yellow">JS</span>><copy>session.sql("show create table posts")</copy>
    ```

## Acknowledgements
* **Author** - Marco Carlessi, Principal Sales Consultant
* **Contributors** -  Perside Foster, MySQL Solution Engineering, Selena Sánchez, MySQL Solutions Engineer
* **Last Updated By/Date** - Selena Sánchez, MySQL Solution Engineering, May 2023
