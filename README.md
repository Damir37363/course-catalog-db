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
