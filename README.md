Postgresql trianing - Laurenz Albe

April. 27, 2020

* Read the docs: www.postgresql.org/docs/11/index.html
* Mailing list: www.postgresql.org/list/
  * pgsql-general
  * pgsql-hackers : development list
* Wiki: wiki.postgresql.org

Install
* postgresql-debuginfo package.
  * it doesn't hurt. doesn't slow things down.
  * if you get a crash from pg, then you can create a stack trace for anyone who needs to support you.
  * `perf` tool on linux can be used for performance monitoring.
* postgresql11-contrib
  * defined in appendix F.
  * always install this.
* user: postgres
  * dedicated user to run postgres
* create a database cluster
  * unfortunate name.
  * does not mean mutliple machines.
  * it's like an instance
  * it's a directory with files and a number of processes running against those files.

```text
  cluster

|----------|
| postgres |
|----------| -- (tcp:5432/unix) listen --
| course   |
|----------|
```

you cannot join tables between two different databases.

Shared between databases:

* users
* tablespaces

To create a cluster:

```bash
$ initdb -D $DATA_DIR -E UTF8 --locale=en_US.utf8
```

interesting options:

* --encoding=ENCODING (Always use UTF-8)
* --locale=LOCALE
* --lc-collate, --lc-ctype
  * determines which character is a space, digit etc.
  * collation: determines how string are sorted and ordered. (cannot be changed after db creation)
    * indexes are sorted lists. order is determined by collation for string data type
    * affects order by clauses based on collation in the database.
    * US-english. Or use C or posix locale. C locale is very fast.

* standard data dir: /var/lib/pgsql/11/data

To start:

```bash
$ pg_ctl -D /home/mokha/development/2020-04-pg-training/db/data -l logfile start
```

* 1 server process for each client connection.
  * good isolation

* `psql` is a command line client for connecting to the server.
  * 4 bits of info needed
    * host: -h
    * port: -p
    * database: -d
    * user: -U

```bash
psql -h 127.0.0.1 -p 5432 -U postgres -d postgres
```

How to quit:
  * \q
  * ^d

* \ -> command for the client
* everything else is sent to SQL interpreter
* version 11 supports `exit`

Defaults:
* -h: unix socket
* -U: logged in user
* environment variables
  * PGDATABASE
  * PGHOST
  * PGPORT
  * PGUSER

```sql
CREATE DATABASE course;
```
* creates a copy of an existing database. it uses `template1` database
* you can specify a different template database.

client commands:
  * \?
  * \watch
  * \i FILE
  * \ir RELATIVE_FILE
  * \h: help about client commands
  * \h CREATE DATABASE
  * \l: list databases
  * \d: describe

# Indexes

We need a table that has a certain size.
We'll create a table with a 1M rows.

```sql
CREATE TABLE test(id bigint GENERATED ALWAYS AS IDENTITY NOT NULL, name text NOT NULL);
```

* id is a autogenerated primary key column, without the primary key constraint.
* always use `bigint` which is an 8 byte integer.
  * prevent exhausting possible range of valid integers to choose from for identifier.
  * difference between SERIAL column
    * `CREATE TABLE test2 (id bigserial PRIMARY KEY);`
    * includes a sequence
    * default value is next item in sequence.
    * IDENTITY Column advantage.
      * manually inserting id's can cause collisions with SERIAL.
      * standards compliant so it's more portable between databases.
* `text` has a theoretical limit is 1GB
  * use `text` when the application doesn't have a limit.
  * avoid arbitrary limits like varchar(255).
  * why set limit if there is not limit.
  * there is not performance impact either.
  * nulls make queries difficult, queries are a little more complex which leads to perf issues.
  * recommends: use `not null`. Easy to go from `not null` to allow `null. Harder the other way`. Easy to go from `not null` to allow `null`. Harder the other way.

```sql
INSERT INTO test(name) VALUES ('hans'), ('laurenz');
INSERT INTO test(name) SELECT name FROM test;

TABLE test is like SELECT * FROM test;

CREATE INDEX test_id_idx ON test (id);
```

```psql
# SELECT * FROM test WHERE id = 42;
 id |  name
----+---------
 42 | laurenz
(1 row)

Time: 161.688 ms
[local:/home/mokha/development/2020-04-pg-training/tmp/sockets]/postgres=
# CREATE INDEX test_id_idx ON test (id);
CREATE INDEX
Time: 2106.364 ms (00:02.106)
[local:/home/mokha/development/2020-04-pg-training/tmp/sockets]/postgres=
# SELECT * FROM test WHERE id = 42;
 id |  name
----+---------
 42 | laurenz
(1 row)

Time: 1.682 ms
```

