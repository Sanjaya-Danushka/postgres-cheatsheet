================================================================================
  PostgreSQL CHEAT SHEET
  Server: localhost:5432   User: alexa   Database: mydb
  Tool: pgcli   (better psql - autocomplete, colors, pretty tables)
================================================================================


--------------------------------------------------------------------------------
 1. CONNECTING
--------------------------------------------------------------------------------

  pgcli -h localhost -U alexa -d mydb         open your database
  pgcli -h localhost -U alexa -d postgres     open postgres db (to create others)
  pgcli -h localhost -U alexa -l              list all databases, then exit
  pgcli -h localhost -U alexa -d mydb -W     force password prompt
  sudo -u postgres pgcli                     as superuser (no password needed)

  INSIDE PGCLI
  \q                  quit
  \c mydb             switch to another database
  \l                  list databases
  \du                 list roles (users)
  \dt                 list tables in current database
  \d notes            describe a table: columns, types, defaults, indexes
  \d+ notes           same, more detail
  \x                  toggle expanded / vertical output
  \e                  open your current query in $EDITOR
  \dp notes           show permissions on a table
  \conninfo           show who you're connected as

  Keys:  Tab = autocomplete      F2 = smart completion
         F3 = multiline mode      F5 = explain the query


--------------------------------------------------------------------------------
 2. DATABASES
--------------------------------------------------------------------------------

  -- create
  CREATE DATABASE shopdb OWNER alexa;

  -- change owner / rename
  ALTER DATABASE shopdb OWNER TO alexa;
  ALTER DATABASE shopdb RENAME TO store;

  -- remove
  DROP DATABASE shopdb;
  DROP DATABASE IF EXISTS shopdb;
  DROP DATABASE shopdb WITH (FORCE);   -- disconnect anyone using it first

  NOTE: you must be connected to a DIFFERENT database (usually "postgres")
        to create or drop one. You cannot drop the one you are inside.


--------------------------------------------------------------------------------
 3. ROLES (USERS)
--------------------------------------------------------------------------------

  -- create
  CREATE ROLE dev WITH LOGIN PASSWORD 'secret';
  CREATE USER dev2 WITH PASSWORD 'secret2';   -- same thing, shorthand

  -- change
  ALTER ROLE alexa CREATEDB;            -- allow making databases
  ALTER ROLE alexa CREATEDB NOCREATEDB; -- take that away
  ALTER ROLE alexa WITH PASSWORD 'newpass';

  -- give another role
  GRANT pg_read_all_data TO alexa;     -- built-in "read everything" role
  GRANT my_role TO alexa;

  -- remove
  DROP ROLE dev;


--------------------------------------------------------------------------------
 4. TABLES
--------------------------------------------------------------------------------

  -- create
  CREATE TABLE notes (
    id         SERIAL PRIMARY KEY,
    title      TEXT NOT NULL,
    body       TEXT,
    created_at TIMESTAMPTZ DEFAULT now()
  );

  -- clone structure only
  CREATE TABLE notes2 (LIKE notes INCLUDING ALL);

  -- rename / empty / remove
  ALTER TABLE notes RENAME TO memos;
  TRUNCATE notes;                  -- delete all rows, keep table
  TRUNCATE notes RESTART IDENTITY; -- also reset the id counter
  DROP TABLE notes;
  DROP TABLE IF EXISTS notes;
  DROP TABLE notes CASCADE;        -- also drop things that depend on it

  TYPES YOU WILL USE
  SERIAL         auto-incrementing integer (creates a sequence)
  TEXT           any length string
  VARCHAR(100)   max 100 chars
  INT            whole number
  NUMERIC(10,2)  exact money / decimals
  BOOLEAN        true / false
  TIMESTAMPTZ    date + time with timezone


--------------------------------------------------------------------------------
 5. COLUMNS
--------------------------------------------------------------------------------

  ALTER TABLE notes ADD COLUMN tag TEXT;
  ALTER TABLE notes DROP COLUMN tag;
  ALTER TABLE notes RENAME COLUMN body TO content;
  ALTER TABLE notes ALTER COLUMN title TYPE VARCHAR(200);
  ALTER TABLE notes ALTER COLUMN title SET NOT NULL;
  ALTER TABLE notes ALTER COLUMN title DROP NOT NULL;
  ALTER TABLE notes ADD CONSTRAINT notes_title_uniq UNIQUE (title);


--------------------------------------------------------------------------------
 6. DATA
--------------------------------------------------------------------------------

  -- read
  SELECT * FROM notes;
  SELECT * FROM notes LIMIT 10;
  SELECT * FROM notes WHERE title LIKE 'a%' ORDER BY id DESC;
  SELECT count(*) FROM notes;

  -- write
  INSERT INTO notes (title, body) VALUES ('first', 'hello');
  INSERT INTO notes (title) SELECT title FROM notes;    -- copy rows
  UPDATE notes SET body = 'new text' WHERE id = 1;
  DELETE FROM notes WHERE id = 1;

  -- join two tables
  SELECT c.name, o.total
  FROM orders o
  JOIN customers c ON c.id = o.customer_id;

  -- reset an auto-increment counter
  SELECT setval('notes_id_seq', 1);

  WARNING: DELETE FROM notes;   with no WHERE deletes EVERY row. No undo.


