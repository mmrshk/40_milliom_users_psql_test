# 40_milliom_users_psql_test
Compare speed of insert using different indexes (no index, btree, hash)

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
# 3: Selection without index
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
```
# 4: Add BTREE index
```
CREATE INDEX btree_index_date_of_birth ON users USING BTREE (date_of_birth);
```

# 5: Selection with BTREE index

```
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
```

# 6: Drop index and create HASH index

```
DROP INDEX IF EXISTS btree_index_date_of_birth;
CREATE INDEX hash_index_date_of_birth ON users USING HASH (date_of_birth);
```

# 7: Selection with HASH index
```
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
```

