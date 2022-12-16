# Correlated and Uncorrelated Subquery

Sub-queries are queries within another query. The result of the inner sub-query is fed to the outer query, which uses that to produce the result. If that outer query is itself the inner query to a further query then the query will continue until the final outer query completes.

There are two types of sub-queries: Correlated sub-queries and Uncorrelated sub-queries.

### Example data

Assume we have two tables: `students` and `grades`.

`students` table contains three columns: id, class_id and name.

| Column name | Type   |
|-------------|--------|
| id          | number |
| class_id    | number |
| name        | string |

`grades` table contains three columns: subject, student_id, grade

| Column name | Type   |
|-------------|--------|
| subject     | string |
| student_id  | number |
| grade       | number |

### Uncorrelated sub-queries

The uncorrelated sub-query is a type of sub-query where inner query doesn't depend upon the outer query for its execution.
It can complete its execution as a standalone query.

Example: Get the grade of students named `Alice`. Here is the query:

```sql
SELECT * FROM grades where student_id IN (SELECT id FROM students WHERE name = "Alice")
```

The inner query `SELECT id FROM students WHERE name = "Alice"` is uncorrelated query because it can be executed standalone.

### Correlated query

The correlated query is a type of sub-query where inner query depends upon the outcome of the outer query in order to perform its execution.

Example: Get name of the students that their grades are greater than average grade of their class.

```sql
SELECT s.name
FROM students s
    JOIN grades g on s.id = g.student_Id
WHERE g.grade > (
        SELECT AVG(grades.grade)
        FROM grades
            JOIN students ON students.id = grades.student_id
        WHERE student.class_id = s.class_id
    )
```

The inner query in this case is correlated sub-query because it use `s.class_id` from outer query to complete its execution.
