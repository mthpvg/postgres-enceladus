# postgres-enceladus

## Setup
- https://postgresapp.com/

## bash command
```bash
psql
createdb enceladus
dropdb enceladus
```

### psql

Connect
```bash
\c enceladus
```

Help
```bash
\h
```

Quit
```bash
\q
```

Create table
```SQL
create table mission_plan(
  the_date date,
  title varchar(100),
  description text
);
```

Drop table
```SQL
drop table mission_plan;
```

Better drop table, to avoid errors
```SQL
drop table if exists mission_plan;
```

Create table with a primary key.
It creates an auto-incrementing integer key.
```SQL
create table mission_plan(
  id serial primary key,
  the_date date,
  title varchar(100),
  description text
);
```

Rules for tables in Postgres are called *constraints*.

ETL: Extract Transform an Load.

Idempotent script. Idempotence: the quality of having the same power.
Looking up a customer's name and address in a database is typically idempotent.
Placing an order for a car for the customer is typically not idempotent.
Wikipedia

Import everything as text. Add types later on. You want the data in the database first, everything else can wait.
Set everything to the text data type, refining the types later on. Makes sense: Postgres is probably much better and tweaking the data than Excel is.

Load the table
```SQL
-- build.sql
drop table if exists master_plan; create table master_plan(
  start_time_utc text,
  duration text,
  date text,
  team text,
  spass_type text,
  target text,
  request_name text,
  library_definition text,
  title text,
  description text
);
```

```bash
psql enceladus -f build.sql
```

```SQL
COPY master_plan FROM '/Users/mthpvg/Dropbox/books/rob-conery/a-curious-moon/book_and_data/code_and_data/data/master_plan.csv' WITH DELIMITER ',' HEADER CSV;
```

Schemas are where every single relation and object in your database are stored. You can think of them as a “namespace” if that helps.

BRB: Be Right Back

Schema:
- The cluster. This is a set of servers (usually just one) that execute the instructions.
- The database.
- One or more schemas. The default is public.
- Tables, views, functions and other relations. These are all attached to a schema.

```SQL
-- psql enceladus -f build2.sql
-- build2.sql
create schema if not exists import;
drop table if exists import.master_plan;
create table import.master_plan(
  start_time_utc text,
  duration text,
  date text,
  team text,
  spass_type text,
  target text,
  request_name text,
  library_definition text, title text,
  description text
);
COPY import.master_plan
FROM '/Users/mthpvg/Dropbox/books/rob-conery/a-curious-moon/book_and_data/code_and_data/data/master_plan.csv' WITH DELIMITER ',' HEADER CSV;
```

```Makefile
DB=enceladus
BUILD=${CURDIR}/build.sql
SCRIPTS=${CURDIR}/scripts
CSV='${CURDIR}/data/master_plan.csv'
MASTER=$(SCRIPTS)/import.sql
NORMALIZE=$(SCRIPTS)/normalize.sql

all: normalize
  psql $(DB) -f $(BUILD)

master:
  @cat $(MASTER) >> $(BUILD)

import: master
  @echo "COPY import.master_plan FROM $(CSV) WITH DELIMITER ',' HEADER CSV;" >> $(BUILD)

normalize: import
  @cat $(NORMALIZE) >> $(BUILD)

clean:
  @rm -rf $(BUILD)
```

The neat thing about using Make is that I can now organize the various SQL commands into separate files, just like a typical software program.


To define:
- Fact/Source table have lookups ids
- Lookup table

Create event table in the public schema.
```SQL
create table events(
  id serial primary key,
  time_stamp timestamptz not null,
  title varchar(500),
  description text,
  event_type_id int,
  spass_type_id int,
  target_id int,
  team_id int,
  request_id int
);
```

Insert some keys from an existing table:
```SQL
insert into events(
  time_stamp,
  title,
  description
)
select
  import.master_plan.date,
  import.master_plan.title,
  import.master_plan.description
from import.master_plan;
```
It produces an error: `ERROR:  column "time_stamp" is of type timestamp with time zone but expression is of type text`

Instead we cast:
```SQL
insert into events(
  time_stamp,
  title,
  description
)
select
  import.master_plan.date::timestamptz,
  import.master_plan.title,
  import.master_plan.description
from import.master_plan;
```
Also an error: `ERROR:  date/time field value out of range: "29-Feb-14"`

**The first rule: dates are always a source of pain.**

```SQL
select date from import.master_plan limit 20;
```
It outputs:
```
date
-----------
14-May-04
14-May-04
14-May-04
14-May-04
14-May-04
14-May-04
14-May-04
14-May-04
15-May-04
15-May-04
15-May-04
15-May-04
15-May-04
15-May-04
15-May-04
15-May-04
15-May-04
15-May-04
15-May-04
15-May-04
(20 rows)
```
Is that date May 14, 2004, or is it May 04, 2014?

```SQL
select '2000-01-01'::timestamptz;
```
```
timestamptz
------------------------
2000-01-01 00:00:00+01
(1 row)
```

Again:
```SQL
select '2000-01-01'::timestamptz at time zone 'UTC';
```
```
timezone
---------------------
1999-12-31 23:00:00
(1 row)
```

New iteration
```SQL
insert into events(
  time_stamp,
  title,
  description
)
select
  import.master_plan.start_time_utc::timestamptz at time zone 'UTC',
  import.master_plan.title,
  import.master_plan.description
from import.master_plan;
```

Data loss like this can happen in only one of two ways: `where` clauses and `joins` are the only parts of SQL statements which can **filter** or remove data.
