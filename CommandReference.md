# NOTE: Redisql is in the process of being renamed to "Alchemy Database", the new project home page can be found at #
# http://code.google.com/p/alchemydatabase/ #

## LEGACY ##


# REDISQL COMMANDS #

Redisql supports:
  1. An OLTP optimised subset of SQL
  1. Commands to transform NOSQL data to SQL and vice versa (dubbed MORPH Commands)
  1. embedded LUA w/ client-server functionality
  1. ALL redis 2.0 commands<br />

Supported SQL data types are currently UNSIGNED INT, TEXT, and FLOAT.<br />
Datetime should be represented as a UNSIGNED INT or FLOAT<br />
TEXT uses variable length storage and compression and can be substituted for any VARCHAR() or CHAR().<br />
UNSIGNED INT stores data using 2 bytes when possible.

## Supported SQL ##

### Create/Drop Operations: ###
  * `CREATE TABLE customer (id INT, group_id INT, name TEXT, phone text, age INT);`
  * `CREATE INDEX cust_group_ind ON customer (group_id);`
  * `DROP TABLE customer`
  * `DROP INDEX cust_group_ind`
  * `DESC customer` - provides detailed memory usage info
  * `DUMP customer` - dumps the table to client
  * `DUMP customer TO MYSQL` - dumps the table in mysql format

### Single Row Operations ###
  * `INSERT INTO customer VALUES (1,300,’John Doe’,’301-933-1891’,30);`
  * `SELECT group_id, phone FROM customer WHERE id = 1`
  * `DELETE FROM customer WHERE id = 1`
  * `UPDATE customer SET phone = ‘703-933-1891’, group_id = 4 WHERE id = 1`
  * `INSERT INTO customer VALUES (1,300,’John Doe’,’301-933-1891’,30) RETURN SIZE;` - returns the size of the row, table, and indices

### Index Operations ###
  * `SELECT group_id, phone FROM customer WHERE group_id BETWEEN 300 AND 400 [ORDER BY column LIMIT x]`
  * `SELECT group_id, phone FROM customer WHERE group_id IN (300, 301, 307, 311) [ORDER BY column LIMIT x]`
  * _Introducing_ the **Non-Relational Range Query** - the IN() can contain ANY redis command that returns rows (e.g. LRANGE, SINTER, ZREVRANGE, etc...)
> Get name and number of the 5 most recent active customers
```
SELECT name, phone FROM customer WHERE group_id IN (LRANGE L_IND_recent_calls 0 5) [ORDER BY column LIMIT x]
```
  * `DELETE FROM customer WHERE group_id BETWEEN 300 AND 400 [ORDER BY column LIMIT x]`
  * `UPDATE customer SET phone = ‘703-933-1891’, group_id = 4 WHERE group_id BETWEEN 300 AND 400 [ORDER BY column LIMIT x]`


### Joins ###
### Join via single Index ###
```
SELECT customer.name, group.slogan
 FROM customer, group
 WHERE customer.group_id = group.id AND
       group.id BETWEEN 300 AND = 400
       [ORDER BY table.column LIMIT x]
```

### Multi Index Join w/ Non-Relational IN() Clause ###
Get patient billing info for patients who got both treatment and prescriptions
```
SELECT patient.name, info.address, bill.amount
FROM patient, info, bill
WHERE patient.id = bill.patient_id AND
      patient.id = info.patient_id AND
      patient.id IN (SINTER treatment_set prescription_set)
      [ORDER BY table.column LIMIT x]
```

### Create table from Join ###
```
SELECT customer.name, group.slogan
FROM customer, group
WHERE customer.group_id = group.id AND
      group.id IN (300, 301, 307, 311)
STORE INSERT new_table
```

### Create table from Range Query ###
```
SELECT *
FROM customer
WHERE group_id BETWEEN 300 AND = 400
STORE INSERT new_table
```
An alternate syntax to Create Table from Range Query
```
CREATE TABLE new_table
AS SELECT * FROM customer WHERE group_id BETWEEN 300 AND = 400
```

### Full Table Scan ###
Find unindexed data – _not recommended, should be avoided when possible_
```
SCANSELECT id, phone FROM customer WHERE name = ‘bill’;
```
Full Table Scan w/ Range Query
```
SCANSELECT id, group, phone FROM customer WHERE age BETWEEN 18 AND 25
```
> _the SCANSELECT command makes it impossible to accidentally do a full table scan w/ SELECT_
<br />
## MORPH Commands ##
### Normalise redis keys into a SQL Table ###
  * NORM wildcard `[`wildcard2,wildcard3`]`
