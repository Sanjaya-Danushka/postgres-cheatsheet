# PostgreSQL Cheat Sheet

**Everything you need to run PostgreSQL, written for people who have never used it.**

No prior database experience assumed. Every command is explained in plain English,
and every risky command tells you what it destroys before you run it.

---

## Table of contents

- [What is PostgreSQL?](#what-is-postgresql)
- [The words you will see everywhere](#the-words-you-will-see-everywhere)
- [Install](#install)
- [Set it up for the first time](#set-it-up-for-the-first-time)
- [Connect to it](#connect-to-it)
- [Your first 10 commands](#your-first-10-commands)
- [Reading data: SELECT](#reading-data-select)
- [Writing data: INSERT, UPDATE, DELETE](#writing-data-insert-update-delete)
- [Data types](#data-types)
- [Constraints (rules for your data)](#constraints-rules-for-your-data)
- [Joining tables](#joining-tables)
- [Grouping and counting](#grouping-and-counting)
- [Creating databases](#creating-databases)
- [Users and roles](#users-and-roles)
- [Permissions](#permissions)
- [Tables: the full set of changes](#tables-the-full-set-of-changes)
- [Indexes](#indexes)
- [Views](#views)
- [Transactions](#transactions)
- [Auto-incrementing IDs](#auto-incrementing-ids)
- [Backups and restore](#backups-and-restore)
- [Making queries fast: EXPLAIN](#making-queries-fast-explain)
- [psql backslash commands](#psql-backslash-commands)
- [pgcli — a friendlier psql](#pgcli--a-friendlier-psql)
- [Python with psycopg](#python-with-psycopg)
- [DBeaver and other GUI tools](#dbeaver-and-other-gui-tools)
- [Running PostgreSQL with Docker](#running-postgresql-with-docker)
- [Start, stop, restart](#start-stop-restart)
- [Where the files live](#where-the-files-live)
- [Security checklist](#security-checklist)
- [Coming from MySQL](#coming-from-mysql)
- [Common errors and how to fix them](#common-errors-and-how-to-fix-them)
- [The one-page cheat sheet](#the-one-page-cheat-sheet)
- [Glossary](#glossary)

---

## What is PostgreSQL?

PostgreSQL (most people call it **"Postgres"**) is a **database**. A database is
just a organised collection of information that software can read and write quickly.

Think of it like this:

| Real life | In a database |
|---|---|
| A filing cabinet | The database |
| A drawer | A table |
| A row in a spreadsheet | A row / record |
| A column heading | A column / field |
| The person who can open a drawer | A role / user |

People use it to store things like user accounts, orders, blog posts, bank
transactions, or IoT sensor readings. Almost every serious website uses Postgres
or something like it.

**Free, open source, and used by companies like Instagram, Spotify, and GitHub.**

There is nothing to "install and start clicking" — Postgres is a *server*. Your
application connects to it over the network, even when it runs on the same
computer. That is why the same SQL works on your laptop and in production.

---

## The words you will see everywhere

Learn these six and the rest of this document makes sense.

```
PostgreSQL server  ──►  database  ──►  schema  ──►  table  ──►  row
(the program)          (shopdb)      (public)     (notes)    (one note)
                              ▲
                              └── created and owned by a role
```

**Server** — the running program. Also called a service, daemon, or instance.
One machine can run several servers on different ports.

**Port** — which "door" to knock on. PostgreSQL uses **5432** by default.

**Database** — a named collection of your data. `mydb`, `shopdb`. A server can
hold many databases. They cannot see into each other.

**Schema** — a folder *inside* a database that holds tables. Most people never
leave the default schema called `public`.

**Table** — the actual grid of rows and columns. `notes`, `users`, `orders`.

**Role** — a user. In PostgreSQL a "role" can log in (`LOGIN`) or just be a
bundle of permissions that gets given to other roles. There is no separate
"user" concept — that is why the screen says *role* and not *user*.

**Row** — one record. One note, one order.
**Column** — one field of every row. `title`, `created_at`.

---

## Install

### Arch Linux

```bash
sudo pacman -S postgresql
```

### Ubuntu / Debian

```bash
sudo apt update
sudo apt install postgresql
```

### macOS (Homebrew)

```bash
brew install postgresql
brew services start postgresql
```

### Windows

Download the installer from <https://www.postgresql.org/download/windows/>.
It installs the server, `psql`, and pgAdmin in one go.

### Any OS, no install — Docker

```bash
docker run --name pg \
  -e POSTGRES_USER=alexa \
  -e POSTGRES_PASSWORD=yourpassword \
  -e POSTGRES_DB=mydb \
  -p 5432:5432 \
  -d postgres:18
```

That is a complete PostgreSQL server in one command. `--name pg` gives it a
name, `-e` sets initial settings, `-p 5432:5432` exposes it to your machine,
and `-d` runs it in the background. `docker stop pg` shuts it down.

---

## Set it up for the first time

The steps below cover the Linux packages. macOS and Docker already do this for
you, so skip to [Connect to it](#connect-to-it) if you used those.

### 1. Create the data directory and initialise the cluster

A "cluster" is just the folder holding your data. `initdb` creates it.

```bash
# --- Arch Linux ---
sudo mkdir -p /var/lib/postgres
sudo chown postgres:postgres /var/lib/postgres
sudo -u postgres initdb -D /var/lib/postgres/data

# --- Ubuntu / Debian (different path, that is normal) ---
sudo mkdir -p /var/lib/postgresql/data
sudo chown -R postgres:postgres /var/lib/postgresql
sudo -u postgres initdb -D /var/lib/postgresql/data
```

You should see `Success. You can now start the database server`.

> **Gotcha:** if `initdb` says `could not create directory ... Permission
> denied`, you missed the `mkdir`/`chown` above. If the service later says
> `"/var/lib/postgres/data" is missing or empty`, you initialised the wrong
> path — check which one `systemctl show postgresql -p Environment` reports.

### 2. Start it and make it start on boot

```bash
sudo systemctl enable --now postgresql
sudo systemctl status postgresql
```

You want to see `Active: active (running)`.

### 3. Create your login role and database

Log in as the built-in superuser first:

```bash
sudo -u postgres psql
```

You get a prompt ending in `postgres=#`. Now type these **one at a time**,
pressing Enter after each:

```sql
CREATE ROLE alexa WITH LOGIN PASSWORD 'yourpassword';
```

```sql
CREATE DATABASE mydb OWNER alexa;
```

```sql
GRANT CREATEDB TO alexa;
```

> **Why one at a time?** If you paste several lines into the interactive psql
> prompt at once, psql can swallow the rest of the paste as an argument to a
> backslash command and fail with a confusing error. Run them one by one, or
> put them in a file and use `psql -f file.sql`.

### 4. Log in as yourself

```bash
psql -h localhost -U alexa -d mydb
```

Type `yourpassword` when prompted.

**The four values you keep needing — remember these:**

| | |
|---|---|
| **host** | `localhost` |
| **port** | `5432` |
| **user** | `alexa` |
| **password** | `yourpassword` |
| **database** | `mydb` |

---

## Connect to it

### The five ways

```bash
psql -h localhost -U alexa -d mydb                 # built in, always available
pgcli -h localhost -U alexa -d mydb                # same thing, much nicer
sudo -u postgres psql                             # as the superuser
psql -h localhost -U alexa -d postgres             # to create other databases
psql -h mydb                                      # as your OS user, if roles match
```

If you would rather not memorise flags, set them once:

```bash
echo "export PGHOST=localhost PGPORT=5432 PGUSER=alexa PGDATABASE=mydb" >> ~/.bashrc
source ~/.bashrc
psql            # now just works
```

### Connection strings

One string works anywhere — Python, DBeaver, Django, a config file:

```
postgresql://alexa:yourpassword@localhost:5432/mydb
```

Or as keyword=value pairs:

```
host=localhost port=5432 dbname=mydb user=alexa password=yourpassword
```

---

## Your first 10 commands

Learn these and you can do real work.

```sql
SELECT 1;                 -- check the server answers
SELECT version();         -- which PostgreSQL is this
\l                        -- list all databases          (\l in psql only)
\c mydb                   -- switch database              (\c in psql only)
\dt                       -- list tables in this database (\dt)
\d notes                  -- describe a table             (\d)
CREATE TABLE notes (id SERIAL PRIMARY KEY, title TEXT NOT NULL, body TEXT);
INSERT INTO notes (title, body) VALUES ('first note', 'hello world');
SELECT * FROM notes;
UPDATE notes SET body = 'edited' WHERE id = 1;
DELETE FROM notes WHERE id = 1;
\q                        -- quit
```

Backslash commands like `\l` and `\dt` are **psql and pgcli features, not SQL.**
If you send them to Python or a web app you will get a syntax error — use real
SQL there instead (the [Listing things with SQL](#listing-things-with-sql-not-backslash-commands) section).

---

## Reading data: SELECT

`SELECT` asks a question. Everything after `FROM` is where to look.

```sql
-- everything
SELECT * FROM notes;

-- just some columns (always better than * — faster, and clearer)
SELECT id, title FROM notes;

-- one row
SELECT * FROM notes WHERE id = 1;

-- text searches
SELECT * FROM notes WHERE title LIKE 'first%';   -- % means "anything"
SELECT * FROM notes WHERE title ILIKE 'FIRST%';  -- case-insensitive
SELECT * FROM notes WHERE title = 'first note';

-- numbers
SELECT * FROM notes WHERE id > 10;
SELECT * FROM notes WHERE id BETWEEN 5 AND 20;     -- inclusive both ends
SELECT * FROM notes WHERE id IN (1, 2, 3);

-- several conditions
SELECT * FROM notes WHERE id > 1 AND title IS NOT NULL;
SELECT * FROM notes WHERE title = 'a' OR title = 'b';
SELECT * FROM notes WHERE NOT archived;

-- NULL means "unknown / not filled in". Never use = NULL.
SELECT * FROM notes WHERE body IS NULL;
SELECT * FROM notes WHERE body IS NOT NULL;

-- sorting and limiting
SELECT * FROM notes ORDER BY created_at DESC;
SELECT * FROM notes ORDER BY title ASC, id DESC;   -- two sort levels
SELECT * FROM notes ORDER BY id DESC LIMIT 10;     -- top 10
SELECT * FROM notes ORDER BY id LIMIT 10 OFFSET 20; -- skip 20, then 10

-- how many
SELECT count(*) FROM notes;
SELECT count(DISTINCT title) FROM notes;
SELECT min(id), max(id), avg(id), sum(id) FROM notes;

-- maths
SELECT id, price * quantity AS total FROM order_items;

-- rename a column in the output
SELECT id AS note_id, title AS heading FROM notes;
```

### The `LIKE` wildcards

```sql
'a%'     starts with a
'%a'     ends with a
'%a%'    contains a
'_'      one single character
'a_1'    a, any one char, then 1
```

### `IS NULL`, never `= NULL`

`NULL` means *we do not know the value*. It is not equal to anything, not even
itself, so `= NULL` always returns nothing. Use `IS NULL` / `IS NOT NULL`.

### Case sensitivity

Values are case sensitive (`'A'` ≠ `'a'`), but **column names are not**
(unquoted `Title` and `title` are the same column).

---

## Writing data: INSERT, UPDATE, DELETE

### Insert one row

```sql
INSERT INTO notes (title, body) VALUES ('my title', 'my body');
```

Only name the columns you are filling — the rest get their default.

### Insert many rows at once (much faster)

```sql
INSERT INTO notes (title, body) VALUES
  ('first',  'body one'),
  ('second', 'body two'),
  ('third',  'body three');
```

### Insert and get the new id back

```sql
INSERT INTO notes (title) VALUES ('new') RETURNING id;
```

### Copy rows from another table

```sql
INSERT INTO notes (title, body) SELECT title, body FROM notes_backup;
```

### Update

```sql
UPDATE notes SET body = 'new text' WHERE id = 1;
UPDATE notes SET title = 'a', body = 'b' WHERE id = 1;      -- several columns
UPDATE notes SET body = 'x' WHERE title LIKE 'test%';        -- many rows
```

> ⚠️ **`UPDATE notes SET body = 'x';` with no `WHERE` rewrites every single
> row.** There is no undo. Always write the `WHERE` first, run it with a `SELECT`
> to see what it touches, *then* change `SELECT` to `UPDATE`.

Safe habit:

```sql
SELECT * FROM notes WHERE id = 1;         -- check
UPDATE notes SET body = 'x' WHERE id = 1; -- then do it
```

### Delete

```sql
DELETE FROM notes WHERE id = 1;
```

> ⚠️ **`DELETE FROM notes;` with no `WHERE` deletes every row.** Same habit:
> test the `WHERE` with `SELECT` first.

### `TRUNCATE` — empty a table fast

```sql
TRUNCATE notes;                   -- deletes all rows, keeps the table
TRUNCATE notes RESTART IDENTITY;  -- also resets the id counter to 1
TRUNCATE notes CASCADE;           -- also empties tables that reference it
```

Faster than `DELETE` because it does not log each row, but it cannot have a
`WHERE`, and in PostgreSQL it cannot run inside a transaction.

---

## Data types

| Type | Use it for | Example |
|---|---|---|
| `SERIAL` | auto-increment number (older style) | `id SERIAL` |
| `INTEGER` / `INT` | whole numbers | `age INT` |
| `BIGINT` | very large whole numbers | `views BIGINT` |
| `SMALLINT` | small whole numbers | |
| `NUMERIC(10,2)` | money, exact decimals | `price NUMERIC(10,2)` |
| `REAL` / `DOUBLE PRECISION` | science, approximate | |
| `TEXT` | any length string | `title TEXT` |
| `VARCHAR(100)` | max 100 characters | `code VARCHAR(10)` |
| `CHAR(1)` | exactly one character | `grade CHAR(1)` |
| `BOOLEAN` | true / false | `is_active BOOLEAN` |
| `DATE` | date only | `2026-09-27` |
| `TIME` | time only | `14:30:00` |
| `TIMESTAMP` | date + time, **no** timezone | |
| `TIMESTAMPTZ` | date + time **with** timezone — **prefer this** | `now()` |
| `INTERVAL` | a length of time | `'3 days'` |
| `UUID` | unique identifier | `gen_random_uuid()` |
| `JSONB` | flexible JSON, searchable | `{"a":1}` |
| `BYTEA` | binary files | |
| `INET` | IP address | |
| `ENUM` | one of a fixed list | see below |

**Money:** never use `FLOAT` or `REAL` for money. `0.1 + 0.2` is not exactly
`0.3` in floating point. Use `NUMERIC(10,2)`.

**Timestamps:** always store `TIMESTAMPTZ`. It keeps the real instant and shows
it in whatever timezone you ask for, so it survives daylight saving and moving
between countries.

### Custom lists with ENUM

```sql
CREATE TYPE order_status AS ENUM ('pending', 'paid', 'shipped', 'cancelled');
CREATE TABLE orders (
  id     SERIAL PRIMARY KEY,
  status order_status NOT NULL DEFAULT 'pending'
);
```

### Arrays

```sql
CREATE TABLE posts (id SERIAL, tags TEXT[]);
INSERT INTO posts (tags) VALUES (ARRAY['python', 'sql']);
SELECT tags[1] FROM posts;
```

### JSONB

```sql
CREATE TABLE events (
  id  SERIAL PRIMARY KEY,
  data JSONB NOT NULL
);
INSERT INTO events (data) VALUES ('{"user": "alexa", "action": "login"}');
SELECT data->>'user' FROM events;                     -- alexa
SELECT * FROM events WHERE data @> '{"action":"login"}';   -- contains
```

### UUIDs

```sql
CREATE EXTENSION IF NOT EXISTS "pgcrypto";
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL
);
```

---

## Constraints (rules for your data)

Constraints stop bad data from ever getting in. Set them up front.

```sql
CREATE TABLE users (
  id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  email      TEXT NOT NULL UNIQUE,
  username   TEXT NOT NULL UNIQUE,
  age        INT CHECK (age >= 0 AND age < 150),
  status     TEXT NOT NULL DEFAULT 'active',
  team_id    INT REFERENCES teams(id) ON DELETE SET NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

| Constraint | What it does |
|---|---|
| `NOT NULL` | value is required |
| `UNIQUE` | no two rows may share this value |
| `PRIMARY KEY` | unique + not null + indexed. Every good table has one |
| `CHECK (expr)` | value must satisfy the expression |
| `DEFAULT expr` | value used if you do not supply one |
| `REFERENCES other(id)` | a foreign key — the value must exist in the other table |
| `GENERATED ALWAYS AS IDENTITY` | auto-increment id (modern replacement for SERIAL) |

**Foreign key actions** — what happens to this row when the row it points at is deleted:

| Action | Meaning |
|---|---|
| `ON DELETE RESTRICT` | refuse the delete (the default, safest) |
| `ON DELETE CASCADE` | delete the child rows too |
| `ON DELETE SET NULL` | set the column to NULL |

### Adding and removing constraints later

```sql
ALTER TABLE users ADD CONSTRAINT age_positive CHECK (age >= 0);
ALTER TABLE users DROP CONSTRAINT age_positive;
ALTER TABLE users ADD UNIQUE (email);
ALTER TABLE users ALTER COLUMN email DROP NOT NULL;
```

---

## Joining tables

A foreign key points at a row in another table. To *use* that relationship you
join. This is the single most important SQL skill.

```sql
SELECT c.name, o.total
FROM orders o
JOIN customers c ON c.id = o.customer_id;
```

Read it right to left: *for each order, find the customer whose id matches, and
show the customer's name and the order's total.*

```sql
-- different names for the same idea
INNER JOIN  = JOIN              only rows that match on both sides
LEFT JOIN                      all rows from the left, NULL where the right has no match
RIGHT JOIN                     all rows from the right
FULL JOIN                      all rows from both, NULL where missing
CROSS JOIN                     every row paired with every row (careful, explodes)
```

### LEFT JOIN, with an example

```sql
-- every customer, plus their orders (or NULL if they never ordered)
SELECT c.name, o.total
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id;
```

```sql
-- every customer and order, all side by side
SELECT c.name, o.total
FROM customers c
FULL JOIN orders o ON o.customer_id = c.id;
```

**Always alias your tables** (`FROM customers c`) when there are two or more —
otherwise you must repeat the long name everywhere.

**`ON` vs `USING`:** `JOIN t ON a.id = b.a_id` and `JOIN t USING (id)` are the
same thing; `USING` is shorter when the column names match.

### Three-table join

```sql
SELECT c.name, o.id AS order_id, p.name AS product
FROM orders o
JOIN customers c ON c.id = o.customer_id
JOIN order_items oi ON oi.order_id = o.id
JOIN products p     ON p.id = oi.product_id;
```

---

## Grouping and counting

```sql
SELECT status, count(*) AS n
FROM orders
GROUP BY status;
```

One output row per `status`, each with how many orders has that status.

```sql
SELECT status, count(*) AS n, sum(total) AS revenue
FROM orders
GROUP BY status
HAVING sum(total) > 1000        -- filter AFTER grouping
ORDER BY revenue DESC;
```

| Clause | When it runs |
|---|---|
| `WHERE` | **before** grouping — filters individual rows |
| `GROUP BY` | collapses rows into groups |
| `HAVING` | **after** grouping — filters the groups |

`WHERE` cannot use `sum(total)`, because the total does not exist yet at that
point. That is the whole difference between `WHERE` and `HAVING`.

```sql
-- useful extras
SELECT count(*) FILTER (WHERE total > 100) AS big_orders FROM orders;
SELECT date_trunc('month', created_at) AS month, count(*) FROM orders GROUP BY month;
```

### Subqueries and CTEs

```sql
-- a subquery
SELECT * FROM orders WHERE customer_id IN (SELECT id FROM customers WHERE country = 'LK');

-- a CTE: name a subquery so the query reads like English
WITH big AS (
  SELECT * FROM orders WHERE total > 1000
)
SELECT c.name, big.total
FROM big
JOIN customers c ON c.id = big.customer_id;
```

---

## Creating databases

You must be connected to a **different** database than the one you are
creating. `postgres` is the usual one for this.

```sql
CREATE DATABASE shopdb;
CREATE DATABASE shopdb OWNER alexa;      -- alexa can then create tables in it
CREATE DATABASE testdb TEMPLATE template0;   -- a pristine, empty copy
```

| Statement | What it does |
|---|---|
| `CREATE DATABASE name;` | make a new empty database |
| `DROP DATABASE name;` | **destroy it and everything inside** |
| `DROP DATABASE IF EXISTS name;` | same, but no error if it is not there |
| `DROP DATABASE name WITH (FORCE);` | also disconnect anyone currently using it |
| `ALTER DATABASE name OWNER TO alexa;` | change who owns it |
| `ALTER DATABASE name RENAME TO other;` | rename it |
| `\c name` | switch into it (psql only) |
| `\l` | list all databases (psql only) |

**Template databases** — you will see these in every `\l` listing and should
never delete them:

| Database | Why it exists |
|---|---|
| `postgres` | the default one; your tools connect here to create others |
| `template1` | the blueprint copied every time you run `CREATE DATABASE` |
| `template0` | a pristine blueprint, no extensions. Your recovery copy |

### Listing things with SQL (not backslash commands)

Backslash commands only exist in psql. Everywhere else, query the catalog:

```sql
-- databases
SELECT datname FROM pg_database WHERE datistemplate = false;

-- tables in the current database
SELECT tablename FROM pg_tables WHERE schemaname = 'public';

-- columns of a table
SELECT column_name, data_type, is_nullable
FROM information_schema.columns
WHERE table_name = 'notes';

-- everything about a table
SELECT * FROM information_schema.tables WHERE table_name = 'notes';
```

---

## Users and roles

```sql
-- create
CREATE ROLE alexa  WITH LOGIN PASSWORD 'yourpassword';
CREATE USER  alexa2 WITH PASSWORD 'otherpassword';   -- shorthand, implies LOGIN
CREATE ROLE readonly WITH LOGIN PASSWORD 'x';        -- will only get read grants

-- a role with no password, used purely as a bag of permissions
CREATE ROLE reporting;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO reporting;

-- change
ALTER ROLE alexa WITH PASSWORD 'newpassword';
ALTER ROLE alexa CREATEDB;           -- may create databases
ALTER ROLE alexa CREATEDB NOCREATEDB;-- may not
ALTER ROLE alexa NOLOGIN;            -- lock the account out
ALTER ROLE alexa VALID UNTIL '2027-01-01';   -- temporary account

-- nest roles
GRANT reporting TO alexa;            -- alexa now has everything reporting has
REVOKE reporting FROM alexa;

-- remove
DROP ROLE reporting;
```

**Cleanup before dropping a role.** Postgres refuses to drop a role that still
owns things, which is a safety net, not a bug:

```sql
REASSIGN OWNED BY oldrole TO alexa;   -- hand over what they own
DROP OWNED BY oldrole;                 -- remove their grants
DROP ROLE oldrole;
```

Built-in roles you can grant instead of inventing permissions:

| Role | Gives |
|---|---|
| `pg_read_all_data` | read every table and view |
| `pg_write_all_data` | write to every table |
| `pg_monitor` | see other people's queries and stats |
| `pg_signal_backend` | cancel other people's queries |

**Never run as the `postgres` superuser in application code.** Create a normal
role for the app and grant it only what it needs.

---

## Permissions

Permissions in PostgreSQL are granted in a chain: *role → database → schema →
table*. A user needs every link in the chain or the operation is refused.

```sql
-- database level
GRANT CONNECT ON DATABASE mydb TO alexa;

-- schema level (often the one people forget)
GRANT USAGE ON SCHEMA public TO alexa;
GRANT ALL ON SCHEMA public TO alexa;
ALTER SCHEMA public OWNER TO alexa;          -- or just hand over the schema

-- table level
GRANT SELECT, INSERT ON notes TO alexa;      -- read and add, nothing else
GRANT ALL ON notes TO alexa;                 -- every privilege
GRANT ALL ON ALL TABLES IN SCHEMA public TO alexa;   -- all existing tables
REVOKE ALL ON notes FROM alexa;

-- change the owner
ALTER TABLE notes OWNER TO alexa;

-- for every table created FROM NOW ON by this role
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT SELECT ON TABLES TO alexa;
```

View them with:

```sql
SELECT * FROM information_schema.role_table_grants WHERE grantee = 'alexa';
```

| Privilege | Allows |
|---|---|
| `SELECT` | read |
| `INSERT` | add rows |
| `UPDATE` | change rows |
| `DELETE` | remove rows |
| `TRUNCATE` | empty the table |
| `REFERENCES` | create foreign keys pointing at it |
| `ALL PRIVILEGES` | all of the above |

---

## Tables: the full set of changes

```sql
CREATE TABLE notes (
  id    SERIAL PRIMARY KEY,
  title TEXT NOT NULL,
  body  TEXT
);

-- copy the structure, no rows
CREATE TABLE notes_copy (LIKE notes INCLUDING ALL);
CREATE TABLE notes_old  AS SELECT * FROM notes;   -- structure AND rows

-- rename
ALTER TABLE notes RENAME TO memos;
ALTER TABLE notes RENAME COLUMN body TO content;

-- columns
ALTER TABLE notes ADD COLUMN tag TEXT;
ALTER TABLE notes ADD COLUMN tag TEXT NOT NULL DEFAULT 'none';
ALTER TABLE notes DROP COLUMN tag;
ALTER TABLE notes ALTER COLUMN title TYPE VARCHAR(200);
ALTER TABLE notes ALTER COLUMN title SET NOT NULL;
ALTER TABLE notes ALTER COLUMN title DROP NOT NULL;
ALTER TABLE notes ALTER COLUMN body SET DEFAULT 'empty';

-- empty or destroy
TRUNCATE notes;
TRUNCATE notes RESTART IDENTITY;
DROP TABLE notes;
DROP TABLE IF EXISTS notes;
DROP TABLE notes CASCADE;   -- also drops anything that depends on it
```

### Comments — the best thing nobody uses

```sql
COMMENT ON TABLE notes IS 'Everything the user wrote';
COMMENT ON COLUMN notes.title IS 'Cannot be empty, max 200 chars';
```

### Copying data in and out with CSV

```sql
-- server-side (needs superuser, file is on the *server*)
COPY notes TO '/tmp/notes.csv' CSV HEADER;
COPY notes FROM '/tmp/notes.csv' CSV HEADER;

-- client-side (normal user, file is on *your* machine) - much safer
\copy notes TO 'notes.csv' CSV HEADER
\copy notes FROM 'notes.csv' CSV HEADER
```

Prefer `\copy`. `COPY` reads files on the server, which is a security risk and
unavailable on managed databases like Atlas or RDS.

---

## Indexes

An index is a sorted lookup table PostgreSQL maintains for you, so `WHERE` on
that column stays fast as the data grows. Without it, a search reads every
single row.

```sql
CREATE INDEX idx_notes_title ON notes (title);
CREATE UNIQUE INDEX idx_users_email ON users (email);
CREATE INDEX idx_orders_multi ON orders (customer_id, created_at DESC);  -- two columns
CREATE INDEX idx_notes_lower_title ON notes (lower(title));  -- case-insensitive search

DROP INDEX idx_notes_title;

-- find out which indexes exist
SELECT indexname, indexdef FROM pg_indexes WHERE tablename = 'notes';
```

**The rules of thumb:**

- Index columns you `WHERE`, `JOIN ON`, or `ORDER BY`.
- Indexing every column makes writes slower and wastes space. It is a
  trade-off, not a free win.
- A column declared `PRIMARY KEY` or `UNIQUE` is already indexed.
- Composite index `(a, b)` helps queries on `a` and on `a, b` — but **not**
  on `b` alone. Column order matters.
- Re-index after a bulk load: `REINDEX TABLE notes;`

---

## Views

A view is a saved query with a name. Run it like a table.

```sql
CREATE VIEW recent_notes AS
SELECT id, title, created_at
FROM notes
WHERE created_at > now() - interval '7 days';

SELECT * FROM recent_notes;

DROP VIEW recent_notes;
CREATE OR REPLACE VIEW recent_notes AS SELECT id, title FROM notes;
```

A `MATERIALIZED VIEW` stores the results on disk. Faster to read, but you must
refresh it yourself:

```sql
CREATE MATERIALIZED VIEW note_counts AS
SELECT title, count(*) FROM notes GROUP BY title;

REFRESH MATERIALIZED VIEW note_counts;
```

---

## Transactions

A transaction is a group of statements that **all succeed or all fail together**.
Either every change is saved, or none of them are.

```sql
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
    UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;      -- both saved

-- or, if something goes wrong
ROLLBACK;    -- neither change is kept
```

Real-world example — transferring money without ever losing or duplicating it:

```sql
BEGIN;
UPDATE accounts SET balance = balance - 500 WHERE id = 1;
-- suppose the next line fails
UPDATE accounts SET balance = balance + 500 WHERE id = 999;
COMMIT;   -- would be wrong; use ROLLBACK instead
```

**Things that cannot run inside a transaction:** `CREATE DATABASE`,
`DROP DATABASE`, `VACUUM`, `CREATE INDEX CONCURRENTLY`, `TRUNCATE`.

---

## Auto-incrementing IDs

### Modern way: identity columns

```sql
CREATE TABLE notes (
  id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  title      TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

`GENERATED ALWAYS` means you can never supply the id by accident.
`GENERATED BY DEFAULT` lets you, which is handy when importing old data.

### Older way: SERIAL

```sql
CREATE TABLE notes (id SERIAL PRIMARY KEY, title TEXT);
```

`SERIAL` is a shorthand that creates a separate `sequence` and sets
`DEFAULT nextval(...)`. It still works everywhere and is still very common.

### Working with sequences

```sql
CREATE SEQUENCE ticket_seq START 1000;

SELECT nextval('ticket_seq');   -- 1000
SELECT currval('ticket_seq');   -- 1000, without advancing
SELECT setval('ticket_seq', 1); -- reset to 1
ALTER SEQUENCE ticket_seq RESTART WITH 1;
DROP SEQUENCE ticket_seq;

SELECT last_value FROM notes_id_seq;
```

### Renumbering after deleting the highest rows

```sql
TRUNCATE notes RESTART IDENTITY;              -- empties everything, resets to 1
-- or, to keep rows but reset the counter:
SELECT setval('notes_id_seq', COALESCE((SELECT max(id) FROM notes), 0) + 1, false);
```

---

## Backups and restore

### Dump one database to a file

```bash
pg_dump -h localhost -U alexa -d mydb > mydb_backup.sql          # plain SQL, readable
pg_dump -h localhost -U alexa -Fc -d mydb -f mydb.dump           # compressed, use this
pg_dump -h localhost -U alexa -d mydb -t notes > notes.sql       -- one table only
pg_dump -h localhost -U alexa -d mydb --data-only > mydb_data.sql
```

`-Fc` (custom format) is smaller and restores faster. Use it for real backups.

### Restore

```bash
psql -h localhost -U alexa -d mydb -f mydb_backup.sql
pg_restore -h localhost -U alexa -d mydb --clean --if-exists -d mydb.dump
```

`--clean --if-exists` drops existing objects first, so you can restore over an
existing database.

### Dump everything (all databases, roles, settings)

```bash
pg_dumpall -h localhost -U postgres > all.sql
pg_dumpall --globals-only > roles.sql      # roles only
```

### Automate it — a nightly cron job

```bash
crontab -e
```

```cron
0 2 * * * pg_dump -h localhost -U alexa -Fc -d mydb -f /home/alexa/backups/mydb-$(date +\%F).dump
```

Run at 2am every day. Note the `\%F` — cron needs the backslash.

---

## Making queries fast: EXPLAIN

When a query is slow, ask PostgreSQL what it is doing.

```sql
EXPLAIN SELECT * FROM notes WHERE title = 'first note';
```

Add `ANALYZE` to actually run it and show real timings:

```sql
EXPLAIN ANALYZE SELECT * FROM notes WHERE title = 'first note';
```

What to look for:

| Line | Meaning |
|---|---|
| `Seq Scan` | read every row. Fine for a small table, bad for a big one |
| `Index Scan` / `Index Only Scan` | used an index. Good |
| `Nested Loop` | usually fine for small tables |
| `rows=100000` vs `rows=1` | the estimate is far off, statistics may be stale |

If estimates are wrong, refresh them:

```sql
ANALYZE notes;
VACUUM ANALYZE notes;
VACUUM;                 -- reclaims space, updates visibility, must run outside a txn
VACUUM FULL notes;      -- rewrites the table and locks it. Rarely needed
```

---

## psql backslash commands

Only in psql (and pgcli). Not SQL.

| Command | Does |
|---|---|
| `\q` | quit |
| `\c mydb` | connect to another database |
| `\l` | list databases |
| `\l+` | list databases, more detail |
| `\du` | list roles |
| `\dt` | list tables |
| `\dt+` | list tables, more detail |
| `\d notes` | describe a table: columns, indexes, constraints |
| `\d+ notes` | more detail |
| `\dn` | list schemas |
| `\ds` | list sequences |
| `\dv` | list views |
| `\di` | list indexes |
| `\df` | list functions |
| `\dT` | list types |
| `\dx` | list extensions |
| `\dp notes` | permissions on a table |
| `\z notes` | same, abbreviated |
| `\conninfo` | who am I, which database, which host |
| `\timing` | show how long each query took |
| `\x` | toggle expanded / vertical output |
| `\e` | open the query in your `$EDITOR` |
| `\i file.sql` | run a file |
| `\o file.txt` | save query results to a file |
| `\copy notes TO 'x.csv' CSV` | export to CSV on your own machine |
| `\h CREATE TABLE` | built-in help for any command |
| `\! ls` | run a shell command without leaving psql |

Two that save real time:

```sql
\x
SELECT * FROM notes;         -- each field on its own line: perfect for wide tables
```

```sql
\timing
SELECT count(*) FROM notes;  -- tells you how slow things really are
```

---

## pgcli — a friendlier psql

`pgcli` is the same database with autocomplete, colours, and formatted tables.
Install it:

```bash
sudo pacman -S pgcli     # Arch
sudo apt install pgcli   # Ubuntu / Debian
pip install pgcli        # anywhere, in a venv
```

```bash
pgcli -h localhost -U alexa -d mydb
```

| Key | Does |
|---|---|
| `Tab` | autocomplete table names, columns, keywords |
| `F2` | smart completion mode |
| `F3` | multiline mode — use this before typing a long statement |
| `F5` | explain the query you are on |
| `Ctrl+R` | reverse history search |

```sql
SELECT * FROM not<Tab>          -- completes to notes
SELECT ti<Tab> FROM notes        -- completes to title
\dt
\q
```

A handy convenience: it asks before running anything destructive.

```bash
pgcli -h localhost -U alexa -l              # list databases and exit
pgcli -h localhost -U alexa -d mydb --ping  # check it is reachable, then exit
```

`pgcli` is interactive only — you cannot pipe SQL into it. For scripts, use
`psql -f file.sql`.

---

## Python with psycopg

PostgreSQL's Python driver is **`psycopg`**. (If you came from MySQL, this plays
the role of `mysql.connector`.)

### Install

```bash
# in a project virtual environment (recommended)
mkdir myproject && cd myproject
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install "psycopg[binary]"

# or system-wide from the Arch repo
sudo pacman -S python-psycopg
```

### Connect

```python
import psycopg

with psycopg.connect(
    host="localhost",
    port=5432,
    dbname="mydb",
    user="alexa",
    password="yourpassword",
) as conn:
    with conn.cursor() as cur:
        cur.execute("SELECT current_user, current_database()")
        print(cur.fetchone())
```

Output: `('alexa', 'mydb')`

### Create a table and insert

```python
with psycopg.connect(**CONN) as conn:
    with conn.cursor() as cur:
        cur.execute("""
            CREATE TABLE IF NOT EXISTS notes (
                id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
                title      TEXT NOT NULL,
                body       TEXT,
                created_at TIMESTAMPTZ NOT NULL DEFAULT now()
            )
        """)

        cur.execute(
            "INSERT INTO notes (title, body) VALUES (%s, %s) RETURNING id",
            ("hello", "world"),
        )
        print("new id:", cur.fetchone()[0])
```

### Read results

```python
        cur.execute("SELECT id, title FROM notes WHERE title LIKE %s ORDER BY id DESC", ("h%",))
        for row in cur.fetchall():
            print(row)          # (1, 'hello')  <- a plain tuple
```

**Rows are tuples by default**, so you read them by position: `row[0]`, `row[1]`.
To get them by column name, ask for dict rows — this is worth doing:

```python
import psycopg
from psycopg.rows import dict_row

# option A: for the whole connection
with psycopg.connect(**CONN, row_factory=dict_row) as conn:
    row = conn.execute("SELECT * FROM notes").fetchone()
    print(row["title"])            # by name

# option B: for one cursor only
with conn.cursor(row_factory=dict_row) as cur:
    cur.execute("SELECT * FROM notes")
    print(dict(cur.fetchone()))    # {'id': 1, 'title': 'hello'}
```

> Without `row_factory=dict_row`, `row["title"]` raises
> `TypeError: tuple indices must be integers`. That is the single most common
> psycopg 3 surprise for people arriving from `mysql.connector`.

### Loading a CSV file in bulk

`copy()` returns a context manager. Write the bytes to it in chunks — do **not**
try to pass a file object as the second argument, because that position is for
query parameters and it fails silently:

```python
with open("notes.csv", "rb") as f, psycopg.connect(**CONN) as conn:
    with conn.cursor() as cur:
        with cur.copy("COPY notes (title, body) FROM STDIN WITH (FORMAT CSV, HEADER)") as cp:
            while chunk := f.read(8192):
                cp.write(chunk)
```

To stream results the other way, `COPY ... TO STDOUT` is available as a
read-only query result — but for exporting files, `\copy` in `psql` is simpler.

### Never use f-strings

```python
cur.execute(f"SELECT * FROM notes WHERE title = '{user_input}'")   # ❌ SQL injection
cur.execute("SELECT * FROM notes WHERE title = %s", (user_input,))  # ✅ safe
```

One `%s` placeholder per value, passed as a tuple. An f-string lets a user type
`'; DROP TABLE notes; --` and delete your data.

### Create and drop databases

```python
ADMIN = dict(host="localhost", dbname="postgres", user="postgres", autocommit=True)

with psycopg.connect(**ADMIN) as conn:
    conn.execute("CREATE DATABASE shopdb OWNER alexa")

with psycopg.connect(**ADMIN) as conn:
    conn.execute("DROP DATABASE IF EXISTS shopdb")
```

> `autocommit=True` is **required** here. Without it psycopg opens a
> transaction, and PostgreSQL refuses with *"CREATE DATABASE cannot run inside
> a transaction block"*.

### Transactions

```python
with psycopg.connect(**CONN) as conn:          # commits on success, rolls back on error
    cur = conn.cursor()
    cur.execute("UPDATE accounts SET balance = balance - 100 WHERE id = 1")
    cur.execute("UPDATE accounts SET balance = balance + 100 WHERE id = 2")
    # if either line raises, the whole thing is rolled back
```

Control it yourself:

```python
conn.autocommit = True        # every statement commits immediately
conn.rollback()               # undo the open transaction
conn.close()                  # close the connection
```

### Loading a CSV

See [Loading a CSV file in bulk](#loading-a-csv-file-in-bulk) above.

### Connection pooling

Opening a connection per request is slow. Reuse them:

```bash
pip install "psycopg[pool]"
```

```python
from psycopg_pool import ConnectionPool

with ConnectionPool("postgresql://alexa:yourpassword@localhost:5432/mydb", min_size=1, max_size=10) as pool:
    with pool.connection() as conn:
        cur = conn.execute("SELECT count(*) FROM notes")
        print(cur.fetchone())
```

### Raw SQL in a real app

For anything beyond a script, use an ORM or query builder instead of hand-writing
SQL everywhere:

- **SQLAlchemy** — works with every database, the usual choice
- **Django ORM** — if you use Django, you get it for free
- **psycopg (3) executemany** — for bulk inserts

---

## DBeaver and other GUI tools

A GUI tool uses the same host / port / user / password. It downloads its own
driver file the first time — you are **not** installing a database.

**DBeaver** (free community edition)

1. New Connection → **PostgreSQL**
2. Host `localhost`, Port `5432`
3. Database `mydb`, Authentication: Database Native
4. Username `alexa`, Password `yourpassword`
5. Test Connection → Finish

> DBeaver Community is relational-only. MongoDB, Cassandra, Redis and other
> NoSQL databases need DBeaver PRO. For MongoDB use the official
> [MongoDB Compass](https://www.mongodb.com/products/compass) instead.

**pgAdmin** — the official PostgreSQL GUI, comes with the Windows installer:

```bash
# Arch / Ubuntu are not in the default repos; Docker is easiest
docker run -p 5050:5050 -e PGADMIN_DEFAULT_EMAIL=me@mail.com \
  -e PGADMIN_DEFAULT_PASSWORD=secret -d dpage/pgadmin4
```

**Adminer** — a single PHP file, nothing to install:

```bash
sudo pacman -S php php-embed
mkdir -p /tmp/adminer
wget -O /tmp/adminer/index.php https://www.adminer.org/latest.php
php -S localhost:8080 -t /tmp/adminer
```

Open <http://localhost:8080>, choose PostgreSQL, log in as `alexa`.

---

## Running PostgreSQL with Docker

Useful for throwaway projects and CI — nothing touches your system.

```bash
docker run --name pg \
  -e POSTGRES_USER=alexa \
  -e POSTGRES_PASSWORD=yourpassword \
  -e POSTGRES_DB=mydb \
  -p 5432:5432 \
  -d postgres:18
```

| Command | Does |
|---|---|
| `docker ps` | is it running |
| `docker stop pg` | stop it |
| `docker start pg` | start it again (data survives) |
| `docker exec -it pg psql -U alexa -d mydb` | open a terminal inside it |
| `docker logs pg` | see the log |
| `docker rm -f pg` | **delete it and all its data** |

Keep data across restarts with a volume:

```bash
docker run --name pg -v pgdata:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=yourpassword -p 5432:5432 -d postgres:18
```

docker-compose.yml:

```yaml
services:
  db:
    image: postgres:18
    environment:
      POSTGRES_USER: alexa
      POSTGRES_PASSWORD: yourpassword
      POSTGRES_DB: mydb
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
volumes:
  pgdata:
```

```bash
docker compose up -d      # start
docker compose down       # stop, keep data
```

---

## Start, stop, restart

```bash
sudo systemctl status postgresql     # is it running?
sudo systemctl start postgresql      # start
sudo systemctl stop postgresql       # stop
sudo systemctl restart postgresql    # restart
sudo systemctl reload postgresql     # re-read config, no restart
sudo systemctl enable postgresql     # start automatically on boot
sudo systemctl disable postgresql    # do not start on boot
```

Without systemd:

```bash
sudo -u postgres pg_ctl -D /var/lib/postgres/data status
sudo -u postgres pg_ctl -D /var/lib/postgres/data start
sudo -u postgres pg_ctl -D /var/lib/postgres/data stop
```

> Closing psql with `\q` only ends **your session**. The server keeps running.
> Those are two different things.

macOS with Homebrew:

```bash
brew services start postgresql
brew services stop postgresql
```

---

## Where the files live

| Path | Holds |
|---|---|
| `/var/lib/postgres/data` | **all your data** (Arch) |
| `/var/lib/postgresql/data` | **all your data** (Debian / Ubuntu) |
| `/var/lib/postgres/data/postgresql.conf` | main settings: port, memory, logging |
| `/var/lib/postgres/data/pg_hba.conf` | **who may connect, and how they prove it** |
| `/var/lib/postgres/data/pg_ident.conf` | maps usernames to system users |
| `/etc/postgresql/` | Debian/Ubuntu config overrides |
| `~/.psqlrc` | your personal psql shortcuts (psql only) |

Changing `postgresql.conf` or `pg_hba.conf` needs a reload to take effect:

```bash
sudo systemctl reload postgresql
```

Change `port` or `listen_addresses` and you need a full **restart**.

Where the log lives:

```bash
sudo journalctl -u postgresql -n 50          # systemd installs
sudo tail -f /var/log/postgresql/postgresql.log   # if logging to a file
```

---

## Security checklist

Do these before putting anything real on a server.

**1. Stop using `trust` authentication.**

A fresh `initdb` writes `trust` into `pg_hba.conf`, which means *anyone who can
reach the port can log in as anyone, with any password, or no password.* Fix it:

```bash
sudo nano /var/lib/postgres/data/pg_hba.conf
```

```
# TYPE  DATABASE  USER  ADDRESS         METHOD
local   all       all                   scram-sha-256
host    all       all   127.0.0.1/32    scram-sha-256
host    all       all   ::1/128         scram-sha-256
```

```bash
sudo systemctl reload postgresql
```

`scram-sha-256` is the current password method. `md5` is obsolete, `trust` and
`password` (cleartext) are worse.

**2. Make sure it only listens locally.**

In `postgresql.conf`:

```
listen_addresses = 'localhost'
```

**3. Never use the `postgres` superuser from application code.** Create a role
per application and grant only what it needs.

**4. Keep the password out of your code.** Use an environment variable:

```python
import os
password = os.environ["PGPASSWORD"]
```

```bash
echo "export PGPASSWORD=yourpassword" >> ~/.bashrc
```

Add `.env` and `*.sql` dumps to `.gitignore`. **Never commit a password.**

**5. Encrypt traffic** if the database is not on localhost:

```
# postgresql.conf
ssl = on
```

**6. Back up regularly**, and actually test that you can restore.

**7. Update.** `apt upgrade`, or `sudo pacman -Syu`.

---

## Coming from MySQL

| MySQL | PostgreSQL | Note |
|---|---|---|
| `SHOW DATABASES;` | `SELECT datname FROM pg_database;` | `SHOW DATABASES` is a **syntax error** in Postgres |
| `SHOW TABLES;` | `SELECT tablename FROM pg_tables WHERE schemaname = 'public';` | |
| `SHOW DATABASES` / `\l` | `\l` | backslash commands are psql-only, not SQL |
| `USE mydb;` | `\c mydb` | psql only; in an app, connect to that database instead |
| `DESCRIBE notes;` | `\d notes` | or `SELECT * FROM notes LIMIT 0` |
| `AUTO_INCREMENT` | `SERIAL` or `GENERATED ALWAYS AS IDENTITY` | |
| `ENGINE=InnoDB` | nothing needed | PostgreSQL has one engine |
| `AUTO_INCREMENT=1000` in table options | `ALTER SEQUENCE notes_id_seq RESTART WITH 1000;` | |
| `TINYINT(1)` for boolean | `BOOLEAN` | |
| `DATETIME` | `TIMESTAMP` or `TIMESTAMPTZ` | |
| `BLOB` | `BYTEA` | or better, a URL to a file |
| backticks `` `col` `` | `"col"` | double quotes, only when needed |
| `CONCAT(a, b)` | `a \|\| b` | `CONCAT()` also works |
| `NOW()` | `now()` | case does not matter for functions |
| `RAND()` | `random()` | |
| `IFNULL(a, b)` | `COALESCE(a, b)` | |
| `GROUP_CONCAT(x)` | `string_agg(x, ',')` | |
| `LIMIT 0, 10` | `LIMIT 10 OFFSET 0` | order is reversed |
| `FOUND_ROWS()` | not available — use `count(*)` in a subquery | |
| `INSERT IGNORE` | `INSERT ... ON CONFLICT DO NOTHING` | |
| `ON DUPLICATE KEY UPDATE` | `INSERT ... ON CONFLICT (col) DO UPDATE SET ...` | |
| `mysqldump` | `pg_dump` | |
| `mysql` CLI | `psql` or `pgcli` | |

The single most common trip-up: **`SHOW DATABASES` and `USE db` are client
commands, not SQL.** They do not exist in PostgreSQL, and you will get
`unrecognized configuration parameter` or a syntax error if you send them to a
real database driver.

---

## Common errors and how to fix them

| Error message | What it means | Fix |
|---|---|---|
| `could not create directory ... Permission denied` | `initdb` cannot write there | `sudo mkdir -p /var/lib/postgres && sudo chown postgres:postgres /var/lib/postgres` |
| `"/var/lib/postgres/data" is missing or empty` | service looks in a different path than you initialised | `systemctl show postgresql -p Environment` to see the real path |
| `FATAL: role "alexa" does not exist` | wrong username, or not connected as you think | `\conninfo` to check; `sudo -u postgres psql -c '\du'` to list roles |
| `FATAL: database "mydb" does not exist` | typo, or never created | `\l` to list them |
| `FATAL: permission denied for schema public` | you are not the database owner (PG 15+ change) | make the database yours, or `GRANT ALL ON SCHEMA public TO alexa;` |
| `ERROR: permission denied for database mydb` | missing CONNECT privilege | `GRANT CONNECT ON DATABASE mydb TO alexa;` |
| `permission denied for table notes` | missing table privilege | `GRANT ALL ON notes TO alexa;` |
| `ERROR: must be owner of table notes` | only the owner may alter a table | `ALTER TABLE notes OWNER TO alexa;` |
| `ERROR: syntax error at or near "notes"` | missing semicolon between statements | end every statement with `;` |
| `invalid integer value "notes" for connection option "port"` | you pasted multiple lines into the psql prompt | run statements one at a time, or use `psql -f file.sql` |
| `unrecognized configuration parameter "databases"` | you used MySQL's `SHOW DATABASES` | `SELECT datname FROM pg_database;` |
| `null value in column "x" violates not-null constraint` | you inserted without a required value | supply it, or give the column a `DEFAULT` |
| `duplicate key value violates unique constraint` | that value already exists | use a different value, or `ON CONFLICT DO NOTHING` |
| `CREATE DATABASE cannot run inside a transaction block` | driver opened a transaction | in psycopg, pass `autocommit=True` |
| `too many connections for role` | you opened connections and never closed them | use a pool; check `max_connections` in `postgresql.conf` |
| `server closed the connection unexpectedly` | crashed, ran out of memory, or was killed | `sudo journalctl -u postgresql -n 50` |
| `could not translate host name` | wrong host, or DNS problem | use `127.0.0.1` instead of `localhost` |
| `connection refused` | the server is not running | `sudo systemctl start postgresql` |
| `no pg_hba.conf entry for host` | `pg_hba.conf` has no matching rule | add a line, then `sudo systemctl reload postgresql` |

---

## The one-page cheat sheet

Print this. It is the 5% you use daily.

```sql
-- CONNECT
psql -h localhost -U alexa -d mydb

-- LOOK AROUND
\l                    list databases
\dt                   list tables
\d notes              describe a table
\du                   list roles
\conninfo             who am I

-- CREATE
CREATE DATABASE shopdb OWNER alexa;
CREATE TABLE notes (id SERIAL PRIMARY KEY, title TEXT NOT NULL, body TEXT);
CREATE INDEX idx_notes_title ON notes (title);
CREATE VIEW v AS SELECT * FROM notes;

-- WRITE
INSERT INTO notes (title, body) VALUES ('a', 'b');
UPDATE notes SET body = 'x' WHERE id = 1;
DELETE FROM notes WHERE id = 1;

-- READ
SELECT * FROM notes;
SELECT * FROM notes WHERE title LIKE 'a%' ORDER BY id DESC LIMIT 10;
SELECT count(*) FROM notes;
SELECT a, count(*) FROM notes GROUP BY a HAVING count(*) > 1;

-- COMBINE
SELECT * FROM a JOIN b ON a.id = b.a_id;

-- SAFE CHANGES
BEGIN;
  -- statements
COMMIT;        -- or ROLLBACK;

-- DESTROY  (no WHERE = everything!)
TRUNCATE notes;
DROP TABLE IF EXISTS notes;
DROP DATABASE IF EXISTS shopdb;
DROP ROLE IF EXISTS alexa;
```

```bash
# SERVER
sudo systemctl status|start|stop|restart|reload postgresql
sudo journalctl -u postgresql -n 50
pg_dump -h localhost -U alexa -Fc -d mydb -f mydb.dump
pg_restore -h localhost -U alexa -d mydb --clean --if-exists -d mydb.dump
```

---

## Glossary

| Word | Meaning |
|---|---|
| **Cluster** | the data directory holding one PostgreSQL server's data |
| **Role** | a user. May or may not be able to log in |
| **Privilege** | a permission like `SELECT` or `INSERT` |
| **Grantor / grantee** | who gave a permission / who received it |
| **Sequence** | the counter behind `SERIAL` and identity columns |
| **CTE** | `WITH name AS (...)` — a named subquery |
| **Index** | a sorted lookup structure that makes searches fast |
| **View** | a saved query with a name |
| **Materialized view** | a view whose results are stored on disk |
| **Vacuum** | housekeeping: reclaims dead space, refreshes statistics |
| **Autovacuum** | PostgreSQL runs vacuum for you automatically |
| **Tuple** | PostgreSQL's word for a row |
| **MVCC** | lets readers work while someone else writes, without blocking |
| **WAL** | write-ahead log — how it survives crashes |
| **Toast** | the internal system that stores very large values |
| **Schema** | a namespace of tables inside a database |
| **Superuser** | unrestricted `postgres`; avoid it in applications |
| **psql** | the command-line client that ships with PostgreSQL |
| **pgcli** | a nicer command-line client |
| **psycopg** | the Python driver |
| **libpq** | the C driver underneath most language bindings |
| **PgBouncer** | a lightweight connection pooler for heavy workloads |

---

## Contributing

Found something wrong or missing? Open an issue or send a pull request —
improvements are welcome.

## License

MIT. Use it however you like.
