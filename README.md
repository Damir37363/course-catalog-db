# course-catalog-db

PostgreSQL schema for the course catalog (Lab 6, "Advanced Web Technologies").

## Constraint violation log

Commands were run with `\set VERBOSITY verbose` on PostgreSQL 16.15, from a clean state (`schema.sql` + `seed.sql`).

| # | What we violate | SQLSTATE | Constraint | API response |
|---|-----------------|----------|------------|--------------|
| 1 | NULL in courses.title | 23502 | NOT NULL (it has no name of its own: the message names the column `title`, field `COLUMN NAME`) | 422; if Pydantic let the empty value through, that is a bug in the code (500) |
| 2 | courses.credits = 0 | 23514 | `courses_credits_check` | 422 |
| 3 | courses.likes = -1 | 23514 | `courses_likes_check` | 500: only our code changes the counter (`likes + 1`), so it must never go negative; if it does, there is a bug in the code |
| 4 | a one-letter student name | 23514 | `students_name_check` | 422 |
| 5 | a duplicate student email | 23505 | `students_email_key` | 409 Conflict |
| 6 | a duplicate course id | 23505 | `courses_pkey` | 409 Conflict |
| 7 | a repeated enrollment in a course | 23505 | `enrollments_course_id_student_id_key` | 409 Conflict |
| 8 | enrolling in a nonexistent course | 23503 | `enrollments_course_id_fkey` | 404: course not found |
| 9 | enrolling a nonexistent student | 23503 | `enrollments_student_id_fkey` | 404: student not found (422 would also work; I chose 404 because the id points to a resource that does not exist) |
| 10 | deleting a course that has enrollments | 23503 on PostgreSQL 16 (23001 on PostgreSQL 18) | `enrollments_course_id_fkey` | 409 Conflict: a course with enrollments can't be deleted |
| 11 | deleting a student (cascade) | no error | `enrollments_student_id_fkey` (`ON DELETE CASCADE`) | not an error: the student's enrollments were deleted by cascade, 6 → 3 enrollments, and 6 again after `ROLLBACK` |

Notes:

- All ten commands ended with an error, and afterwards the tables still held 6 courses, 3 students and 6 enrollments: none of them changed the data.
- Errors are told apart by the code class (every constraint violation starts with `23`) and by the constraint name, not by the message text: the text and even the code of experiment 10 depend on the server version.
- The `Failing row contains (4, A, a@example.com)` line of experiment 4 shows that student number 4 was issued before the constraints were checked and is not returned to the counter after the error. A `GENERATED ALWAYS AS IDENTITY` column guarantees unique numbers, not the absence of gaps.
- Extra check: `INSERT INTO students (id, ...) VALUES (100, ...)` fails with SQLSTATE `428C9` ("cannot insert a non-DEFAULT value into column id", the column is `GENERATED ALWAYS`). This is not a user error but a bug in the code: the API would answer 500.

**Why check the name length in the database if Pydantic already checks it (`Field(min_length=2)`)?** The API is not the only thing that writes to the database: there are also other services, import scripts, migrations, manual fixes in `psql` and future versions of the API with their own bugs. A check in the code protects only the data that goes through that code, while a constraint in the database protects the data no matter who writes it and cannot be bypassed. Pydantic still stays useful: it rejects a bad request earlier and gives the client a clear 422 message, without a round trip to the database. So the rule is checked twice on purpose: the code is for a friendly answer, the database is for the guarantee.

## Index: EXPLAIN before and after

Query: `SELECT * FROM enrollments WHERE student_id = 777;`

Database `idx_demo` (the same schema, without the index at first): 500 courses, 20,000 students, 50,000 enrollments (`experiments/idx_data.sql`). PostgreSQL 16.15; the times depend on the computer, the plans do not.

| | Before the index | After the index |
|---|------------------|-----------------|
| Access method | `Seq Scan` on enrollments | `Bitmap Index Scan` on `enrollments_student_id_idx` + `Bitmap Heap Scan` |
| cost | 0.00..1042.00 | 4.31..15.59 |
| Rows Removed by Filter | 49997 | no filter: the 3 rows are found through the index |
| Execution Time (best of three runs) | 3.018 ms (4.671 / 3.131 / 3.018) | 0.014 ms (0.053 / 0.018 / 0.014) |

About 200 times faster. The index takes 896 kB next to the 3336 kB table (about a quarter of its size) and slows inserts, updates and deletes a little, because every change has to be reflected in the index too.

Before (`EXPLAIN ANALYZE`, third run):

```text
 Seq Scan on enrollments  (cost=0.00..1042.00 rows=3 width=34) (actual time=0.084..3.009 rows=3 loops=1)
   Filter: (student_id = 777)
   Rows Removed by Filter: 49997
 Planning Time: 0.034 ms
 Execution Time: 3.018 ms
```

After (`CREATE INDEX enrollments_student_id_idx ON enrollments (student_id);`, `EXPLAIN ANALYZE`, third run):

```text
 Bitmap Heap Scan on enrollments  (cost=4.31..15.59 rows=3 width=34) (actual time=0.005..0.007 rows=3 loops=1)
   Recheck Cond: (student_id = 777)
   Heap Blocks: exact=3
   ->  Bitmap Index Scan on enrollments_student_id_idx  (cost=0.00..4.31 rows=3 width=0) (actual time=0.003..0.003 rows=3 loops=1)
         Index Cond: (student_id = 777)
 Planning Time: 0.023 ms
 Execution Time: 0.014 ms
```

The other two plans from the experiment (before the index):

| Query | Plan | Where the index comes from |
|---|---|---|
| `students WHERE email = 'student777@example.com'` | `Index Scan using students_email_key` | the `UNIQUE` constraint creates an index by itself |
| `enrollments WHERE course_id = 'course-7'` | `Bitmap Index Scan on enrollments_course_id_student_id_key` | the index on the unique pair `(course_id, student_id)` also works for a search on the first column of the pair |

In the main database, a primary-key lookup on the six-row `courses` table still uses `Seq Scan` (cost 1.07): the whole table fits in one data page, so reading it is cheaper than going through the index.

Answers to the questions:

1. PostgreSQL creates an index by itself only for `PRIMARY KEY` and `UNIQUE`, because it needs one to check uniqueness quickly. `email` is `UNIQUE`, and `course_id` is the first column of the unique pair `(course_id, student_id)`, so both are covered. A `FOREIGN KEY` creates no index: the database cannot know which queries you are going to run. `student_id` is only the *second* column of the pair, and a B-tree sorted by `(course_id, student_id)` does not help to find a value of the second column alone, so the search had to read the whole table.
2. For six rows the whole table fits in one data page, and reading it sequentially (cost 1.07) is cheaper than first going through the index and then to the table. The planner compares costs, and "there is an index" does not mean "the index will be used". For 50,000 rows the sequential read costs 1042 and the index about 15, so the index wins clearly.
3. (a) The query "one student's enrollments" (`WHERE student_id = ...`), also a `JOIN` that starts from a student. (b) The cascading delete of a student: to delete the student's enrollments, the database has to find all rows with that `student_id`, and without the index it would read the whole `enrollments` table for every deleted student.
