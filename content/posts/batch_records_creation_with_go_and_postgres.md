+++
date = '2025-04-10T10:00:00+09:00'
draft = false
title = 'Batch records creation with Go and Postgres'
tags = ['postgres','psql','pg_bulkload','golang','pgx','sqlc']
+++

Let's talk about 2 problems in this post.
- batch insert in Postgres
- batch insert in SQLC (with Golang)

## Resource preparation

To simplify, we will use an **users** table with 4 fields: id(text, primary key), name(text), created_at and updated_at.
The insert data can be generated with [**pg-batch-play**](https://github.com/HoangNguyen689/pg-batch-play) tool.

```bash
# generate 1M users data in SQL format with single INSERT command

go run main.go genbatchcreateuser -s 1000000 -f sql -t singular
```

Besides, we will also use **psql** tool to observe, let's enable the timing option.

```psql
db=# \timing
Timing is on.
```

## Batch insert in Postgres

### INSERT command

We can use INSERT command to insert many records to Postgres.

```sql
INSERT INTO users VALUES
    ('1', 'User 1'),
    ('2', 'User 2'),
    ('3', 'User 3');
```

The above command is fast with small number of records. How about 1 million user records?

```psql
db=# \i batch_create_1000000_users.sql

psql:_out/batch_create_1000000_users.sql:1: server closed the connection unexpectedly
        This probably means the server terminated abnormally
        before or while processing the request.
psql:_out/batch_create_1000000_users.sql:1: error: connection to server was lost
make: *** [ssh/db] Error 2
```

I have run the Postgres server with 2GB memory and 6vCPUs.
The Postgres server received a signal kill. When observing the **docker stats**, there has been slightly changed in memory usage.
Let's increase the memory to 8GB and try again.

```psql
db=# \i batch_create_1000000_users.sql

INSERT 0 1000000
Time: 6114.668 ms (00:06.115)
```

About 6 seconds. Not so bad. Technicaly, we total can use INSERT to do the task. With powerful hardware, the limitation is expanded.
Of course, Postgres has [the hard limit](https://www.postgresql.org/docs/current/limits.html). But before reaching the 1GB command limitation, the hardware limitations likely will be reached first.

Who concerns about the command string limitation can read the code here:
- [StringInfo](https://github.com/postgres/postgres/blob/master/src/include/lib/stringinfo.h)
- [MaxAllocSize](https://github.com/postgres/postgres/blob/master/src/include/utils/memutils.h#L43)

The INSERT command has long time execution and the limitation about the command string (as well as the hardware). To gain more performance, we can use COPY command or 3rd party tool like pg_bulkload.

### COPY command

The COPY command is native supported in Postgres. It is more efficient than INSERT command, because:
- It doesn't need to parse the SQL syntax. Just read the data file directly.
- It read the file by block, consume less memory (eliminate the hardware limitation).
- It can disable WAL to move faster.

### pg_bulkload

The pg_bulkload is a 3rd party tool. To use it, we need to install it along side with Postgres.
It even faster than COPY command in case of large data set.
- It doesn't call the SQL command, ignore the parse and planner.
- It directly write to table heap file, not through the Postgres engine.
- It can bypass WAL,
- No trigger, no constraints, no index when loading.

Of course, the data must be clean. With WAL off, there's no way to rollback. So we need to be careful.

### Benchmark

Let's compare the performance with inserting 1 million user records.
Here is the result. We will use this alias:

| Alias                       | Command                                                           |
|-----------------------------|-------------------------------------------------------------------|
| execute_psql_command        |time docker exec -i postgres_instance psql -U postgres -d db       |
| execute_pg_bulkload_command |time docker exec -i postgres_instance pg_bulkload -d db -U postgres|


| Method               | Command                                                               | Time (second) |
|----------------------|-----------------------------------------------------------------------|---------------|
| INSERT               | execute_psql_command < _out/batch_create_1000000_users.sql            |7.065          |
| COPY                 | execute_psql_command -c "\COPY users(id, name) FROM '/users.csv' CSV" |2.148          |
| pg_bulkload WAL on   | execute_pg_bulkload_command /tmp/bulkload_on.ctl                      |2.089          |
| pg_bulkload WAL off  | execute_pg_bulkload_command /tmp/bulkload_off.ctl                     |1.776          |
| pg_bulkload parralel | execute_pg_bulkload_command /tmp/bulkload_parralel.ctl                |1.552          |

At this amout of data, the pg_buldload is faster than COPY, but not much.
The pg_buldload parralel won the race.

## Batch insert in SQLC

[SQLC](https://github.com/sqlc-dev/sqlc) is the tool generates Go code from SQL queries.
[pgx](https://github.com/jackc/pgx) is the Postgres driver for Go.

Why this combo?
- Postgres has one more famous lib, it is [pq](https://github.com/lib/pq). But it is currently in maintenance mode and also recommends to use pgx.
- To deal with SQL, there are many opinions. Golang libs also provide ORM (like gorm). Or we can handle the SQL directly. Combining of generating code and writing SQL, SQLC is a good choice for simplicy and efficiency.

### pgx batch opepration

As mentioned above, with small amount of data, we can use INSERT command, even if they are separated like this:

```sql
INSERT INTO users (id, name) VALUES ('1', 'User 1');
INSERT INTO users (id, name) VALUES ('2', 'User 2');
INSERT INTO users (id, name) VALUES ('3', 'User 3');
```

It is *acceptable*. When a SQL command is executed, Postgres will do a chain.

SQL string → [Parse] → [Rewrite] → [Plan] → [Acquire locks] → [Execute] → [Return result]

pgx provides the **batch** definition, that allows multiple commands to be executed in a single round trip.
That makes the time processing reduced a little bit, but the Postgres server is still doing the same work for each command.

We can write the SQL like this to use the batch operation from sqlc + pgx:

```sql
-- name: BatchInsertUsers :batchexec
INSERT INTO users (id, name)  VALUES ($1, $2)
ON CONFLICT (id) DO NOTHING;
```

We can tool it a little bit more with **PREPARE STATEMENT**.

```sql
PREPARE insert_user(varchar, varchar) AS
INSERT INTO users (id, name) VALUES ($1, $2);

EXECUTE insert_user('1', 'User 1');
EXECUTE insert_user('2', 'User 2');
EXECUTE insert_user('3', 'User 3');
```

The parser and planner will be called only once. The execution is faster.
But the reality is the batch operation still execute multiple commands.

Do I miss the INSERT command with multiple values? The answer is no. The SQLC is type-safe generator.
It need to know the params of the command before it generates the code.
INSERT command with multiple values has the dynamic params, so the SQLC can't handle it.

### unnest function

Luckily, Postgres has the **unnest** function, that can be used to convert an array to a set of rows.
We can do the batch insert with a little bit tricky like this:

```sql
INSERT INTO users (id, name)
SELECT unnest(@ids::text[]) AS id,
  unnest(@names::text[]) AS name;
```

Now, we can pass the params by 2 arrays that has the same length. Because the params is now determined (array type), SQLC can handle it normally.

### Benchmark

Let's do a small test with copy, batch insert and unnest insert.


```bash
=== RUN   BenchmarkAllMethods/BatchInsert-Size-1000000
BenchmarkAllMethods/BatchInsert-Size-1000000
BenchmarkAllMethods/BatchInsert-Size-1000000-12                        1        14823435666 ns/op       1303606000 B/op  9006197 allocs/op
=== RUN   BenchmarkAllMethods/CopyFrom-Size-1000000
BenchmarkAllMethods/CopyFrom-Size-1000000
BenchmarkAllMethods/CopyFrom-Size-1000000-12                           1        5473882250 ns/op        96364032 B/op    3000087 allocs/op
=== RUN   BenchmarkAllMethods/Unnest-Size-1000000
BenchmarkAllMethods/Unnest-Size-1000000
BenchmarkAllMethods/Unnest-Size-1000000-12                             1        7107586666 ns/op        567170472 B/op   2000092 allocs/op
```

With 1 million records, COPY is the fastest method with 5.4 seconds, the Unnest is slower but quite near with 7.1 seconds.
The batch insert is the slowest with 14.8 seconds and the huge memory usage of 1.3GB.

However the unnest method also consume a lot of memory, about 567MB. Compare to COPY with just 96MB, it is not so good.

How about the size of 100 records?

```bash
=== RUN   BenchmarkAllMethods/BatchInsert-Size-100
BenchmarkAllMethods/BatchInsert-Size-100
BenchmarkAllMethods/BatchInsert-Size-100-12                  624           1832595 ns/op           92372 B/op        927 allocs/op
=== RUN   BenchmarkAllMethods/CopyFrom-Size-100
BenchmarkAllMethods/CopyFrom-Size-100
BenchmarkAllMethods/CopyFrom-Size-100-12                    1274           1017753 ns/op           35979 B/op        338 allocs/op
=== RUN   BenchmarkAllMethods/Unnest-Size-100
BenchmarkAllMethods/Unnest-Size-100
BenchmarkAllMethods/Unnest-Size-100-12                      1156           1135188 ns/op           42296 B/op        223 allocs/op
```

The order is the same, but the difference between COPY and Unnest is not so much.
The allocation of unnest is also the lowest (save Garbage collector).

So, we can use the trick to insert the batch records in SQLC.
From benchmark, we can see that with small number of records, the unnest method is quite efficient.

## Conclusion

In this post, we have discussed about the batch insert directly in Postgres and indirectly through SQLC (Golang).
Let's choose the suitable method for your case.

