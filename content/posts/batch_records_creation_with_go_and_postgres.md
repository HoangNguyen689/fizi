+++
date = '2025-04-01T10:52:10+09:00'
draft = false
title = 'Batch records creation with Go and Postgres'
tags = ['postgres','golang','pgx','sqlc']
+++

Today I write a small note about creating multiple records in SQL database.
The stack include:
- Postgres
- Golang
- Docker (Podman)

## Setup

We will use the simple {{< highlight go "hl_inline=true, style=github" >}}users{{< /highlight >}} table, with 2 main fields:
- id (text): primary key
- name (text)

## Batch records creation in Postgres

We will start with the singular simplest SQL query.

```sql
INSERT INTO users VALUES ('1', 'User 1');
```

This query wil create a record in `users` table. What if we want to add many records.
The easiest way is to do what is succeeded again

```sql
INSERT INTO users VALUES ('3', 'User 3');
INSERT INTO users VALUES ('4', 'User 4');
INSERT INTO users VALUES ('5', 'User 5');
```

If we run these queries, the response time still fast. Because our `users` table is simple, has two fields, has no index, and we only insert three records.

Although the response time is small, we still can measure it. One of the way is using psql.
To make it easy to compare, we store the queries in file.
- `singular_user.sql` for 1 query.
- `multiple_3_users.sql` for 3 queries.
- `multiple_1000_users.sql` for 1000 queries.
- and so on ...

Let's go inside psql console and enable timing

```psql
db=# \timing
Timing is on.
```

And we can test the time each case
```psql
db=# \i singular_user.sql
INSERT 0 1
Time: 4.669 ms
```

```psql
db=# \i multiple_3_users.sql
INSERT 0 1
Time: 1.303 ms
INSERT 0 1
Time: 0.605 ms
INSERT 0 1
Time: 0.536 ms
```

As we can see, although we put multiple queries in one file, they are still processed in turn.
In each query, the database system does the sequence of actions.

<TODO: Insert image>
```
SQL string
    ↓
[Parse]
    ↓
[Rewrite]
    ↓
[Plan]
    ↓
[Acquire locks]
    ↓
[Execute]
    ↓
[Return result]
```

Although the queries format seems similar, when receiving one query, all the above actions must be done.
Can we improve it by the similarity in the above queires.
Yes, we can. Using `Prapared statement` is the key. By preparing the format of the queries, we can by pass the `Plan` step, and excute it immediately.

```sql
PREPARE insert_user(varchar, varchar) AS
INSERT INTO users (id, name) VALUES ($1, $2);

EXECUTE insert_user('3', 'User 3');
EXECUTE insert_user('4', 'User 4');
EXECUTE insert_user('5', 'User 5');
```

We can even do it better. `INSERT` provides a way to create multiple values in one query.
```sql
INSERT INTO users (id, name) VALUES
    ('3', 'User 3'),
    ('4', 'User 4'),
    ('5', 'User 5');
```

By using this way, we can save many "duplicate step" inside the database system, as well as round time trip between client and database system server.

Cool!

Are you curious that how many values can we insert in one query?
Of course, there is a limit. You can check it [here](https://www.postgresql.org/docs/current/limits.html?utm_source=chatgpt.com)

The field limit is 1GB. So technically, we can't insert more than 1GB of data in one query. But if we look further, the limitation is more strict.

The query is store in **StringInfo** , which [currently limited to a length of 1GB](https://github.com/postgres/postgres/blob/master/src/include/lib/stringinfo.h). Also, the **MaxAllocSize** is [1GB](https://github.com/postgres/postgres/blob/3c6e8c123896584f1be1fe69aaf68dcb5eb094d5/src/include/utils/memutils.h#L40). So, the actual data which can be inserted is smaller than the 1GB limitation.

However, 1GB of text query is big. We will reach the mitation of the resource (RAM, CPU, ...etc) before we reach the 1GB limit. Want to try?

Let's create a docker machine that has 2GB memory and 6 vCPUs.
You can test with any size of data you want. But we will test with 500.000 records and 1.000.000 records.
With an empty users table, the 500.000 records is quite fast,but the 1.000.000 records is cracked.
When seeing the postgres log, just the signal kill is displayed. It usually mean we get Out of Memory (OOM).
We cab see memory usage by `docker stats` command. And we can see the raise of memory in instant.

Let raise the memory of docker instant to 8GB and try again. Now, we can insert 1.000.000 records in one query quite fast.

But it seems that the memory dependency is not good. Any other ways to achieve large batch insertions?

There are 2 ways.
- Use COPY command
- Use 3rd tool: pg_bulkload

The COPY command is a fast way to put a large data set into a table.

Why is COPY fast?

It pass the data directly into the table.

How about pg_bulkload?

It even faster than COPY command in case of large data set.

It process directly with the table heap file, has option to by pass WAL, no trigger, no constraints, no index when loading.

Here is the benchmark of inserting 1.000.000 user records.

| Method | Command | Time |
|--------|---------|------|
| INSERT | time podman exec -i postgres_instance psql -U postgres -d db < _out/batch_create_1000000_users.sql |0.04s user 0.05s system 1% cpu 7.065 total|
| COPY   | time podman exec -i postgres_instance psql -U postgres -d db -c "\COPY users(id, name) FROM '/tmp/batch_create_1000000_users.csv' CSV" | 0.03s user 0.02s system 2% cpu 2.148 total |
| pg_bulkload WAL on | time podman exec -i postgres_instance pg_bulkload -d db -U postgres /tmp/bulkload_on.ctl | 0.03s user 0.02s system 2% cpu 2.089 total |
| pg_bulkload WAL off | time podman exec -it postgres_instance pg_bulkload -d db -U postgres /tmp/bulkload_off.ctl | 0.03s user 0.02s system 2% cpu 1.776 total |
| pg_bulkload parralel | time podman exec -it postgres_instance pg_bulkload -d db -U postgres /tmp/bulkload_parralel.ctl | 0.02s user 0.01s system 1% cpu 1.552 total |

As the above result, we can see the INSERT multiple values is slowest, took 7.065 seconds.
The COPY command is faster, took 2.148 seconds.
The pg_bulkload is the fastest, took 1.552 seconds.

## Golang implementation

There're 2 libs that deal with Postgres in Golang.
- pg
- pgx

The pgx is popular now and strongly supported.

To deal with SQL, there're many ways.

To be continued!
