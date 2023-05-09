# Using index to optimize query

We went through some definition when using explain. Now we will use explain to analyze how index help improving query time.

First, we will index the `grade` column.

```sql
CREATE INDEX grade_asc ON grades (grade ASC);
CREATE INDEX class_asc ON grades (class ASC);
```

## Query with index

Let's start with a simple query:

```sql
test=# explain analyze select * from grades where grade = 5;
                                                          QUERY PLAN                                                          
------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on grades  (cost=1073.97..9629.48 rows=96200 width=29) (actual time=4.582..23.935 rows=100334 loops=1)
   Recheck Cond: (grade = 5)
   Heap Blocks: exact=7353
   ->  Bitmap Index Scan on grade_asc  (cost=0.00..1049.92 rows=96200 width=0) (actual time=3.531..3.531 rows=100334 loops=1)
         Index Cond: (grade = 5)
 Planning Time: 0.056 ms
 Execution Time: 26.336 ms
(7 rows)
```

Compare with the result of this query without index, the planner decided to use two-step plan. You can think that, without index, the planner only use 1, now it uses two, so it maybe be slower. But, read carefully in the result, you can see that the total time to fetch all rows is way better than without index. (9629 ms with 19853 ms)
Now we will analyze the result.

- In first step, the planner's going to visit an index named `grade_asc` to find the locations of rows matching the index condition. Total time to fetch all records is 1049 ms.
- In second step, the planner will go to the table and fetch needed rows. After step 1, the planner will have a list of physical address of rows that match with conditions, but it doesn't go to each physical address to get value, it will sort the addresses first, and then fetch pages that contain the addresses, and of course, page contains the invalid rows, so you can see that in this step, there is `Recheck Cond` to re-check condition. This is why in this query, the planner using two-step plan, to minimize the cost of separate fetches.

## Does planner always use index?

Check this:

```sql
test=# explain analyze select * from grades where grade > 8;
                                                          QUERY PLAN                                                           
-------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on grades  (cost=2222.16..12061.82 rows=198933 width=29) (actual time=9.033..49.874 rows=199366 loops=1)
   Recheck Cond: (grade > 8)
   Heap Blocks: exact=7353
   ->  Bitmap Index Scan on grade_asc  (cost=0.00..2172.42 rows=198933 width=0) (actual time=7.597..7.598 rows=199366 loops=1)
         Index Cond: (grade > 8)
 Planning Time: 0.073 ms
 Execution Time: 56.940 ms
(7 rows)
                                                              
test=# explain analyze select * from grades where grade > 2;
QUERY PLAN                                                   
----------------------------------------------------------------------------------------------------------------
 Seq Scan on grades  (cost=0.00..19853.00 rows=798967 width=29) (actual time=0.008..78.880 rows=799012 loops=1)
   Filter: (grade > 2)
   Rows Removed by Filter: 200988
 Planning Time: 0.045 ms
 Execution Time: 96.779 ms
(5 rows)
```

Assume query with condition `grade > 8` is query A, the other is query B. The different between two queries is the condition. Why planner will use `Index Scan` for query A but `Seq Scan` for query B?
To answer this question, we will make a statistic for the rate of each grade.

```sql
test=# select count(*) from grades;
  count  
---------
 1000000
(1 row)

test=# select count(*) from grades where grade > 8;
 count  
--------
 199366
(1 row)

test=# select count(8) from grades where grade > 2;
 count  
--------
 799012
(1 row)
```

We have total 1000000 records, 199366 records have grade > 8 and 799012 records have grade > 2. The number of records in case `grade > 8` is quite little compare with the total records, so using index scan will reduce alot of time. But, in case `grade > 2`, the records is quite large, so using index scan doesn't reduce so much time (maybe it will take more time because after you get physical addresses, you need to sort them to fetch pages contain correct values), so in this case, the planner decide not to use `Index Scan`.

So, using index or not depends on the rate between the estimate number of rows that match with condition and the total rows.