# PSQL test with a 40 million users 
Compare the speed of selection/insertion using different indexes (no index, btree, hash)

Result:
| Index type | Execution Time | Result | Insertion time (synchronous_commit = on) | Result | Insertion time (synchronous_commit = of) | Result |
|-----------------|-----------------|-----------------|-----------------|-----------------|-----------------|-----------------|
| No index | 955.997 ms/967.199 ms | slowest | 5.528 ms | medium | 0.573 ms | fastest |
| BTREE | 18.681 ms/23.183 ms | medium | 2.221 ms | fastest | 0.762 ms| medium |
| HASH | 16.091 ms/18.914 ms | fastest  | 13.124 ms | slowest |  6.533 ms| slowest | 


STEPS:
# 1: Create DB
```
CREATE DATABASE users;
CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,
    username VARCHAR(100),
    date_of_birth DATE
);
```
# 2: Fill with data
```
INSERT INTO users (username, date_of_birth)
SELECT
    'User ' || gs.n,
    '1970-01-01'::date + (random() * (current_date - '1970-01-01'::date))::int
FROM
    generate_series(1, 40000000) AS gs(n);
```
# 3: Selection and insertion without index
```
1 Attempt
users=# EXPLAIN ANALYZE SELECT  * FROM users WHERE date_of_birth = '2019-06-19';
                                                        QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------
 Gather  (cost=1000.00..464313.17 rows=2029 width=21) (actual time=0.229..955.686 rows=1990 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   ->  Parallel Seq Scan on users  (cost=0.00..463110.27 rows=845 width=21) (actual time=2.470..946.035 rows=663 loops=3)
         Filter: (date_of_birth = '2019-06-19'::date)
         Rows Removed by Filter: 13332670
 Planning Time: 0.098 ms
 Execution Time: 955.997 ms

2 Attempt

users=# EXPLAIN ANALYZE SELECT  * FROM users WHERE date_of_birth = '1971-11-11';
                                                        QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------
 Gather  (cost=1000.00..464313.17 rows=2029 width=21) (actual time=4.856..966.910 rows=2057 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   ->  Parallel Seq Scan on users  (cost=0.00..463110.27 rows=845 width=21) (actual time=3.300..954.498 rows=686 loops=3)
         Filter: (date_of_birth = '1971-11-11'::date)
         Rows Removed by Filter: 13332648
 Planning Time: 0.118 ms
 Execution Time: 967.199 ms

3 Insertion

EXPLAIN ANALYZE INSERT INTO users (username, date_of_birth)
SELECT
    'User ' || gs.n,
    '1970-01-01'::date + (random() * (current_date - '1970-01-01'::date))::int
FROM
    generate_series(1, 5) AS gs(n);
                                                          QUERY PLAN
------------------------------------------------------------------------------------------------------------------------------
 Insert on users  (cost=0.00..0.26 rows=0 width=0) (actual time=5.485..5.486 rows=0 loops=1)
   ->  Subquery Scan on "*SELECT*"  (cost=0.00..0.26 rows=5 width=226) (actual time=2.825..2.845 rows=5 loops=1)
         ->  Function Scan on generate_series gs  (cost=0.00..0.18 rows=5 width=36) (actual time=0.066..0.075 rows=5 loops=1)
 Planning Time: 1.071 ms
 Execution Time: 5.528 ms
(5 rows)
```
# 4: Add BTREE index
```
CREATE INDEX btree_index_date_of_birth ON users USING BTREE (date_of_birth);
```

# 5: Selection and insertion with BTREE index

