# Understanding bitmap scan

## Problem

As we may know:

- *Seq Scan*: (Sequence Scan) is a plan that the planner will go through all pages (also called heap) sequentially to find the valid rows. So even if there is a few records that matched the condition, we still need to check all records. (*Note*: Seq Scan is much faster than random I/O access)
- *Index Scan*: is a plan that the planner is using index to find the desired rows, this way is very fast to find the indexed value, but to get all the information related to that value, you need to go back to heap and get that data from heap page. So, the flow would be: When scanning the index (typically B-Tree structure), whenever the planner found a matched value, it will have index value (includes exact location of data in heap page: page number and offset of that value in page). After having that location, the planner will immediately go fetch that data from heap pages. So for each match, the planner will go to heap page to get data. If the number of matches is small, there is no problem here, but if not, this kind of getting data is called random I/O access. And you can see that there maybe a change the planner will go to one page many times. You can image the brief flow will be: **Fetch from index -> Fetch from heap page -> Fetch from index -> Fetch from heap page -> ... (so on). This causes random page access and hence random I/O**

So you can see that `Seq Scan` is slow, and `Index Scan` can be fast, but it's still maybe slow if the number of values matched condition is huge.

From this problem, `Bitmap scan` was born to solve this problem.

## Bitmap Index Scan and Bitmap Heap scan

You can see a reply from Postgres team in here [Ref](https://www.postgresql.org/message-id/12553.1135634231@sss.pgh.pa.us)

> A bitmap scan fetches all the tuple-pointers from the index in one go, sorts them using an in-memory "bitmap" data structure, and then visits the table tuples in physical tuple-location order.
> The bitmap scan improves locality of reference to the table at the cost of more bookkeeping overhead to manage the "bitmap" data structure --- and at the cost that the data is no longer retrieved in index order, which doesn't matter for your query but would matter if you said ORDER BY.
> 
> If the bitmap gets too large we convert it to "lossy" style, in which we only remember which pages contain matching tuples instead of remembering each tuple individually.  When that happens, the table-visiting phase has to examine each tuple on the page and recheck the scan condition to see which tuples to return.

You can imagine a `bitmap` is a single **bit array**, the length will equal to number of heap pages, each bit is represent for a page, if that bit is `1` (true), that means the page is represented by that bit has value that matched with condition, otherwise that bit is `0` (false).

So, how planner use `bitmap` to optimize searching data?

### Case query only has one condition

If this case, the planner will use `Bitmap Index Scan` plan with the index, the result is a bitmap where the pages that might contain condition is true.
And then, the planner will use `Bitmap Heap Scan` plan to go through heap pages sequentially to fetch the pages that might contain value satisfied condition (The input of this plan is the bitmap from `Bitmap Index Scan`), and recheck values in that page to get the valid result. So the planner still access heap pages sequentially, but only need to check the pages is marked `1` in bitmap.

Example:

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

The planner firstly use `Bitmap Index Scan` to know what pages might contain satisfied value. And then, the planner will use `Bitmap Heap Scan` to fetch pages that might have satisfied value (using bitmap array from previous step), and the planner have to `Recheck Cond` because these pages may contain invalid rows.

### Case query has more than one condition

Assume you are searching for `x = 1` and `y = 2`, you have indexes on `x` and `y`.

The planner does not combine two condition into one bitmap right away. It split them into two different bitmap. And then, base on the condition `AND` or `OR`, these two bitmaps will be ANDed or ORed with each other (In this case, you want to find x = 1 **and** y = 2 so two bitmaps will be ANDed), finally we will have one bitmap satisfied both condition.

Illustration:

Assume data is saved follow this table:

| Page | Data in page                  |
|------|-------------------------------|
| 0    | x = 1, y = 2<br/>x = 2, y = 1 |
| 1    | x = 5, y = 6<br/>x = 2, y = 1 |
| 2    | x = 8, y = 9<br/>x = 1, y = 9 |
| 3    | x = 5, y = 6<br/>x = 2, y = 3 |
| 4    | x = 5, y = 6<br/>x = 6, y = 2 |
| 5    | x = 1, y = 6<br/>x = 6, y = 1 |

Bitmap for condition `x = 1` will be:

|           |       |   |       |   |   |       |
|-----------|-------|---|-------|---|---|-------|
| **Index** | 0     | 1 | 2     | 3 | 4 | 5     |
| **Value** | **1** | 0 | **1** | 0 | 0 | **1** |

Bitmap for condition `y = 2` will be:

|           |       |   |   |   |       |   |
|-----------|-------|---|---|---|-------|---|
| **Index** | 0     | 1 | 2 | 3 | 4     | 5 |
| **Value** | **1** | 0 | 0 | 0 | **1** | 0 |

After 

Check this sql:

```sql
test=# explain analyze select * from grades where grade in (2, 5, 6) and class in (2, 7, 3, 5);
                                                              QUERY PLAN                                                               
---------------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on grades  (cost=6670.81..16874.15 rows=99698 width=33) (actual time=18.786..34.063 rows=100441 loops=1)
   Recheck Cond: ((grade = ANY ('{2,5,6}'::integer[])) AND (class = ANY ('{2,7,3,5}'::integer[])))
   Heap Blocks: exact=8334
   ->  BitmapAnd  (cost=6670.81..6670.81 rows=99698 width=0) (actual time=17.830..17.831 rows=0 loops=1)
         ->  Bitmap Index Scan on grade_asc  (cost=0.00..3158.03 rows=300900 width=0) (actual time=10.447..10.447 rows=300269 loops=1)
               Index Cond: (grade = ANY ('{2,5,6}'::integer[]))
         ->  Bitmap Index Scan on class_asc  (cost=0.00..3462.69 rows=331333 width=0) (actual time=7.047..7.047 rows=333282 loops=1)
               Index Cond: (class = ANY ('{2,7,3,5}'::integer[]))
 Planning Time: 0.187 ms
 Execution Time: 36.033 ms
(10 rows)
```