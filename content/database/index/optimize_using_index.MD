# Using index to optimize query

We went through some definition when using explain. Now we will use explain to analyze how index help improving query time.

First, we will index the `grade` column.

```sql
CREATE INDEX grade_asc ON grades (grade ASC);
CREATE INDEX class_asc ON grades (class ASC);
CREATE INDEX name_asc ON grades (name ASC);
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

## Index Only Scan

`Index Only Scan` means the database using index to find the information satisfied condition and do not need to go back to heap pages to get extra information, all needed information is already in the index.
Example: You index the `name` column, and you only want to get `name` information (of course with condition of column `name`), the planner is going to use `Index Only Scan` because the info `name` you want to get is already in the index.

Check two queries:

```sql
test=# explain analyze select name from grades where name = 'a';
                                                      QUERY PLAN                                                       
-----------------------------------------------------------------------------------------------------------------------
 Index Only Scan using name_asc on grades  (cost=0.42..4.44 rows=1 width=21) (actual time=7.815..7.816 rows=0 loops=1)
   Index Cond: (name = 'a'::text)
   Heap Fetches: 0
 Planning Time: 0.090 ms
 Execution Time: 7.858 ms
(5 rows)

test=# explain analyze select * from grades where name = 'a';
                                                    QUERY PLAN                                                    
------------------------------------------------------------------------------------------------------------------
 Index Scan using name_asc on grades  (cost=0.42..8.44 rows=1 width=33) (actual time=0.047..0.047 rows=0 loops=1)
   Index Cond: ((name)::text = 'a'::text)
 Planning Time: 0.109 ms
 Execution Time: 0.077 ms
(4 rows)
```

The only difference between these two queries is the `select` statement. The first query is `select name`, the second query is `select *` (get all information of the rows).
You can see that the planner using two different plan to run these queries.
- The first query using `Index Only Scan` because you only want `name` info, and it's on the index. So, no need to go back to table to get other data.
- The second query using `Index Scan` because after find the info satisfied query condition, you want all information of the row (includes name, grade, class, id). The index doesn't contain these info so database need to go back to table to fetch related info for matched rows.

The `Index Only Scan` will be faster than `Index Scan` because it doesn't need to go back to table to get extra info.

**What if** you still want to get extra info (like you want to query by `name` but get info of `name` and `grade`) and using `Index Only Scan`, not `Index Scan`?

There is a way to do that, is you create an index based on `name` column, and you include the grade info to that index.

```sql
create index name_asc_include_grade on grades (name asc) include (grade);
```

Before creating the index:

```sql
test=# explain analyze select name, grade from grades where name = 'a';
                                                    QUERY PLAN                                                    
------------------------------------------------------------------------------------------------------------------
 Index Scan using name_asc on grades  (cost=0.42..8.44 rows=1 width=25) (actual time=0.040..0.041 rows=0 loops=1)
   Index Cond: ((name)::text = 'a'::text)
 Planning Time: 0.093 ms
 Execution Time: 0.077 ms
(4 rows)
```

After creating the index:

```sql
test=# explain analyze select name, grade from grades where name = 'a';
                                                             QUERY PLAN                                                              
-------------------------------------------------------------------------------------------------------------------------------------
 Index Only Scan using name_asc_include_grade on grades  (cost=0.42..4.44 rows=1 width=25) (actual time=0.041..0.041 rows=0 loops=1)
   Index Cond: (name = 'a'::text)
   Heap Fetches: 0
 Planning Time: 0.244 ms
 Execution Time: 0.066 ms
(5 rows)
```

The drawback of this plan is it's going increase the size of the index (it's kind of obvious because you push extra info to the index). But, notice that the database load all index to memory to process search function. So if the size of the index is too large, it'll create more IO, so it can slow down your query.

```sql
test=# \di+ name_asc*
                                                List of relations
 Schema |          Name          | Type  |  Owner   | Table  | Persistence | Access method | Size  | Description 
--------+------------------------+-------+----------+--------+-------------+---------------+-------+-------------
 public | name_asc               | index | postgres | grades | permanent   | btree         | 39 MB | 
 public | name_asc_include_grade | index | postgres | grades | permanent   | btree         | 47 MB | 
(2 rows)
```

You can see that if we only index `name` column, the size of the index is 39 MB, but if you include grade to the index, the size is 47 MB. **Be careful when using this feature!**

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