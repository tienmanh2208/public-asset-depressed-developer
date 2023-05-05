# Understanding query planner and optimizer

## Where

Firstly, we will analyze the query to get all student with grade 5:

```sql
test=# explain analyze select * from grades where grade = 5;
                                                  QUERY PLAN                                                   
---------------------------------------------------------------------------------------------------------------
 Seq Scan on grades  (cost=0.00..19853.00 rows=96200 width=29) (actual time=0.010..58.765 rows=100334 loops=1)
   Filter: (grade = 5)
   Rows Removed by Filter: 899666
 Planning Time: 0.046 ms
 Execution Time: 60.563 ms
(5 rows)
```

This is example response from query planner when you analyze a query.

- Seq Scan is scanning data plan to filter data. There are 3 different plans DBMS may use depending on situation.
  - Sequential scans: Scan all records in the table
  - Index scans: Scan records using index
  - Bitmap index scans
- `cost <Estimated start-up cost>..<Estimated total cost>`: The cost has two numbers
  - The first number is the time the DBMS thinks it may need to fetch the first page (Maybe zero). If this number is huge, it means you're doing alot of stuffs before fetching data. In the example above, time to get the first page is 0.00, and 
  - The second number is total time the DBMS may need to fetch all the result.
- `rows`: Estimated number of rows output
- `width`: Estimated average width of rows output (in bytes)
- `Filter`: This is the conditions in where clause. This means, when it scans through each row, it will check, only rows that pass the condition will be outputted.

In the result above, you can see:

- DBMS using `Seq Scan` plan for this query. So, it'll scan all 1000000 rows and filter out valid rows.
- Time to fetch the first page may be 0.00, and time to fetch all result maybe is 19853ms.
- Estimated return rows is 96200
- Estimated average width of rows output is 29 bytes.

- If you run a query the first time, if you run the exact same query after that, you can see the execution time will be much more faster than the first one because of caching technique of DBMS

## Where and sort

We will sort `name` column in ascending order, and see how it goes:

```sql
test=# explain analyze select * from grades where grade = 5 order by name asc;
                                                            QUERY PLAN                                                             
-----------------------------------------------------------------------------------------------------------------------------------
 Gather Merge  (cost=16625.84..25979.20 rows=80166 width=29) (actual time=74.854..104.903 rows=100334 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   ->  Sort  (cost=15625.82..15726.03 rows=40083 width=29) (actual time=66.495..68.293 rows=33445 loops=3)
         Sort Key: name
         Sort Method: quicksort  Memory: 3885kB
         Worker 0:  Sort Method: quicksort  Memory: 3623kB
         Worker 1:  Sort Method: quicksort  Memory: 2990kB
         ->  Parallel Seq Scan on grades  (cost=0.00..12561.33 rows=40083 width=29) (actual time=0.020..28.105 rows=33445 loops=3)
               Filter: (grade = 5)
               Rows Removed by Filter: 299889
 Planning Time: 0.162 ms
 Execution Time: 107.044 ms
(13 rows)
```

You can see that there is alot going on here.

The planner has decided to use a three-step plan. In these case, we'll read from bottom to up direction.

- Firstly, the planner will scan the table to get rows that pass the condition in filter. It takes 0 ms to start, and about 12561 to fetch all the results. Estimated number of rows is 40083 with average length is 29 bytes. After this step, all the valid rows will be loaded into memory to be executed in next step.
- Secondly, the planner will sort all the rows by `name` field. In this process, the planner uses two worker to sort data (It will split rows to 2 chunk, and sort them independently). Look at the cost, we can see that it's gonna take 15625 ms to finish the sorting action (this amount of time includes time in first step, that means, actually, the planner only needs 15625 - 12561 = 3064 to finish the sorting action). The time finish is 15726, it's quite similar to start-up cost because all the data is already in the memory, no need to fetch them from databases.
- The final step, the planner will merge the results of two independent workers into one results.

## Optimize using limit

How limit help the query? Let's explain the query:

```sql
test=# explain analyze select * from grades where grade = 5 order by name asc limit 10;
                                                               QUERY PLAN                                                                
-----------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=14427.54..14428.70 rows=10 width=29) (actual time=56.225..58.154 rows=10 loops=1)
   ->  Gather Merge  (cost=14427.54..23780.89 rows=80166 width=29) (actual time=56.224..58.150 rows=10 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         ->  Sort  (cost=13427.51..13527.72 rows=40083 width=29) (actual time=53.322..53.324 rows=8 loops=3)
               Sort Key: name
               Sort Method: top-N heapsort  Memory: 26kB
               Worker 0:  Sort Method: top-N heapsort  Memory: 26kB
               Worker 1:  Sort Method: top-N heapsort  Memory: 26kB
               ->  Parallel Seq Scan on grades  (cost=0.00..12561.33 rows=40083 width=29) (actual time=0.029..45.292 rows=33445 loops=3)
                     Filter: (grade = 5)
                     Rows Removed by Filter: 299889
 Planning Time: 0.091 ms
 Execution Time: 58.202 ms
(14 rows)
```

You can see that, if we add `limit` to our query, the planner will add fourth step is Limit to limit the number of return rows. We should concentrate on the start-up time of the Limit, it's 14427 ms, and it's exactly the same with start-up cost of Gather Merge step, It's not waiting for the `Gather Merge` end, Because we only takes `n` values, so the `Limit` step will start with `Gather Merge` step to optimize the performance of the query. The process will stop when there are enough rows. You can see the total cost of `Limit` step is way less than the total cost of `Gather Merge` step because it doesn't need to wait the entire result set has been sorted. 

```sql
test=# explain analyze select * from grades where grade = 5 order by name asc offset 5000 limit 100;
                                                               QUERY PLAN                                                                
-----------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=16813.51..16825.18 rows=100 width=29) (actual time=96.854..99.824 rows=100 loops=1)
   ->  Gather Merge  (cost=16230.14..25583.49 rows=80166 width=29) (actual time=94.583..99.682 rows=5100 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         ->  Sort  (cost=15230.12..15330.32 rows=40083 width=29) (actual time=91.245..91.395 rows=2379 loops=3)
               Sort Key: name
               Sort Method: top-N heapsort  Memory: 1058kB
               Worker 0:  Sort Method: top-N heapsort  Memory: 1061kB
               Worker 1:  Sort Method: top-N heapsort  Memory: 1060kB
               ->  Parallel Seq Scan on grades  (cost=0.00..12561.33 rows=40083 width=29) (actual time=0.024..38.525 rows=33445 loops=3)
                     Filter: (grade = 5)
                     Rows Removed by Filter: 299889
 Planning Time: 0.064 ms
 Execution Time: 99.985 ms
(14 rows)
```

If we add offset to the query, you can see that the start-up cost of `Limit` step is not the same with `Gather Merge` because it needs to skip the first 5000 result before fetching the first valid result. But it's still better than waiting for `Gather Merge` step to finish.