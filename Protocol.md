# Client/Server Protocol #

Redisql uses the NOSQL data-store redis' client/server protocol. Redis has a very simple, powerful, and efficient ASCII protocol for transferring data via TCP, UDP, and Unix Domain Sockets.
<br />

Redis protocol is specified [here](http://code.google.com/p/redis/wiki/ProtocolSpecification). This is the starting point for learning about the protocol, and this is where the protocol is defined.
<br />

Redisql's commands basically have the same syntax, except SQL Keywords are required, so clients need to insert them where appropriate.<br />

Redisql requests use redis' [unified request protocol](http://code.google.com/p/redis/wiki/ProtocolSpecification#The_new_unified_request_protocol).<br />
Redisql responses are 100% redis compliant (e.g. CREATE TABLE -> +OK, SELECT `*` FROM table -> Multi-Bulk)

## List of number of arguments and placement of SQL keywords for Redisql commands ##
  1. CREATE TABLE has 4 arguments "CREATE TABLE table\_name (col1 type,col2 type,,,)"
  1. DROP TABLE has 2 arguments "DROP TABLE table\_name"
  1. DESC TABLE has 1 argument "DESC table\_name"
  1. DUMP TABLE has 1 argument "DUMP table\_name"
  1. CREATE INDEX has 6 arguments "CREATE INDEX index\_name ON table (column)"
  1. DROP INDEX has 2 arguments "DROP INDEX index\_name"
  1. INSERT has 5 arguments "INSERT INTO table VALUES (val1,val2,,,)"
  1. SELECT has 6 arguments "SELECT column1,column2,,, FROM tbl1,tbl2, WHERE where\_clause"
  1. UPDATE has 6 arguments "UPDATE table SET col1=val1,col2=val2,,, WHERE where\_clause"
  1. DELETE has 5 arguments "DELETE FROM table WHERE where\_clause"
  1. SCANSELECT has 6 arguments "SCANSELECT column1,column2,,, FROM tbl WHERE where\_clause"
  1. NORM has 2 arguments "NORM main\_wildcard secondary\_wildcard,,,," - 2nd is optional
  1. DENORM has 2 arguments "DENORM table\_name main\_wildcard"
  1. LUA has 1 arguments "LUA lua\_cmd"

Optional Syntaxes
  1. DUMP can also dump to Mysql format, where it has 5 aguments "DUMP table\_name TO MYSQL mysql\_table\_name"
  1. SCANSELECT can also have 4 arguments when it has no where-clause "SCANSELECT column1,column2,,, FROM tbl"


### EXAMPLES ###
The following examples do not explicitly write out the "\r\n" at the end of each line, but the protocol does require them.

**SELECT** Example:
```
SELECT user.name, user_info.address FROM user, user_info WHERE user.id = user_info.user_id AND user.id = 7
```
would be transformed into the following in the redis line protocol:
```
*6
$6
SELECT
$28
user.name, user_info.address
$4
FROM
$15
user, user_info
$5
WHERE
$43
user.id = user_info.user_id AND user.id = 7
```
  * The first line is the number of arguments
  * Lines that start w/ $X, mean that the next line has X bytes
  * that is the whole protocol, it is straightforward.
<br />
This protocol seems to have a lot of overhead when compared to a binary protocol. In practice the overhead the protocol produces is not that significant when network overhead and various operating system page copy steps are taken into account. The upside is, it is human readable ... writing clients is a breeze, you can even tap into data streams using the linux _tcpdump_ command.
<br />

**CREATE TABLE** example
```
*4
$6
CREATE
$5
TABLE
$10
table_name
$40
(id INT primary key, name TEXT, age INT)
```

**INSERT** example:
```
*5
$6
INSERT
$4
INTO
$5
table
$6
VALUES
$48
(a, b, c, d, e, f, g, h, i, j, k, l, m, n, o, p)
```

**UPDATE** example:
```
*6
$6
UPDATE
$4
table
$5
SET
$37
a=1, b = 2, c=3, d=4, e=5, f=6, g = 7
$5
WHERE
$6
id = 7
```

This redis protocol has been implemented [in over 15 languages in redis](http://code.google.com/p/redis/wiki/SupportedLanguages), so supporting Redisql ONLY requires adding 14 commands to these clients, as Redisql has not introduced any changes to the core protocol.<br />

Ported libraries for Redisql include
  1. redisql-rb: Ruby client http://github.com/JakSprats/redis-rb (fork of redis-rb)
  1. NODE.JS client http://github.com/JakSprats/node_Redisql (patch to node\_redis) - actively maintained
  1. Predisql: PHP 5.3 CLIENT: http://github.com/JakSprats/predis (a fork of Predis)
  1. Embedded Lua: https://github.com/JakSprats/Redisql/blob/master/helper.lua

### NOTES ###
  * in the future, the Ruby and PHP clients will follow the example of the Node.js cilents and become very small patchs that are 100% on top of the redis clients.
  * **SELECT** is used by both redis (select DB) and Redisql (select `*` from table where id = 1) in different capacities. Clients must either override redis' default SELECT functionality to accept either 1 (e.g. select DB) or 5 arguments (e.g. select `*` from table where id = 1) OR come up w/ a two different commands for redis' and Redisql' select functionality (the former is recommended, but requires overriding and is more complex)