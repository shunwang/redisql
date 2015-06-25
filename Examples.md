# Redisql Examples #

**[Jump to examples for Mysql users](http://code.google.com/p/redisql/wiki/Examples#MYSQL_EXAMPLES)**<br />
**[Jump to examples for Redis users](http://code.google.com/p/redisql/wiki/Examples#REDIS_EXAMPLES)**<br />
**[Jump to examples for InSwapMode example](http://code.google.com/p/redisql/wiki/Examples#INSWAP_EXAMPLE)**<br />
<br />

Redisql provides a single home for your SQL and NOSQL data. Its event driven fast and you can effectively use all of your RAM (even go into swap if you arent scared) to store your data and have it retrieved consistently at RAM speed<br />

The following examples showcase how Redisql can
  * transform SQL data to NOSQL data structures (denormalise and go schemaless)
  * transfrom NOSQL data into SQL tables (normalise to save RAM or to backup to a RDBMS)
  * transfrom various NOSQL data structures to other NOSQL data structures (repackage your data to your use case)

<br />
## MYSQL EXAMPLES ##
### MYSQL EXAMPLE ONE: SQL tables at NOSQL speed ###
If your database is bottlenecking and you need NOSQL speed, that is what Redisql was built for<br />
Redisql is a full relational database and is VERY fast
  1. export table(s) to Redisql (e.g. w/ PHP function [importFromMysql()](http://github.com/JakSprats/predis/blob/master/lib/Predisql.php)),
  1. cutover your application to use an Redisql library (which speak SQL and support RDBMS functionality, meaning you simply switch libraries and maybe rename a few functions, if your current mysql library uses non standard function names)
  1. That's it, you are set
  1. The Redisql server can even be on a different machine if you want to free up resources for your mysql-server
Best to [read](http://code.google.com/p/redisql/wiki/CommandReference) up on the SQL that Redisql supports and what it does not support, if you are following best OLTP practices, you should be good to go.<br />
Also note that joining a Mysql table w/ a Redisql during runtime is not yet supported
<br /><br />

### MYSQL EXAMPLE TWO: Try NOSQL out ###
Curious about NOSQL? Want to try it out w/o the risk of data lock in, use Redisql. Redisql provides mechanisms to get your data into any Redisql/redis data-structure and get it back out, there is no risk of product data lock in. To go schemaless:<br />
  1. export you mysql table into redisql (w/ simple [scripts](http://github.com/JakSprats/predis/blob/master/examples/schemaless.php)),
  1. denormalise Redisql's tables into ANY redis datastructure (LIST,SET,ZSET,HASH) using the SELECT STORE syntax, which provides the possibilities to break up tables any way you want
```
SELECT ....
FROM table
WHERE ....
STORE redis_command redis_object
```
    * Hash Table "user\_pass" - lookup passwords directly using "name" (and the hash table is only as big as the results of the WHERE Clause)
```
SELECT name, passwd FROM user WHERE ... STORE HSET user_pass
```
    * ZSET "user\_action" - you can now perform set logic on user actions OR range query via timestamp
```
SELECT timestamp, action FROM user_history WHERE ... STORE ZSET user_action
```
    * mysql-to-redisql-export and table-denormalisation are demo'ed [here](http://github.com/JakSprats/predis/blob/master/examples/schemaless.php)
  1. Go hog wild w/ your new NOSQL Data. You are schemaless, you have lists, queues, sets, ordered sets, hash tables, message queues ....
  1. if you find NOSQL wasnt the right fit for your problem, then export your redis datastructures back into mysql by
    * converting them to a redisql table
```
CREATE TABLE back_to_sql AS DUMP redis_object
```
    * export from redisql to mysql w/ the DUMP TO MYSQL command (piped into mysql)
```
DUMP back_to_sql TO MYSQL
```
> > createTableFromRedisObject() and dumpToMysql() are demo'ed [here](http://github.com/JakSprats/predis/blob/master/examples/backup_redis_to_mysql.php)


### MYSQL EXAMPLE THREE: Export Table from Mysql To NOSQL and Go Schemaless ###
One of the most mind blowing features of NOSQL is it is SCHEMALESS. This means if you decide to give your users ten new attributes today and 20 new attributes tommorow and next week decide all of those attributes are stupid ... you can add and delete these attributes on-the-fly w/o updating your schema. No "ALTER TABLE ADD/DELETE COLUMN" w/ its unpredicable run-time behaviour. Go schemaless and add "columns" at will, cause they are no longer columns :)<br />
<br />
Steps to go from mysql to schemaless nosql
  1. Read out a table's contents from mysql and insert them into Redisql<br />

> A 50 line php example can be found [here](http://github.com/JakSprats/predis/blob/master/lib/Redisql.php), (function importFromMysql() lines 208-262)
  1. Denormalise the table using the command
```
DENORM table "table:*"
```
Thats it, you will now have one hash-table per sql-table-row with names "table:1, table:2, table:3, ....".<br />
You are schemaless. And for the meek/sensible, there are ways to reverse this process described [here](http://code.google.com/p/redisql/wiki/Examples#REDIS_EXAMPLE_ONE:_backup_redis_data_to_Mysql_for_Data_Mining_or), you can schema-up :)
A working PHP example, can be found [here](http://github.com/JakSprats/predis/blob/master/examples/schemaless.php) .. it is basically 2 function calls :)
<br />
<br />
<br />

## REDIS EXAMPLES ##
### REDIS EXAMPLE ONE: backup redis data to Mysql for Data Mining or Data Analytics ###
Redisql provides the ability to normalise a redis data structure (list,set,zset & hash) to a SQL table. It is then trivial to dump this table to file and have Mysql read it in and archive it. From there it is simple to do datamining on your redis objects by merging them into aggregate tables and performing relational logic on said tables.<br />
<br />
This can be done at command line with the commands: (the output of the 2nd command can be read straight into mysql)
```
CREATE TABLE X AS DUMP Redis_Object
DUMP X TO MYSQL
```
[Here](http://github.com/JakSprats/predis/blob/master/examples/backup_redis_to_mysql.php) is a 15 line php script that backs up EVERY KEY (that isnt a string:) in your redisDB to mysql


### REDIS EXAMPLE TWO: normalise redis keys to save memory ###
If your data in redis is composed of denormalised strings and you are getting low on memory, it is simple to normalise them into a Redisql table and save incredible amounts of memory, and in most use cases increase performance (e.g. because you only have to do one lookup for all of a single user's data, as opposed to 2+ in the denormalised case). If your redis data has reached a point where it can be put into a schema, storing the data in Redisql tables has many benefits.<br />
<br />
Redisql normalises redis keys into tables by searching the entire key space with wildcards and storing matching keys into the target normalised table.<br />
<br />
To normalise keys into a single table using a single wildcard, use the command:
```
NORM "user:*"
```
This will create the Redisql table "user" from all keys matching the "user:`*`" wildcard and create columns automatically (using the pattern "user:`*`:column")<br />

It is also possible to normalise a denormalised star schema of keys using several wildcards w/ the following command:
```
NORM "user:*" payment,address
```
This will create the Redisql tables "user\_address,user\_payment,user" from all keys matching the "user\_address:`*`,user\_payment:`*`,user:`*`" wildcards (in that order) and create columns automatically (using the patterns "user\_address:`*`:column, user\_payment:`*`:column, user:`*`:column")<br />
<br />
A script demonstrating these functionalities can be found [here](http://github.com/JakSprats/predis/blob/master/examples/pop_norm.php)<br />
<br />
In benchmarks, the memory savings from normalisation start at 300% and in a "normal" 4 column table, you can expect about 1000% memory improvements. Normalised data is inherently more compact.

### REDIS EXAMPLE THREE: create a Mysql Cache for "old" redis data ###
In Memory Databases are fantastic, until they get low on RAM, which is somewhat of an eventuality.<br />
<br />
Its a good practice to archive data you probably will not access again on the frontend from your In Memory Database to a disk based Database (e.g. mysql) and then build a thin cache layer that will retrieve archived data from the disk based Database as needed.<br />
<br />
Building the thin cache layer is surprisingly easy to program. Here is a PHP script that creates a Mysql Cache for the redis ZSET. The object can be found [here](http://github.com/JakSprats/predis/blob/master/examples/ZsetCache.php) _(only 100 lines)_ and an example using tweets can be found [here](http://github.com/JakSprats/predis/blob/master/examples/tweet/tweet_archiver.php).<br />
<br />
With this ZSetCache class, you need only to define your own achiving criterion and scripts, (specific to your data - some peoples data is old after 30 minutes, some after 30 days)<br />
<br />
The approach: cache-old-data-to-disk is a fantastic way to ensure your In Memory Database stays well within its RAM's size and can also be used to avoid spending money on hardware by avoiding the need to scale horizontally.
<br />
<br />
## INSWAP EXAMPLE ##
**coming soon**