--------------------------------------------------------------------------------
 7. PERMISSIONS
--------------------------------------------------------------------------------

  GRANT CONNECT ON DATABASE mydb TO alexa;
  GRANT ALL ON SCHEMA public TO alexa;
  GRANT ALL ON ALL TABLES IN SCHEMA public TO alexa;
  GRANT SELECT, INSERT ON notes TO alexa;
  REVOKE ALL ON notes FROM alexa;
  ALTER TABLE notes OWNER TO alexa;

  -- apply to every table you create from now on
  ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT ON TABLES TO alexa;

  VIEW: \dp notes    \z notes    \dn+


--------------------------------------------------------------------------------
 8. OTHER THINGS YOU CAN LIST
--------------------------------------------------------------------------------

  \dn      schemas          \dv   views         \di   indexes
  \ds      sequences        \df   functions     \dt   tables
  \dT      types            \dx   extensions    \d    describe anything


--------------------------------------------------------------------------------
 9. FROM PYTHON  (driver: psycopg, NOT mysql.connector)
--------------------------------------------------------------------------------

  Install (isolated, recommended):
      mkdir myproject && cd myproject
      python -m venv .venv
      source .venv/bin/activate
      pip install "psycopg[binary]"

  Install (system-wide from Arch repo):
      sudo pacman -S python-psycopg

  CODE
      import psycopg

      ADMIN = dict(host="localhost", dbname="postgres",
                   user="postgres", autocommit=True)
      APP   = dict(host="localhost", dbname="mydb",
                   user="alexa", password="yourpassword", autocommit=True)

      # create a database
      with psycopg.connect(**ADMIN) as conn:
          conn.execute("CREATE DATABASE shopdb OWNER alexa")

      # create a table + insert + read
      with psycopg.connect(**APP) as conn:
          conn.execute("""
              CREATE TABLE notes (
                  id    SERIAL PRIMARY KEY,
                  title TEXT NOT NULL,
                  body  TEXT
              )
          """)
          conn.execute(
              "INSERT INTO notes (title, body) VALUES (%s, %s) RETURNING id",
              ("hello", "world")
          )
          print(conn.execute("SELECT * FROM notes").fetchall())

      # drop a table
      with psycopg.connect(**APP) as conn:
          conn.execute("DROP TABLE IF EXISTS notes")

      # drop a database
      with psycopg.connect(**ADMIN) as conn:
          conn.execute("DROP DATABASE IF EXISTS shopdb")

  RULES
    * autocommit=True is REQUIRED for CREATE/DROP DATABASE.
      Without it psycopg opens a transaction and Postgres refuses.
    * Use %s placeholders, never f-strings. That is how you avoid SQL injection.
    * The "postgres" user has no password prompt on this machine
      because local auth is set to "trust" (see section 10).


--------------------------------------------------------------------------------
10. SERVER CONTROL  (run in a normal shell, not inside pgcli)
--------------------------------------------------------------------------------

  sudo systemctl status postgresql     # is it running?
  sudo systemctl start postgresql      # start
  sudo systemctl stop postgresql       # shut down
  sudo systemctl restart postgresql    # restart
  sudo systemctl enable postgresql     # already done: starts on boot

  \q only closes YOUR session. The server keeps running.

  Edit config:  sudo nano /var/lib/postgres/data/postgresql.conf
  Edit logins:  sudo nano /var/lib/postgres/data/pg_hba.conf
  Reload config without restart:  sudo systemctl reload postgresql

  WHERE THINGS LIVE
  Data:          /var/lib/postgres/data
  Config:        /var/lib/postgres/data/postgresql.conf
  Auth rules:    /var/lib/postgres/data/pg_hba.conf
  Logs:          sudo journalctl -u postgresql -n 50


--------------------------------------------------------------------------------
11. GUI OPTIONS  (both use the same host/user/password as pgcli)
--------------------------------------------------------------------------------

  DBeaver  - already installed. New Connection > PostgreSQL
             host localhost, port 5432, user alexa, database mydb
             It downloads its own JDBC driver, no database install needed.

  Adminer  - browser-based, no install
             sudo pacman -S php php-embed
             mkdir -p /tmp/adminer
             wget -O /tmp/adminer/index.php https://www.adminer.org/latest.php
             php -S localhost:8080 -t /tmp/adminer
             then open http://localhost:8080


--------------------------------------------------------------------------------
12. REMOVING EVERYTHING
--------------------------------------------------------------------------------

  DROP TABLE IF EXISTS notes, orders CASCADE;   -- inside mydb
  DROP DATABASE IF EXISTS shopdb;
  DROP ROLE IF EXISTS alexa;


--------------------------------------------------------------------------------
13. TWO WARNINGS ABOUT YOUR SETUP
--------------------------------------------------------------------------------

  * pg_hba.conf is set to "trust" - passwords are NOT actually checked, and any
    program on this machine can connect as alexa. To turn on real password
    checks, edit /var/lib/postgres/data/pg_hba.conf and change these lines:

        local   all   all                 scram-sha-256
        host    all   all   127.0.0.1/32  scram-sha-256
        host    all   all   ::1/128       scram-sha-256

    then:  sudo systemctl reload postgresql

  * role "alexa" has no CREATEDB yet, so CREATE DATABASE will fail until you run:
        ALTER ROLE alexa CREATEDB;

================================================================================
