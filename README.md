# pkmaspsql10
## Making Use of Indexes
```
INSERT INTO t_test (name) SELECT 'hans' FROM generate_series(1, 2000000); 
SELECT name, count(*) FROM t_test GROUP BY 1; 
\timing
```

### Making use of EXPLAIN
show the execution plan of a statement  
Four steps:
- The parser will check for syntax errors and obvious problems
- The rewrite system takes care of rules (views and other things)
- The optimizer will figure out how to execute a query in the most efficient way and work out a plan
- The plan provided by the optimizer will be used by the executor to finally create the result

PostgreSQL will use a parallel sequential scan.
In PostgreSQL 9.6 and 10.0, the number of parallel workers will be determined by the size of the table. The larger an operation is, the more parallel workers PostgreSQL will fire up. For a very small table, parallelism is not used as it would create too much overhead.  
To disable,
```
SET max_parallel_workers_per_gather TO 0; 
```
you can also decide the change in the ```postgresql.conf``` file,permanent change.

### Digging into the PostgreSQL cost model
If a query can be executed by the executor in many different ways, PostgreSQL will decide on the execution plan promising the lowest cost possible. 
```
SELECT pg_relation_size('t_test') / 8192.0; 
```
```
test=# SHOW cpu_tuple_cost;
cpu_tuple_cost
---------------- 
0.01(1 row)

test=# SHOW cpu_operator_cost;  
cpu_operator_cost
------------------- 
0.0025
```

```totalcost =  pg_relation_size('t_test') / 8192.0 + cpu_operator_cost*count + cpu_tuple_cost*count```



### Deploying simple indexes

```
CREATE INDEX idx_id ON t_test (id);  
\di+
```
remember indexes are not free, we have to write to all those indexes on INSERT, which seriously slows down the writing.

### Making use of sorted output
### Using more than one index at a time
PostgreSQL will go for a bitmap scan.

### Using bitmap scans effectively
use cases
- Avoiding using the same block over and over again
- Combining relatively bad conditions  

__In PostgreSQL 10.0, parallel bitmap heap scans are supported.__

### Improving speed using clustered tables
```
EXPLAIN (analyze true, buffers true, timing true)       SELECT *    FROM t_test    WHERE id < 10000; 
```
The idea is that analyze will not just show the plan but also execute the query and show us what has happened.


###### order by random
```
CREATE TABLE t_random AS SELECT * FROM t_test ORDER BY random();  
```
add vacuum
```
VACUUM ANALYZE t_random;  
```
tablename attname correlation
```
SELECT tablename, attname, correlation FROM pg_stats WHERE tablename IN ('t_test', 't_random') ORDER BY 1, 2; 
```

### Clustering tables
```CLUSTER``` that allows us to rewrite a table in the desired order.
Note:
- The CLUSTER command will lock the table while it is running. You cannot insert or modify data while CLUSTER is running. This might not be acceptable on a production system.  

example
```
CLUSTER t_random USING idx_random;  
```

### Making use of index only scans
index scan vs index only scan

### Combined indexes

### Adding functional indexes
```
CREATE INDEX idx_cos ON t_random (cos(id));
```
```
SELECT age('2010-01-01 10:00:00'::timestamptz); 
```

### Reducing space consumption
A b-tree will contain a pointer to each row in the table, and so it is certainly not free of charge.  
To figure out how much space an index will need:  
```
\di+
```
###### Create partial index
```
CREATE INDEX idx_name ON t_test (name) WHERE name NOT IN ('hans', 'paul'); 
\di+ idx_name 
```
### Adding data while indexing
CREATE INDEX CONCURRENTLY.  
Building the index will take a lot longer (usually at least twice as long), but you can use the table normally during index creation.
```
CREATE INDEX CONCURRENTLY idx_name ON t_test (name); 
```
### Creating new operators
```
CREATE OR REPLACE FUNCTION normalize_si(text) RETURNS text AS $$         
BEGIN         
RETURN substring($1, 9, 2) ||substring($1, 7, 2)||substring($1, 5, 2)||substring($1, 1, 4);
END; $$
LANGUAGE 'plpgsql' IMMUTABLE;
```



## 10. Making Sense of Backups and Replication
### Understanding the transaction log
Write Ahead Log (WAL) or xlog.

### Looking at the transaction log
```
/var/lib/pgsql/10/data/pg_xlog 
```
### Understanding checkpoints
Never touch the transaction log manually. 

### Configuring for archiving
Point-In-Time Recovery (PITR)
- You will lose less data because you can restore to a certain point in time and not just to the fixed backup point
- Restoring will be faster because indexes don't have to be created from scratch. They are just copied over and are ready to use
```
wal_level = replica    # used to be "hot_standby" in older versions for version prior 9.6 it is minimal
max_wal_senders = 10   # at least 2, better at least 2
```
Basically, there are two means of transporting the WAL:
- Using __pg_receivewal__ (up to 9.6 known as pg_receivexlog)
- Using filesystem as a means to archive
```
mkdir /archive
chown postgres.postgres archive
```
The following entries can be changed in the ```postgresql.conf``` file
```
archive_mode = on
archive_command = 'cp %p /archive/%f'
```







## 13. Migrating to PostgreSQL
```
SELECT * FROM generate_series(1, 4) AS x,
LATERAL (SELECT array_agg(y) FROM generate_series(1, x) AS y
) AS z; 
```
```
 x  | array_agg 
----+----------- 
 1  | {1} 
 2  | {1,2} 
 3  | {1,2,3} 
 4  | {1,2,3,4} 
 ```
