+++
date = '2025-04-01T10:52:10+09:00'
draft = false
title = 'Batch records creation with Go and Postgres'
+++

Today I write a small note about creating multiple records in SQL database.
The stack include:
- Postgres
- Golang
- Docker (Podman)

## Setup

We will use the simple `users` table, with 2 main fields:
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


To be continued ...

