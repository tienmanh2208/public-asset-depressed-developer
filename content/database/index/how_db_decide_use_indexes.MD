# How the database optimizer decides to use indexes

We have a table with this information:

```sql
test=# \d grades;
                                   Table "public.grades"
 Column |         Type          | Collation | Nullable |              Default               
--------+-----------------------+-----------+----------+------------------------------------
 id     | integer               |           | not null | nextval('grades_id_seq'::regclass)
 name   | character varying(50) |           | not null | 
 grade  | integer               |           | not null | 
 class  | integer               |           | not null | 
Indexes:
    "class_asc" btree (class)
    "grade_asc" btree (grade)
    "grades_id_key" UNIQUE CONSTRAINT, btree (id)
    "name_asc" btree (name)
    "name_asc_include_grade" btree (name) INCLUDE (grade)

test=# select count(*) from grades;
count  
---------
 1000000
(1 row)
```

The pattern of query we will test:

```sql
select * from <table> where <column1> = ? and <column2> = ?
```

## Case table doesn't use index

If the estimated rows are too large (compare with total rows), the planner decides not to use index because it's not worth to use that plan.

```sql
test=# explain analyze select * from grades where class > 2 and grade > 2;
                                                   QUERY PLAN                                                   
----------------------------------------------------------------------------------------------------------------
 Seq Scan on grades  (cost=0.00..23334.00 rows=666529 width=33) (actual time=0.014..78.045 rows=666192 loops=1)
   Filter: ((class > 2) AND (grade > 2))
   Rows Removed by Filter: 333808
 Planning Time: 0.132 ms
 Execution Time: 90.667 ms
(5 rows)
```

Because the number of rows having `class > 2 and grade > 2` is quite large (about 666529), so the planner only uses Seq Scan for this query.

## Case table uses both two indexes

If the number of rows for condition 1 and condition 2 is huge, the planner will use both two indexes of two columns to filter data.

```sql
test=# explain analyze select * from grades where class in (2,3,4,5) and grade in (3,6,7);
                                                              QUERY PLAN                                                               
---------------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on grades  (cost=6710.53..16939.69 rows=101075 width=33) (actual time=34.343..205.991 rows=99744 loops=1)
   Recheck Cond: ((grade = ANY ('{3,6,7}'::integer[])) AND (class = ANY ('{2,3,4,5}'::integer[])))
   Heap Blocks: exact=8334
   ->  BitmapAnd  (cost=6710.53..6710.53 rows=101075 width=0) (actual time=32.659..32.661 rows=0 loops=1)
         ->  Bitmap Index Scan on grade_asc  (cost=0.00..3199.03 rows=305300 width=0) (actual time=15.407..15.407 rows=299479 loops=1)
               Index Cond: (grade = ANY ('{3,6,7}'::integer[]))
         ->  Bitmap Index Scan on class_asc  (cost=0.00..3460.71 rows=331067 width=0) (actual time=16.844..16.844 rows=332683 loops=1)
               Index Cond: (class = ANY ('{2,3,4,5}'::integer[]))
 Planning Time: 1.152 ms
 Execution Time: 210.090 ms
(10 rows)
```

Because condition `class in (2, 3, 4, 5)` has about 305300 results and condition `grade in (3, 6, 7)` has bout 331067 results, it's quite huge, so the planner used bot two indexes `grade_asc` and `class_asc` to filter data.

## Case table uses only one indexes

If there is any condition that has number of results is acceptable small (compare to total rows), the planner will run that condition first (using the index), and the data will filter that result by another condition.

```sql
test=# explain analyze select * from grades where class = 2 and grade in (3, 6, 7);
                                                         QUERY PLAN                                                         
----------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on grades  (cost=901.93..10568.97 rows=25045 width=33) (actual time=6.156..38.445 rows=25172 loops=1)
   Recheck Cond: (class = 2)
   Filter: (grade = ANY ('{3,6,7}'::integer[]))
   Rows Removed by Filter: 57801
   Heap Blocks: exact=8334
   ->  Bitmap Index Scan on class_asc  (cost=0.00..895.67 rows=82033 width=0) (actual time=4.483..4.483 rows=82973 loops=1)
         Index Cond: (class = 2)
 Planning Time: 0.120 ms
 Execution Time: 39.475 ms
(9 rows)

test=# explain analyze select * from grades where class in (2, 3, 4, 5) and grade = 3;
                                                          QUERY PLAN                                                          
------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on grades  (cost=1135.97..11278.30 rows=34210 width=33) (actual time=5.584..38.106 rows=33445 loops=1)
   Recheck Cond: (grade = 3)
   Filter: (class = ANY ('{2,3,4,5}'::integer[]))
   Rows Removed by Filter: 66435
   Heap Blocks: exact=8334
   ->  Bitmap Index Scan on grade_asc  (cost=0.00..1127.42 rows=103333 width=0) (actual time=4.228..4.228 rows=99880 loops=1)
         Index Cond: (grade = 3)
 Planning Time: 0.109 ms
 Execution Time: 39.299 ms
(9 rows)
```

You can see that two above queries only use 1 index to scan to get data.

The first query, the condition `class = 2` only has about 82033 valid rows, so the planner decided to use index `class_asc` to get the data satisfied condition `class = 2`.
And then, the database will use this result to filter the other condition `grade in (3, 6, 7)`.

The second query, the condition `grade = 3` only has about 103333 valid rows, so the planner will use `grade_asc` to filter data with `grade = 3` first, after that, the database will use that result to filter the other condition `class in (2, 3, 4, 5)`.

**NOTE** Why the planner knows if a condition returns many data or not to decide to use suitable plans? That's because the database does its own statistic about the data we saved. So it'll know how much data may pass a condition (Of course it's not 100% right, but mostly it is).