# Basic definition

Series này sẽ tập trung tìm hiểu và thảo luận về index trong SQL. Chúng ta sẽ nói về index là gì, cách tổ chức data trong index, index hoạt động ra sao, làm thế nào để có thể sử dụng index hiệu quả.

Trong series này, Postgres sẽ được sử dụng làm môi trường test chính, thỉnh thoảng các hệ quản trị cơ sở dữ liệu khác có thể được sử dụng để so sánh với nhau.

Các định nghĩa cơ bản cần nắm được:

- `explain analyze <query>` Dùng để phân tích câu query mình đang chạy. Câu lệnh này sẽ **ước tính** xem câu query này nên sử dụng plan nào, bao nhiêu record có thể bị scan qua, thời gian thực thi có thể là bao lâu

## Install tools

I'm gonna use image `postgres` version `14-alpine` for this tutorial.

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


