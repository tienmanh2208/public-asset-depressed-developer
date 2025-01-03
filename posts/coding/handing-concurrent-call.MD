# Handling concurrent calls

Handling concurrent calls is an essential part of logic BE, it maintains data integrity.
There are many use cases that need to handle concurrent calls like order products, transfer money, ...

This post will discuss some of the cases I have encountered.

## 1. Many request update the same resources

Assume you have a table user_balances, that stores balance of users.

Table structure:

| Column name | Type        |
|-------------|-------------|
| id          | int4        |
| user_id     | int4        |
| balance     | int8        |
| created_at  | timestamptz |
| updated_at  | timestamptz |

Assume you have multiple requests (both deduct and add) to the balance of a user, how can we update the balance so it'll have the exact amount eventually?

The normal flow would be:

// Image illustrates flow get balance, add, and update to db

Request 1 wants to add r1 to balance of user.
Request 2 wants to add r2 to balance of user.

Assume request 1 come first. The flow will be:
1. Request 1 get balance user (= b)
2. There will be two parallel actions:
   1. Request 1 update balance user in server (= b + r1)
   2. Request 2 get balance of user to update (= b because request 1 not update to db yet)
3. Request 1 update balance to db (= b + r1)
4. Request 2 calculate balance (= b + r2)
5. Request 2 update balance to db (= b + r2)

As you can see, if we use this flow, the actual balance (= b + r2) is different with expected balance (= b + r1 + r2).

To prevent this from happening, we need to change how we update the balance by updating directly to DB without getting it to server to calculate.

The query will be:

```sql
update balances set balance = balance + r1 where user_id = 1
```

This way, DB will handle the concurrent update for you, and in the end, balance of user will be `b + r1 + r2` as expected.

Talk more about how DB handles this case: When the first query comes, the db will lock the row being updated, until the transaction commits, it will execute the next query that updates the new value to that row.
So assume, your transaction has about 10 queries (the query update is in 3rd), DB will execute the rest 7th transaction before letting another query update the row. It's like you uses queue to handle the updating balance.

There is another solution for this problem is lock the row you are about to update it.
You can use `select ... for update`, so the db will prevent other to GET the balance UNTIL you commit your transaction.
But this approached will prevent others from reading data, so it may block many other reading requests.

## 2. Throw error when the resource has been updated

Assume you have a table `events`, and if there are multiple requests want to update a row in table, you only let the first one update it, the rest will be rejected.
How can you handle this situation?

Solution for this is to use `updated_at` field to determine if that event is updated or not. We treat updated_at as a version of row, because whenever we update data in events, the updated_at will be changed to record the update time.
So, we just need to make sure that the version we are updating is the version we are having by compare updated_at we have with updated_at in database.

Flow will be:

1. Get event from `events` table
2. Run query update event record with common condition AND `updated_at` = updated_at you get at the previous step.
3. If returned updated_row = 0, means `updated_at` in db is different with our updated_at, so we may throw error or retry update if needed.

NOTE: If you're using Date in Node.js, you may face a problem that even you have the latest version of row, the updated_row still = 0. Why is that?

It's because the precision of Date in Node.js is different with precision with timestamps in DB (in my case is Postgres).

The Date object in Node.js only saves up to 3 digits of precision for seconds; otherwise, timestamptz datatype in db saves up to 6 digits of precision for seconds.

So, if data in the database is "2024-12-25 12:32:21.125456", the value of Date object after casting this value is "2024-12-25 12:32:21.125".
So when you compare these two values, the condition will return false, hence updated_row = 0.

To solve this problem, you may extract timestamp up to milliseconds to compare in SQL.

SQL:

```sql
update events
set ...
where
    ...
    and floor(extract(epoch from updated_at) * 1000) = ${eventUpdatedAt.getTime()}
```

## 3. Throw error when data of entire table changed

Assume you have a table `events` with following structure:

| Column name      | Type                     | Nullable |
|------------------|--------------------------|----------|
| id               | serial                   | FALSE    |
| name             | varchar(255)             | TRUE     |
| start_time       | timestamp with time zone | TRUE     |
| end_time         | timestamp with time zone | TRUE     |
| display_end_time | timestamp with time zone | TRUE     |

constraint:
- start_time <= end_time <= display_end_time
- time ranges must not be overlapped

How can you archive this constraint when multiple requests create/edit event at the same time?
Take a moment to think about the solution.

Here is my solution:

What is the hardest part of this constraint? It's the `time ranges must not be overlapped`. The constraint `start_time <= end_time <= display_end_time` is easy to maintain with some logic in the server.

Let's assume there is two request come to create event with `start_time = '2024-01-01'` and `display_end_time = '2025-01-01'`.
So, the expected result would be 1 request will create successfully and the other is failed.

But, if these requests come to server mostly simultaneous, we may face the same problem with [Many request update the same resources](#1-many-request-update-the-same-resources).
So, it comes to my thinking that, instead of checking if a row is updated while I'm analyzing the data, I'll check if the entire table is updated or not.
Of course, you can still use the old solution by creating a table, storing the last time data of event table is updated, but with that, we have to create new table, and I dont want that.

Okay, we had the idea, but how to implement it? To implement this solution, we need to know the last time data in the events table is updated.
We can archive this by using query: `select max(updated_at) from events`, right?

So, in the update query, we have to compare the `updated_at` we are having with the result of above SQL.
If affected row = 0, it means data in the table has been updated, and we should re-analyze data before insert/update to prevent the overlapping of time ranges.

Another thing is, we can not use `where` when insert data, so we need to use a trick that we insert data from `select` statement, in that select, we will inject where condition of us to make sure data does not change.

The query to insert will be:

```sql
insert into events (...) select (...) where (select floor(extract(epoch from max(updated_at)) * 1000) from events) = ${maxUpdatedAt.getTime()}
```
