+++
date = '2025-04-01T10:52:10+09:00'
draft = false
title = 'Batch records creation with Go and Postgres'
tags = ['postgres','golang','pgx','sqlc']
+++

Let's talk about 2 problems in this post.
- batch insert in Postgres
- batch insert in SQLC (with Golang)

# Resource preparation

To simplify, we will use an **users** table with 4 fields: id(text, primary key), name(text), created_at and updated_at.
The insert data can be generated with [**pg-batch-play**](https://github.com/HoangNguyen689/pg-batch-play) tool.

```bash
# generate 1M users data in SQL format with single INSERT command

go run main.go genbatchcreateuser -s 1000000 -f sql -t singular
```

Besides, we will also use **psql** tool to observe, so let's enable the timing option.

```psql
db=# \timing
Timing is on.
```

# Batch insert in Postgres

## INSERT command

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
Of course, Postgres has [the hard limit](https://www.postgresql.org/docs/current/limits.html). But before reaching the 1GB query limitation, the hardware limitations likely will be reached first.

Who concerns about the query string limitation can read the code here:
- [StringInfo](https://github.com/postgres/postgres/blob/master/src/include/lib/stringinfo.h)
- [MaxAllocSize](https://github.com/postgres/postgres/blob/master/src/include/utils/memutils.h#L43)

The INSERT command has long time execution and the limitation about the query string (as well as the hardware). To gain more performance, we can use COPY command or 3rd party tool like pg_bulkload.

## COPY command

The COPY command is native supported in Postgres. It is more efficient than INSERT command, because:
- It doesn't need to parse the SQL syntax. Just read the data file directly.
- It read the file by block, consume less memory.(eliminate the hardware limitation)
- It can disalbe WAL to move faster.

## pg_bulkload

The pg_bulkload is a 3rd party tool. To use it, we need to install it along side with Postgres.
It even faster than COPY command in case of large data set.
- It doesn't call the SQL command, ignore the parse and planner.
- It directly write to table heap file, not through the Postgres engine.
- It can bypass WAL,
- No trigger, no constraints, no index when loading.

Of course, the data must be clean. With WAL off, there's no way to rollback. So we need to be careful.

## Benchmark

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

# Batch insert in SQLC



## Golang implementation

There're 2 libs that deal with Postgres in Golang.
- pg
- pgx

The pgx is popular now and strongly supported.

To deal with SQL, there're many ways.

To be continued!
