# Basic definition

This series is focused on discussing the index in SQL.
We're going to talk about: What is index, how data is structured in the index,
how index works, how to optimize queries using index.

I'm going to use `Postgres` as the main test environment, sometimes I use another DBMS to compare them.

Before reading this series,
I recommend you to understand some basic knowledge about database internals by reading this post:
[Understand database internals](https://depressed-developer.vercel.app/detail/database/4)

We will use `explain analyze <query>` to analyze the query.
This command will show us which plan the planner is going to use, estimation execution time and return records.

NOTE: `explain analyze` command will actually runs the query, so, all the information in response will be the actual result (like time to filter, time to validate, scanned rows). So, with `UPDATE`, `DELETE`, `INSERT` statement, you should push the explain query into a transaction in case you dont want to update/lose your data.

```sql
begin;
explain analyze ...;
rollback;
```

## Install tools

I'm going to use image `postgres` version `14-alpine` for this tutorial.

You can create a container running in your localhost to test:

```shell
docker run --name postgres_test -e POSTGRES_PASSWORD=1 -d postgres:14-alpine
docker exec -it postgres_test bash
psql -U postgres
CREATE DATABASE test;
\c test

# If you already have database test, you can access into test database by:
psql -U postgres test # Access database named test
```

```sql
# Create table
CREATE TABLE grades (
  id SERIAL UNIQUE NOT NULL,
  name VARCHAR(50) NOT NULL,
  grade INT NOT NULL,
  class INT NOT NULL
);

INSERT INTO grades (name, grade, class)
SELECT
  left(md5(random()::text), 20),
  floor(random() * 10 + 1)::int,
    floor(random() * 12 + 1)::int
from generate_series(1, 1000000);
```

Verify data generated in the table:

```sql
test=# select count(*) from grades;
  count  
---------
 1000000
(1 row)
```


