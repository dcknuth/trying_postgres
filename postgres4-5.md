# Following the PostgreSQL Documentation 4-5

## Section 4 - SQL Syntax
This covers the tutorial section starting [here](https://www.postgresql.org/docs/current/sql-syntax-lexical.html). Let's jump in.
```sql
-- again, we are going to need a table to run the examples
CREATE TABLE my_table (
    a       int,
    my_data text
);
-- most items outside of quotes are case-insensitive
UPDATE MY_TABLE SET A = 5;
uPDATE my_TabLE SeT a = 5;
-- but by convention SQL key words are in UPPER and other stuff is in lower
UPDATE my_table SET a = 5;
-- you can double quote identifiers to make them case sensitive or include
--  other things (like spaces) into the identifier
UPDATE "MY_TABLE" SET a = 5;
-- errors on MY_TABLE now like it should
-- string constants are in single quotes
SELECT 'foo';
-- two strings on the same line is bad, but with a newline is treated
--  as a continuation of the string
SELECT 'foo'
    'bar';
-- you can use an escape extension by starting with E
SELECT E'foo\nbarr';
SELECT E'\u0441\u043B\u043E\u043D';
-- first one worked, but the second did not or can't be printed to the
--  screen in the current language. Let's try again
SELECT U&'\0441\043B\043E\043D';
-- same result, but seems like it could be stored and printed with a
--  matching language. Unicode is hard
-- dollar quoting as an alternative to quotes
SELECT $$Dianne's horse$$;
SELECT $SomeTag$Dianne's horse$SomeTag$;
-- bit string constants
SELECT B'1001';
-- or in hex
SELECT X'1FF';
-- like this to get to an int
INSERT INTO my_table VALUES (0b1001, 'testing');
SELECT * FROM my_table;
-- or x for hex or o for octal
-- underscores can be used for visible grouping of large ints
INSERT INTO my_table VALUES(1_500_000_000, 'more testing');
SELECT * FROM my_table;
-- casting from a string constant
INSERT INTO my_table VALUES (CAST ( B'1001' AS INT ), 'retest');
SELECT * FROM my_table;
/* using a multi-line comment to say we are skipping the operators
and special characters sections
comment /* nesting */ is OK */
-- also skipping operator precedence and the next section for lack of
--  runnable examples
```
We pick back up at Array Constructors
```sql
SELECT ARRAY[1,2,3+4];
-- casting an array
SELECT ARRAY[1,2,22.7]::integer[];
-- note that the default behavior is to round the real
-- 2D array
SELECT ARRAY[[1,2], [3,4]];
-- there can be interesting combinations of arrays as long as dimensions
--  match
CREATE TABLE arr (
    f1 int[],
    f2 int[]
);
INSERT INTO arr VALUES (ARRAY[[1,2],[3,4]], ARRAY[[5,6],[7,8]]);
SELECT ARRAY[f1, f2, '{{9,10},{11,12}}'::int[]] FROM arr;
-- you can turn a selection into an array
SELECT ARRAY(SELECT elevation FROM cities);
SELECT ARRAY(SELECT ARRAY[i, i*2] FROM generate_series(1,5) AS a(i));
-- didn't know there was a generate_series function
-- 'ROW' makes a row/composite (values of different types)
SELECT ROW(1,2.5,'this is a test');
CREATE TABLE mytable (
    f1 int,
    f2 float,
    f3 text
);
CREATE FUNCTION getf1(mytable)
    RETURNS int AS 'SELECT $1.f1'
    LANGUAGE SQL;
SELECT getf1(ROW(1,2.5,'this is a test'));
CREATE TYPE myrowtype AS (f1 int, f2 text, f3 numeric);
CREATE FUNCTION getf1(myrowtype)
    RETURNS int AS 'SELECT $1.f1'
    LANGUAGE SQL;
-- now need a cast to tell what to call
SELECT getf1(ROW(1,2.5,'this is a test'));
-- failed, now with cast
SELECT getf1(ROW(1,2.5,'this is a test')::mytable);
SELECT getf1(CAST(ROW(11,'this is a test', 2.5) AS myrowtype));
-- compare two rows
SELECT ROW(1,2.5,'this is a test') = ROW(1, 3, 'not the same');
-- returns 'f'
```
Functions
```sql
-- create a test function
CREATE FUNCTION concat_lower_or_upper(a text, b text,
    uppercase boolean DEFAULT false)
    RETURNS text AS
    $$
        SELECT CASE
            WHEN $3 THEN UPPER($1 || ' ' || $2)
            ELSE LOWER($1 || ' ' || $2)
            END;
    $$
    LANGUAGE SQL IMMUTABLE STRICT;
-- positional notation
SELECT concat_lower_or_upper('Hello', 'World', true);
SELECT concat_lower_or_upper('Hello', 'World');
-- named notation
SELECT concat_lower_or_upper(a => 'Hello', b => 'World');
SELECT concat_lower_or_upper(a => 'Hello', uppercase => true, b => 'World');
-- := could also be used to assign, but is older
-- mixed notation
SELECT concat_lower_or_upper('Hello', 'World', uppercase => true);

-- TODO, there are still a couple of items worth doing at the end of 4
```

## Section 5 - Data Definition
This covers the tutorial section starting [here](https://www.postgresql.org/docs/current/ddl-basics.html)

### Table Basics
```sql
CREATE TABLE my_first_table (
    first_column    text,
    second_column   integer
);
CREATE TABLE products (
    product_no      integer,
    name            text,
    price           numeric
);
-- remove the first table
DROP TABLE my_first_table;
-- drop if exists (non standard)
DROP TABLE IF EXISTS products;
```

### Default Values
```sql
CREATE TABLE products (
    product_no  integer,
    name        text,
    price       numeric DEFAULT 9.99
);
DROP TABLE products;
-- SERIAL will create a default that is unique and increasing
CREATE TABLE products (
    product_no  SERIAL,
    name        text
);
INSERT INTO products (name) VALUES ('carrots');
INSERT INTO products (name) VALUES ('milk');
SELECT * FROM products;
-- the product_no takes care of itself
```

### Generated Columns
```sql
CREATE TABLE people (
    name        text,
    height_cm   numeric,
    height_in   numeric GENERATED ALWAYS AS (height_cm / 2.54) STORED
);
INSERT INTO people VALUES('Dave', 200);
SELECT * FROM people;
-- I'm not actually that tall
```

### Constraints
```sql
DROP TABLE products;
CREATE TABLE products (
    product_no  integer,
    name        text,
    price       numeric CHECK (price > 0)
);
INSERT INTO products VALUES (1, 'carrot', -1.50);
-- insert fails during the constraint check
-- you can also provide a name for your constraint for better error
--  messaging
DROP TABLE products;
CREATE TABLE products (
    product_no  integer,
    name        text,
    price       numeric CONSTRAINT positive_price CHECK (price > 0)
);
INSERT INTO products VALUES (1, 'carrot', -1.50);
-- another example using multiple columns
DROP TABLE products;
CREATE TABLE products (
    product_no  integer,
    name        text,
    price       numeric CONSTRAINT positive_price CHECK (price > 0),
    discounted_price numeric,
    CHECK (discounted_price < price AND discounted_price > 0)
);
INSERT INTO products VALUES (1, 'carrot', 1.50, 1.25);
INSERT INTO products VALUES (2, 'milk', 3.00, 3.50);
-- we used two different ways to provide the checks, with and separate
--  from the column declaration
-- non NULL constraint
DROP TABLE products;
CREATE TABLE products (
    product_no  integer NOT NULL,
    name        text NOT NULL,
    price       numeric
);
INSERT INTO products VALUES (NULL, 'carrot', 1.50);
-- fails as expected
-- this (below) assumes the DEFAULT is NULL (for price)
DROP TABLE products;
CREATE TABLE products (
    product_no  integer NOT NULL,
    name        text NOT NULL,
    price       numeric NULL
);
INSERT INTO products VALUES (1, 'carrot');
SELECT * FROM products;

-- Unique Constraints
DROP TABLE products;
CREATE TABLE products (
    product_no  integer UNIQUE,
    -- or could be: UNIQUE (product_no)
    name        text,
    price       numeric
);
INSERT INTO products VALUES (1, 'carrot');
INSERT INTO products VALUES (1, 'milk');
-- fails the constraint for a unique product_no
-- another way to describe the constraint for multiple columns
CREATE TABLE example (
    a   integer,
    b   integer,
    c   integer,
    UNIQUE (a, c)
);
INSERT INTO example VALUES (1, 2, 3);
-- this does not mean that each column is unique on its own
INSERT INTO example VALUES (4, 5, 3);
-- but that the collection of the two together is unique
INSERT INTO example VALUES (4, 6, 3);
-- only now does it fail the constraint with an a,c 4,3 already there
DROP TABLE products, example;
-- you can also name a unique constraint
CREATE TABLE products (
    product_no  integer CONSTRAINT must_be_unique UNIQUE,
    name        text,
    price       numeric
);
INSERT INTO products VALUES (1, 'carrot');
INSERT INTO products VALUES (1, 'milk');
DROP TABLE products;
-- NULLs can be repeated unless you use this form
CREATE TABLE products (
    product_no  integer UNIQUE NULLS NOT DISTINCT,
    name        text,
    price       numeric
);
INSERT INTO products VALUES (NULL, 'carrot');
INSERT INTO products VALUES (NULL, 'milk');

-- Primary Keys
-- primary keys are both unique and not null
DROP TABLE products;
CREATE TABLE products (
    product_no  integer PRIMARY KEY,
    name        text,
    price       numeric
);
-- each table can only have one primary key, but that key can span
--  multiple columns in the same way as unique
CREATE TABLE example (
    a   int,
    b   int,
    c   int,
    PRIMARY KEY (a, c)
);
-- in either a unique or a primary key a b-tree is created alongside
--  to manage those column/s

-- Foreign Keys
-- a foreign key is a column in another table
CREATE TABLE orders (
    order_id    integer PRIMARY KEY,
    product_no  integer REFERENCES products (product_no),
    quantity    integer
);
-- now an order can only get inserted if the product_no exists in the
--  products table
-- since the same identifier was used in both tables, it could have been
--  shortened to: product_no integer REFERENCES products,
-- you can refer to multiple columns
DROP TABLE example;
CREATE TABLE example (
    c1  integer,
    c2  integer,
    UNIQUE (c1, c2)
);
CREATE TABLE t1 (
    a   integer PRIMARY KEY,
    b   integer,
    c   integer,
    FOREIGN KEY (b, c) REFERENCES example (c1, c2)
);
-- the columns pointed to need to be UNIQUE
DROP TABLE t1;
-- also, you can name the relation, but it needs to have this format:
CREATE TABLE t1 (
    a   integer PRIMARY KEY,
    b   integer,
    c   integer,
    CONSTRAINT to_example
        FOREIGN KEY (b, c) REFERENCES example (c1, c2)
);
-- references can be self-referential and point to itself
CREATE TABLE tree (
    node_id     integer PRIMARY KEY,
    parent_id   integer REFERENCES tree,
    name        text
);
-- this works without: REFERENCES tree (node_id) because there is only
--  one UNIQUE key that it could be, the primary key
-- in this case, the top level of the tree would reference NULL
-- to allow one to many relations
DROP TABLE orders;
CREATE TABLE orders (
    order_id            integer PRIMARY KEY,
    shipping_address    text
);
CREATE TABLE order_items (
    product_no  integer REFERENCES products,
    order_id    integer REFERENCES orders,
    quantity    integer,
    PRIMARY KEY (product_no, order_id)
);
-- the primary key is self referential to two different foreign keys in
--  two tables
-- what do we do about deletion when there are things that reference them
-- by default, there is a warning and you have to add something like
--  'CASCADE'. You can also define the behavior in the table
DROP TABLE order_items;
CREATE TABLE order_items (
    product_no integer REFERENCES products ON DELETE RESTRICT,
    order_id integer REFERENCES orders ON DELETE CASCADE,
    quantity integer,
    PRIMARY KEY (product_no, order_id)
);
-- add some stuff
INSERT INTO products VALUES (1, 'carrot', 1.00);
INSERT INTO orders VALUES (101, 'my house');
INSERT INTO order_items VALUES (1, 101, 3);
-- try to delete product, should be refused
DELETE FROM products WHERE product_no = 1;
-- errors as expected
-- this should automatically do a cascade
DELETE FROM orders WHERE order_id = 101;
SELECT * FROM orders;
SELECT * FROM order_items;
-- deleted from both tables with no error
-- example of a conflicting constraint
CREATE TABLE tenants (
    tenant_id integer PRIMARY KEY
);
CREATE TABLE users (
    tenant_id   integer REFERENCES tenants ON DELETE CASCADE,
    user_id     integer NOT NULL,
    PRIMARY KEY (tenant_id, user_id)
);
CREATE TABLE posts (
    tenant_id   integer REFERENCES tenants ON DELETE CASCADE,
    post_id     integer NOT NULL,
    author_id   integer,
    PRIMARY KEY (tenant_id, post_id),
    FOREIGN KEY (tenant_id, author_id)
        REFERENCES users ON DELETE SET NULL (author_id)
);
INSERT INTO tenants VALUES(1);
INSERT INTO users VALUES(1, 10);
INSERT INTO posts VALUES(1, 101, 10);
DELETE FROM tenants WHERE tenant_id = 1;
SELECT * FROM tenants;
SELECT * FROM users;
SELECT * FROM posts;
-- that deleted everything
-- the docs sound like the behavior in complex definitions can get
--  tough to follow

-- Exclusion Constraints
CREATE TABLE circles (
    c   circle,
    EXCLUDE USING gist (c WITH &&)
);
-- WTF?
-- there seems to be a data type of circle in Postgres that has a 2D point
--  and a radius. Then, there is a search tree called a Generalized Search
--  tree that is selected by 'gist'. && is a geometric overlap operator.
--  So... this will exclude the entry of any circle that overlaps with one
--  already in the table. We have to try this
-- enter a big circle at 0,0
INSERT INTO circles VALUES ('(0,0),100');
SELECT * FROM circles;
-- OK, that one is in
INSERT INTO circles VALUES ('(2,2),1');
-- sure enough, it blocks us from inserting an overlapping circle
-- can we put in a non-overlapping one?
INSERT INTO circles VALUES ('(200,200),2');
-- huh, it works
```

### System Columns - 5.5
```sql
-- let's see if we can see the system columns
SELECT tableoid FROM circles;
-- each row has the table id, so we get two values
SELECT xmin, cmin, xmax, cmax, ctid, c FROM circles;
-- most of these point to transaction IDs, xmin has the inserts
DELETE FROM circles WHERE c = '(0,0),100';
SELECT xmin, cmin, xmax, cmax, ctid, c FROM circles;
-- seems like we would need to see log or some kind of deleted
--  row list to see a non-0 in xmax/cmax
```

### Modifying Tables - 5.6
```sql
-- add a column
SELECT * FROM products LIMIT 1;
ALTER TABLE products ADD COLUMN description text;
SELECT * FROM products LIMIT 1;
-- remove a column
ALTER TABLE products DROP COLUMN description;
SELECT * FROM products LIMIT 1;
-- add with a constraint
ALTER TABLE products ADD COLUMN description text CHECK (description <> '');
SELECT * FROM products LIMIT 1;
-- the description is currently empty, but if we add another item, it
--  should enforce the not (<>) empty constraint on the new item
INSERT INTO products VALUES (2, 'potato', 0.75, '');
-- fails
INSERT INTO products VALUES (2, 'potato', 0.75, 'single russet potato');
SELECT * FROM products;
-- works
-- as seen before, if we have a foreign key dependency we may need to use
--  CASCADE to also drop the dependant columns. Not needed here but...
ALTER TABLE products DROP COLUMN description CASCADE;
-- doesn't hurt
SELECT * FROM products;
-- add a constrain
ALTER TABLE products ADD CHECK (name <> '');
-- with a name
ALTER TABLE products ADD CONSTRAINT some_name UNIQUE (product_no);
-- foreign key
CREATE TABLE product_groups (
    product_group_id    integer PRIMARY KEY,
    name                text
);
INSERT INTO product_groups VALUES(1, 'produce');
ALTER TABLE products ADD COLUMN product_group_id integer;
ALTER TABLE products ADD FOREIGN KEY (product_group_id)
    REFERENCES product_groups;
-- the table being referenced seems to need a primary key and needs to
--  exist in the table being modified, at least in this form
-- if you want a 'not null' constraint that can't be a table constraint
ALTER TABLE products ALTER COLUMN product_no SET NOT NULL;
\d products;

-- Removing a Constraint
-- you need to know the name to remove the constraint
-- I did not see the non null constraint we just added on its own, so
--  let's remove the some_name constraint from a while back, it's there
ALTER TABLE products DROP CONSTRAINT some_name;
\d products;
-- yep, it dropped
-- 'not null' has a special syntax
ALTER TABLE products ALTER COLUMN product_no DROP NOT NULL;
-- which give us an error because it is also a primary key
ALTER TABLE my_table ALTER COLUMN a SET NOT NULL;
\d my_table;
ALTER TABLE my_table ALTER COLUMN a DROP NOT NULL;
\d my_table;
-- that works
-- change a default
ALTER TABLE products ALTER COLUMN price SET DEFAULT 7.77;
\d products;
-- does not affect existing rows
-- remove the default
ALTER TABLE products ALTER COLUMN price DROP DEFAULT;
\d products;

-- changing a data type
ALTER TABLE products ALTER COLUMN price TYPE numeric(10,2);
\d products;
-- rename a table
ALTER TABLE products RENAME TO items;
\d items;
-- works
```

### Privileges - 5.7
We just assigned an account owner and have been connected as just one user, so we may have to go back and modify some things.
```sql
-- we are connected as dave, let's see if we can create a user joe
CREATE USER joe WITH PASSWORD '87654321';
-- nope, dave owns the database, but cannot create users
-- let's try as the postgres user
CREATE USER joe WITH PASSWORD '87654321';
-- now we should be able to grant joe access as dave, the owner
GRANT SELECT ON mytable TO joe;
-- try select as joe
SELECT * FROM mytable;
-- works, but there are no rows
\dp mytable
-- looks like joe can run the \dp command also
-- we could also give Column privileges
```

### Row Security - 5.8
```sql
-- back to user dave
DROP TABLE accounts;
CREATE TABLE accounts (
    manager         text,
    company         text,
    contact_email   text
);
ALTER TABLE accounts ENABLE ROW LEVEL SECURITY;
CREATE POLICY account_managers ON accounts TO managers
    USING (manager = current_user);
-- error: the role manager does not exist
CREATE ROLE managers;
-- looks like just being the table owner is not enough to create a role
-- worked as user postgres, then back to use dave
-- now we can create the policy as dave
CREATE POLICY account_managers ON accounts TO managers
    USING (manager = current_user);
-- put in some test rows
INSERT INTO accounts VALUES('dave', 'ACME', 'coyote@acme.com');
INSERT INTO accounts VALUES('joe', 'Cyberdyne Systems', 'miles@cyberdyne.com');
-- now switch to joe and see what happens
\q
-- psql -U joe mydb
SELECT * FROM accounts;
-- permission denied. Let's try just the row assigned to joe
SELECT * FROM accounts WHERE manager = 'joe';
SELECT current_user;
-- we must be missing the part where we give the role permissions
-- no, if we list the policies (as postgres)
SELECT * FROM pg_policies;
-- it seems like it should be OK (cmd = ALL)
SELECT * FROM pg_roles;
GRANT managers TO joe;
SELECT * FROM pg_auth_members;
-- joe's ID seems to have a line with the managers roleid
-- back as joe
SELECT * FROM accounts;
SELECT * FROM accounts WHERE manager = 'joe';
-- still not working
-- back to user postgres
SELECT a.rolname AS member, b.rolname AS role
    FROM pg_auth_members AS m
    JOIN pg_roles AS a ON a.oid = m.member
    JOIN pg_roles AS b ON b.oid = m.roleid
    WHERE b.rolname = 'managers';
-- back to assigning rights to managers?
GRANT SELECT ON accounts TO managers;
\c mydb joe -- connect as joe without a \q and rerunning psql
SELECT * FROM accounts;
-- finally works with joe only seeing his row
SELECT policyname, permissive, roles, cmd, qual
    FROM pg_policies
    WHERE tablename = 'accounts';
-- still don't understand why the select grant was needed
-- let's try another test from the manual which has been put in
--  the file rls_test.sql
\i rls_test.sql
-- then to test
set role admin;
table passwd;
-- admin can list the table
set role alice;
table passwd;
-- alice can't list the table
SELECT user_name, real_name, home_phone, extra_info, home_dir, shell
    FROM passwd;
-- alice can select items from the table, that were allowed to all users
UPDATE passwd SET user_name = 'joe';
-- alice can't set columns that were not granted access to the user listed
UPDATE passwd SET real_name = 'Alice Doe';
SELECT real_name FROM passwd;
-- alice can set the real_name
UPDATE passwd SET real_name = 'John Doe' WHERE user_name = 'admin';
-- but only for themselves
UPDATE passwd SET shell = '/bin/xx';
-- only certain shells are allowed
UPDATE passwd SET pwhash = 'abc';
-- for both this and the set of real_name, RLS silently filters to just
--  the row that alice can modify
-- I skipped trying the rest of this section, but the summary is that
--  security is difficult. Be careful
```
[this](https://www.enterprisedb.com/postgres-tutorials/how-implement-column-and-row-level-security-postgresql) was a good resource for more information about row and column security. Asking chatgpt worked well also

### Schemas
```sql
-- create a schema
CREATE SCHEMA myschema;
-- create a table in the schema
CREATE TABLE myschema.mytable (
    a   int,
    b   int
);
\d
-- does not show our new schema or table
DROP SCHEMA myschema;
-- because we made a table in the schema use cascade
DROP SCHEMA myschema CASCADE;
-- show the 'search path'
SHOW search_path;
SELECT search_path;
-- seems we need 'show' for search_path, but 'select' for something like
--  current_user
SET search_path TO myschema,public;
CREATE SCHEMA myschema;
CREATE TABLE myschema.mytable (
    a   int,
    b   int
);
\d
-- now shows our new schema and table at the top
SELECT * FROM mytable;
SELECT * FROM public.mytable;
SELECT * FROM myschema.mytable;
-- search path is working like it should
-- check to see what schemas/namespaces there are
SELECT nspname FROM pg_catalog.pg_namespace;
-- create a schema for someone else
-- need to do this as a superuser, so use postgres
\c mydb postgres
CREATE SCHEMA for_joe AUTHORIZATION joe;
-- or leave out the name for a user named schema
CREATE SCHEMA AUTHORIZATION joe;
-- look at the namespace
SELECT * FROM pg_catalog.pg_namespace;
-- the rest of the section just notes that schemas can create security
--  and portability issues. Be careful
```

### Inheritance
```sql
-- we did part of the example in the tutorial. Let's see what's left
\d
\d+ cities -- \d+ will list the dependant/child table
\d capitals
-- we are set, but will only do examples that were not in the tutorial
-- explicitly specify that descendant tables are included
SELECT name, elevation
    FROM cities*
    WHERE elevation > 500;
-- tableoid can tell you the originating table
SELECT c.tableoid, c.name, c.elevation
    FROM cities AS c
    WHERE c.elevation > 500;
-- we will need a join with pg_class to get the table names instead of
--  the ids
SELECT p.relname, c.name, c.elevation
    FROM cities AS c, pg_class AS p
    WHERE c.elevation > 500 AND c.tableoid = p.oid;
-- this uses some implicit joins. Let's try to break those out explicitly
SELECT p.relname, c.name, c.elevation
    FROM cities AS c
    INNER JOIN pg_class AS p ON c.tableoid = p.oid
    WHERE c.elevation > 500;
-- same result, but more clear what is happening
-- we can also use the regclass alias type
SELECT c.tableoid::regclass, c.name, c.elevation
    FROM cities AS c
    WHERE c.elevation > 500;
-- INSERT always acts on the exact table specified, so this will fail
INSERT INTO cities (name, population, elevation, state)
    VALUES ('Albany', NULL, NULL, 'NY');
```

### Table Partitioning - 5.11
```sql
CREATE TABLE measurement (
    city_id     int NOT NULL,
    logdate     date NOT NULL,
    peaktemp    int,
    unitsales   int
);
-- Now say we would like to partition this table between recent and old
DROP TABLE measurement;
CREATE TABLE measurement (
    city_id     int NOT NULL,
    logdate     date NOT NULL,
    peaktemp    int,
    unitsales   int
) PARTITION BY RANGE (logdate);
-- then make monthly partitions (we will just do two)
CREATE TABLE measurement_y2006m02 PARTITION OF measurement
    FOR VALUES FROM ('2006-02-01') TO ('2006-03-01');
CREATE TABLE measurement_y2006m03 PARTITION OF measurement
    FOR VALUES FROM ('2006-03-01') TO ('2006-04-01');
-- then make some special partitions for recent data
CREATE TABLE measurement_y2006m04 PARTITION OF measurement
    FOR VALUES FROM ('2006-04-01') TO ('2006-05-01')
    TABLESPACE fasttablespace;
-- this gives an error because we never setup this tablespace
-- let's try to set it up, we will need to go back user postgres
CREATE TABLESPACE fasttablespace OWNER dave
    LOCATION 'F:\postgres_store';
SELECT * FROM pg_tablespace; -- or \db or \db+
-- seems to have worked, back to user dave
-- now the table creation with the new tablespace is working
-- last one specifies a number of workers
CREATE TABLE measurement_y2006m05 PARTITION OF measurement
    FOR VALUES FROM ('2006-05-01') TO ('2006-06-01')
    WITH (parallel_workers = 4)
    TABLESPACE fasttablespace;
\db+
-- I guess we can't do \db+ as a non superuser
SELECT * FROM pg_tablespace;
-- create an index on the thing we are partitioning on
CREATE INDEX ON measurement (logdate);
-- checking postgresql.conf, enable_partition_pruning is 'on'
-- we would have to make new partitions each month. They should probably
--  be scripted and scheduled
-- old months of data could be copied to cheap storage and deleted
-- delete an old month
DROP TABLE measurement_y2006m02;
-- I will stop here, but there is much more about implementing this
--  well, with best practices in the documentation
```
We already did a good bit of the rest of section 5, so I will skip it