> All redis-keys matching "wildcard" are normalised into a SQL table
```
NORM "user:*"
```
    1. take ALL string KEYS matching the wildcard "user:`*`"
    1. create column\_names using the pattern "user:`*`:column\_name"
    1. the pk will be the part matching "`*`"
    1. put 1-3 together into a single Redisql table (w/ many rows) named "user".
    1. an example in PHP can be found [here](http://github.com/JakSprats/predis/blob/master/examples/pop_norm.php)
> Star schema normalisation: All redis-keys matching "user\_address:`*`,user\_payment:`*`,user:`*`" are normalised into the SQL tables "user\_address,user\_payment,user" respectively
```
NORM "user:*" payment,address
```
    1. This will create the Redisql tables "user\_address,user\_payment,user" from
    1. all keys matching the "user\_address:`*`,user\_payment:`*`,user:`*`" wildcards (in that order)
    1. creating columns automatically (using the patterns "user\_address:`*`:column, user\_payment:`*`:column, user:`*`:column")
    1. an example in PHP can be found [here](http://github.com/JakSprats/predis/blob/master/examples/pop_norm.php)

### Store the results of an Alsoql SELECT into a redis datastructure ###
  * SELECT ..... STORE REDIS\_COMMAND
```
SELECT name, salary
FROM employee
WHERE city = "san Francisco"
STORE HSET SanFranWorker
```
    1. create a redis Hash Table named `SanFranWorker` with all the employees from city "San Francisco"
    1. the values returned from the SELECTed column "name" will be the Hash Table's keys
    1. the values returned from the SELECTed column "salary" will be the Hash Table's values
> Given the SQL table `CREATE TABLE actionlist (id INT PRIMARY KEY, user_id INT, timestamp INT, action TEXT)`<br />
> The `$` on the end of `user_action_zset$` has a special functionality
```
SELECT user_id, timestamp, action
FROM actionlist
WHERE id BETWEEN 1 AND 20000
STORE ZADD user_action_zset$
```
    1. for each user\_id, a distinct ZSET will be created named "user\_action\_zset:1, user\_action\_zset:2", etc...
    1. each row will execute the command `ZADD user_action_zset${user_id} timestamp action`
    1. afterwards the command `ZREVRANGE user_action_zset:1 0 1` would return user 1's last two actions
    1. this functionality is demonstrated in this [Ruby script](http://github.com/JakSprats/redis-rb/blob/master/examples/action_list.rb)

### Create a Redisql table from the results of a redis command ###
  * CREATE TABLE table AS REDIS\_COMMAND
> NOTE: the ZSET tweets uses timestamp as its score
```
CREATE TABLE yesterdays_tweets AS ZRANGEBYSCORE tweets $two_days_ago $yesterday
```
    1. take the members of the ZSET from two\_days\_ago until yesterday
    1. store them in a Redisql table named "yesterdays\_tweets" (pk will be auto incremented INT)
    1. an example in PHP can be found in the example script [here](http://github.com/JakSprats/predis/blob/master/examples/tweet/tweet_archiver.php) that calls the ZSetCache class [here](http://github.com/JakSprats/predis/blob/master/examples/ZsetCache.php)

### Denormalise a Redisql table into one redis hash-table per table row ###
  * DENORM table denorm\_wildcard
```
DENORM user "user:*"
```
    1. denormalise the "user" table into an individual hash-table per row, named "user:pk" (e.g. user:1, user:2)
    1. the keys of these hashes will be the column-names of the "user" table.
> this is a simple mechanism to "Go Schemaless", an example script in PHP showcasing this can be found [here](http://github.com/JakSprats/predis/blob/master/examples/schemaless.php)

<br />
## LUA Command ##
Redisql has embedded [LUA](http://www.lua.org/about.html) <br />
[IBM's lua description](http://www.ibm.com/developerworks/linux/library/l-embed-lua/index.html): _The Lua programming language is a small scripting language specifically designed to be embedded in other programs. Lua’s C API allows exceptionally clean and simple code both to call Lua from C, and to call C from Lua”_<br />
Redisql has embedded Lua in its C-server and C in its embedded Lua :)<br />
The speed of embedded LUA commands in Redisql is impressive: [benchmark](http://groups.google.com/group/redisql-dev/msg/e24027e82e5c2094)<br />

### Simple LUA ###
```
LUA "x=45;return x;"
```
### embedded LUA "client()" function ###
The lua function "client()" will call the Redisql server internally as if it were a client call.<br />
Command to set a user's last\_login and return his status
```
LUA "client('SET','user:123:last_login',1289597410); return client('GET', 'user:123:status');"
```
### helper.lua ###
The file [helper.lua](https://github.com/JakSprats/Redisql/blob/master/helper.lua) contains a full Redisql client library, defining simple versions of Redisql commands: e.g. get(), set(), zadd(), hset(), create\_table(), select(), update(), norm()<br />
When helper.lua's functions have been loaded into Redisql, the above command could be simplified as
```
LUA "set('user:123:last_login',1289597410); return get('user:123:status');"
```
Loading helper.lua into the Redisql server can be done either by specifiying the "luafilename" in the config file "redisql.conf" or by the following command
```
./redisql-cli CONFIG SET luafilename helper.lua
```
The "CONFIG SET luafilename" also provides a simple way to modify lua functions in Redisql w/o server restarts. Modify the commands in helper.lua and then load them, this will **reset Lua's state** and the new functions will be in place.<br />
### Global Lua Variable Server ###
This form of embedding Lua extends Redisql to be a global lua variable server. Lua variables have global scope, so if one client sets them another can read them.<br />
[Lua tables](http://lua-users.org/wiki/TablesTutorial) are a very flexible and useful datatype in Lua<br />
Here is a command sequence from different clients accessing the same Lua table:
```
LUA "user_123_t = { 1,1,2,3,5,8,13 }; return 'SET lua table';"
LUA "return user_123_t[4];"
```
NOTE: for those requiring enhanced Lua performance, Redisql's Makefile has an option (and instructions) to link to libluajit instead of liblua.
<br />
## REDIS Commands ##
redis supports commands to deal with many data structures, including (strings, sets, lists, sorted-sets, and hash tables) - [full list](http://code.google.com/p/redis/wiki/CommandReference)<br />
redis project home page **[here](http://code.google.com/p/redis/)**<br />
Extensive documentation on redis commands can be found [here](http://code.google.com/p/redis/wiki/CommandReference)<br />
<br />
The following is an IFRAME of redis' command reference page, complete w/ links describing each command<br />
&lt;wiki:gadget url="http://www.allinram.info/alsosql/gadget.xml"  border="0" width="1200" height="4050"/&gt;

### SQL NOTES ###
  1. Column ordering in INSERT statements not yet supported
  1. Table aliasing in JOIN statements not yet supported