```
1 Attempt

users=# EXPLAIN ANALYZE SELECT  * FROM users WHERE date_of_birth = '2019-06-19';
                                                               QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on users  (cost=24.16..7593.53 rows=2029 width=21) (actual time=1.023..18.385 rows=1990 loops=1)
   Recheck Cond: (date_of_birth = '2019-06-19'::date)
   Heap Blocks: exact=1981
   ->  Bitmap Index Scan on btree_index_date_of_birth  (cost=0.00..23.66 rows=2029 width=0) (actual time=0.521..0.521 rows=1990 loops=1)
         Index Cond: (date_of_birth = '2019-06-19'::date)
 Planning Time: 0.321 ms
 Execution Time: 18.681 ms

2 Attempt
users=# EXPLAIN ANALYZE SELECT  * FROM users WHERE date_of_birth = '1971-11-11';
                                                               QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on users  (cost=24.16..7593.53 rows=2029 width=21) (actual time=1.012..22.911 rows=2057 loops=1)
   Recheck Cond: (date_of_birth = '1971-11-11'::date)
   Heap Blocks: exact=2048
   ->  Bitmap Index Scan on btree_index_date_of_birth  (cost=0.00..23.66 rows=2029 width=0) (actual time=0.443..0.447 rows=2057 loops=1)
         Index Cond: (date_of_birth = '1971-11-11'::date)
 Planning Time: 2.448 ms
 Execution Time: 23.183 ms

3 Insertion

users=# EXPLAIN ANALYZE INSERT INTO users (username, date_of_birth)
SELECT
    'User ' || gs.n,
    '1970-01-01'::date + (random() * (current_date - '1970-01-01'::date))::int
FROM
    generate_series(1, 5) AS gs(n);
                                                          QUERY PLAN
------------------------------------------------------------------------------------------------------------------------------
 Insert on users  (cost=0.00..0.26 rows=0 width=0) (actual time=2.141..2.142 rows=0 loops=1)
   ->  Subquery Scan on "*SELECT*"  (cost=0.00..0.26 rows=5 width=226) (actual time=0.091..0.116 rows=5 loops=1)
         ->  Function Scan on generate_series gs  (cost=0.00..0.18 rows=5 width=36) (actual time=0.015..0.026 rows=5 loops=1)
 Planning Time: 0.109 ms
 Execution Time: 2.221 ms
(5 rows)
```

# 6: Drop index and create HASH index

```
DROP INDEX IF EXISTS btree_index_date_of_birth;
CREATE INDEX hash_index_date_of_birth ON users USING HASH (date_of_birth);
```

# 7: Selection with HASH index
```
1 Attempt
users=# EXPLAIN ANALYZE SELECT  * FROM users WHERE date_of_birth = '2019-06-19';
                                                               QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on users  (cost=63.72..7633.09 rows=2029 width=21) (actual time=2.361..15.755 rows=1990 loops=1)
   Recheck Cond: (date_of_birth = '2019-06-19'::date)
   Heap Blocks: exact=1981
   ->  Bitmap Index Scan on hash_index_date_of_birth  (cost=0.00..63.22 rows=2029 width=0) (actual time=1.819..1.819 rows=1990 loops=1)
         Index Cond: (date_of_birth = '2019-06-19'::date)
 Planning Time: 2.773 ms
 Execution Time: 16.091 ms

2 Attempt
users=# EXPLAIN ANALYZE SELECT  * FROM users WHERE date_of_birth = '1971-11-11';
                                                               QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on users  (cost=63.72..7633.09 rows=2029 width=21) (actual time=2.909..18.548 rows=2057 loops=1)
   Recheck Cond: (date_of_birth = '1971-11-11'::date)
   Heap Blocks: exact=2048
   ->  Bitmap Index Scan on hash_index_date_of_birth  (cost=0.00..63.22 rows=2029 width=0) (actual time=2.175..2.175 rows=2057 loops=1)
         Index Cond: (date_of_birth = '1971-11-11'::date)
 Planning Time: 0.112 ms
 Execution Time: 18.914 ms

3 Insertion
users=# EXPLAIN ANALYZE INSERT INTO users (username, date_of_birth)
SELECT
    'User ' || gs.n,
    '1970-01-01'::date + (random() * (current_date - '1970-01-01'::date))::int
FROM
    generate_series(1, 5) AS gs(n);
                                                          QUERY PLAN
------------------------------------------------------------------------------------------------------------------------------
 Insert on users  (cost=0.00..0.26 rows=0 width=0) (actual time=13.077..13.078 rows=0 loops=1)
   ->  Subquery Scan on "*SELECT*"  (cost=0.00..0.26 rows=5 width=226) (actual time=0.232..0.292 rows=5 loops=1)
         ->  Function Scan on generate_series gs  (cost=0.00..0.18 rows=5 width=36) (actual time=0.014..0.042 rows=5 loops=1)
 Planning Time: 0.122 ms
 Execution Time: 13.124 ms
(5 rows)

```