You describe how the result should look like.
The db will figure out how to best do that.

* uuid is 16 bytes wide.
* nothing wrong with it and go ahead and use it.
* nice for distributed generation of identifiers.

# query life cycle

1. query is parsed by parser for syntax.
2. query re-writer.
3. query planner or query optimizer. (AI component that tries to enumerate or walk through different possible ways to execute query.)

Prepend `EXPLAIN` to query to see execution plan

```sql
EXPLAIN SELECT * FROM test WHERE id = 42;
```


```sql
# EXPLAIN SELECT * FROM test WHERE id = 42;
                               QUERY PLAN
-------------------------------------------------------------------------
 Index Scan using test_id_idx on test  (cost=0.43..8.45 rows=1 width=14)
   Index Cond: (id = 42)
(2 rows)

Time: 0.536 ms
```

What do #'s mean?

* cost: no meaning in reality. estimate of cost. from how many rows determines how expensize pg thinks it will be.
* cost=0.43..8.45. initial cost to get first result.. total cost to get all rows.
* rows=1 how many rows it thinks it will return
* width=13 estimated with in bytes.


`\di+ to describe index`

```sql
# \dt+ test
                  List of relations
 Schema | Name | Type  | Owner |  Size  | Description
--------+------+-------+-------+--------+-------------
 public | test | table | mokha | 266 MB |
(1 row)


# \di+ test_id_idx
                          List of relations
 Schema |    Name     | Type  | Owner | Table |  Size  | Description
--------+-------------+-------+-------+-------+--------+-------------
 public | test_id_idx | index | mokha | test  | 135 MB |
(1 row)
```

It made the query faster but we pay a price for having the index.


divided into blocks

------------------
| table rows     | 8K block 0
------------------
| 2 |            | 8K block 1
------------------
| 1 |            | 8K block 2
------------------

tables are unordered.
updates, deletes will change order.
cannot rely on order of items in the table.
database table is also called a HEAP.
It's unordered.
It's a pile of rows.
Indexes are a different affair. sorted list of index items


Index is kept in order.

-----
| 1 | ---> points to a physical location of row in a datafile.
| 2 |
| 3 |

To maintain the order of an index, deletes, updates of data rows means
having to re-order the index. Insert, update and delete statements will
increase cost to maintain index. Indexes will negatively impact performance
for insert, update and deletes.

Who is responsible for indexes?

* do not believe this: `dba has to figure out performance bottleneck and figure out correct indexes.`
* poor dba doesn't understand the data in the table.
  * doesn't know what the data means.
  * doesn't know what has to be fast.
  * in some cases adding an index afterwards cannot improve a bad query.
* during development make sure to choose good indexes.

What are indexes are useful for?

Library visual

* table is a library of books
  * each row is a book
  * shelves are blocks
* library catalogue: ordered list of books (index)

* index can be used for a '<' condition.

```sql
# EXPLAIN SELECT * FROM test WHERE id < 42;
                                QUERY PLAN
--------------------------------------------------------------------------
 Index Scan using test_id_idx on test  (cost=0.43..9.15 rows=41 width=14)
   Index Cond: (id < 42)
(2 rows)

Time: 0.630 ms
```

```sql
# EXPLAIN SELECT * FROM test WHERE id > 4000000000;
                               QUERY PLAN
-------------------------------------------------------------------------
 Index Scan using test_id_idx on test  (cost=0.43..4.45 rows=1 width=14)
   Index Cond: (id > '4000000000'::bigint)
(2 rows)

Time: 0.571 ms
```

* b-tree indexes can be read in both directions. index wasn't used. why?
* pg can use the index but chooses not to use the index.

![index-scan](./index-scan.png)

Back and forth between index and table. This is fine when a few rows to scan.
When a large amount of rows is large. access pattern between heap and index is
rando I/O. Not as good as sequential I/O. Exceeding a large # of rows means
it's more efficient to just do a sequential scan rather than go back and forth between
index and datafiles.

```sql
# EXPLAIN SELECT min(id) FROM test;
                                              QUERY PLAN
------------------------------------------------------------------------------------------------------
 Result  (cost=0.47..0.48 rows=1 width=8)
   InitPlan 1 (returns $0)
     ->  Limit  (cost=0.43..0.47 rows=1 width=8)
           ->  Index Only Scan using test_id_idx on test  (cost=0.43..213123.91 rows=6291456 width=8)
                 Index Cond: (id IS NOT NULL)
(5 rows)

Time: 0.766 ms
```
