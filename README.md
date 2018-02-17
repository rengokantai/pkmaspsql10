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
