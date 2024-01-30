# The Official Postgres Tutorial in Sections 1-3
[here](https://www.postgresql.org/docs/current/tutorial-start.html)

## First, install on Windows
The official downloads are [here](https://www.enterprisedb.com/downloads/postgres-postgresql-downloads) and I pretty much went with the defaults. After this and skipping the installation of any additions I see there are a number of "PostgreSQL Server" processes running and a "pg_ctl", so I assume everything is ready.

## Create a Database
Try to create a database with (using the terminal in VS Code for the Windows command line parts of this)
```
createdb mydb
```
Which fails with the Windows equivalent of command not found. To fix this, add to your account path with:
Start -> Settings -> Search "path" -> Edit environment variables for your account -> Path -> Edit... button -> New button -> "C:\Program Files\PostgreSQL\16\bin (or wherever it is for you) -> OK -> OK
Then restart VS Code (or wherever you are entering the commands)
Now the password fails because it is trying to run as the current user instead of a sysadmin. One way around this is to use the password you provided at setup with
```
createdb -U postgres mydb
```
The -U is to use the default superuser account. This will get the database setup and then you can add your user that you will be running as (dave in this case)
```
createuser -U postgres dave
```
Then we need an SQL prompt to set the password and change the DB owner
```
psql -U postgres
```
Then we can change the password and owner
```sql
ALTER USER dave WITH PASSWORD '12345678';
ALTER DATABASE mydb OWNER dave;
\q
```
Now we can connect with our default user (dave) with the password we set (12345678)
```
psql mybd
```

### Doing some basic SQL stuff - 1.4
This will be some of the official PostgreSQL tutorial. It starts [here](https://www.postgresql.org/docs/current/tutorial-accessdb.html)
```sql
SELECT version();
SELECT current_date;
SELECT 2 + 2;
```
Special backslash commands
```sql
\h
\h SELECT
\p
\echo 'Hello World'
\d
\l
\conninfo
\getenv PSQLVAR ENVVAR
```
### The SQL Language - 2.3
```sql
CREATE TABLE weather (
    city        varchar(80),
    temp_lo     int,            -- low temperature
    temp_hi     int,            -- high temperature
    prcp        real,           -- precipitation
    date        date
);
```
This gives a 'ERROR:  permission denied for schema public'. We need to exit our user session with \q and log in as the postgres user and give the user some more privileges with the following (after connecting with 'psql -U postgres mydb'):
```sql
GRANT ALL PRIVILEGES ON DATABASE mydb TO dave;
GRANT ALL ON SCHEMA public TO dave;
```
This should work, but undoes some permissions so you may want to create a schema with specific authorization. Back to a connection as user 'dave'
```sql
CREATE TABLE weather (
    city        varchar(80),
    temp_lo     int,            -- low temperature
    temp_hi     int,            -- high temperature
    prcp        real,           -- precipitation
    date        date
);
CREATE TABLE cities (
    name        varchar(80),
    location    point
);
\d -- to see our new tables

INSERT INTO weather VALUES ('San Francisco', 46, 50, 0.25, '1994-11-27');
INSERT INTO cities VALUES ('San Francisco', '(-194.0, 53.0)');
-- another way to insert
INSERT INTO weather (city, temp_lo, temp_hi, prcp, date)
    VALUES ('San Francisco', 43, 57, 0.0, '1994-11-29');
-- this allows a different order or omission of a variable
INSERT INTO weather (date, city, temp_hi, temp_lo)
    VALUES ('1994-11-29', 'Hayward', 54, 37);
-- the COPY command could also be used to read in rows from a file
-- example COPY weather FROM 'C:/Path/to/your/data.txt';

SELECT * FROM weather;
-- or
SELECT city, temp_lo, temp_hi, prcp, date FROM weather;
SELECT city, (temp_hi+temp_lo)/2 AS temp_avg, date FROM weather;
SELECT * FROM weather
    WHERE city = 'San Francisco' AND prcp > 0.0;
SELECT * FROM weather
    ORDER BY city;
-- and with multiple sort orders
SELECT * FROM weather
    ORDER BY city, temp_lo;
SELECT DISTINCT city
    FROM weather;
-- and ordered
SELECT DISTINCT city
    FROM weather
    ORDER BY city;
```
### Joins - 2.6
```sql
SELECT * FROM weather JOIN cities ON city = name;
-- without the duplicate city name
SELECT city, temp_lo, temp_hi, prcp, date, location
    FROM weather JOIN cities ON city = name;
-- fully qualified
SELECT weather.city, weather.temp_lo, weather.temp_hi,
       weather.prcp, weather.date, cities.location
    FROM weather JOIN cities ON weather.city = cities.name;
-- short form
SELECT *
    FROM weather, cities
    WHERE city = name;
-- outer join
SELECT *
    FROM weather LEFT OUTER JOIN cities on weather.city = cities.name;
-- right outer join
SELECT *
    FROM weather RIGHT OUTER JOIN cities on weather.city = cities.name;
-- full outer join
SELECT *
    FROM weather FULL OUTER JOIN cities on weather.city = cities.name;
-- self join
SELECT w1.city, w1.temp_lo AS low, w1.temp_hi AS high,
       w2.city, w2.temp_lo AS low, w2.temp_hi AS high
    FROM weather AS w1 JOIN weather AS w2
        ON w1.temp_lo < w2.temp_lo AND w1.temp_hi > w2.temp_hi;
-- to save typing without AS
SELECT *
    FROM weather w JOIN cities c on w.city = c.name;
```
### Aggregate Functions - 2.7
```sql
SELECT max(temp_lo) FROM weather;
-- to use with a WHERE filter to get the city of occurrence
-- have to use a select from another select
SELECT city FROM weather
    WHERE temp_lo = (SELECT max(temp_lo) FROM weather);
-- counting up items and grouping
SELECT city, count(*), max(temp_lo)
    FROM weather
    GROUP BY city;
-- somehow, counting up 1s makes more sense to me
SELECT city, count(1), max(temp_lo)
    FROM weather
    GROUP BY city;
-- a post filter using HAVING, which is new to me
SELECT city, count(1), max(temp_lo)
    FROM weather
    GROUP BY city
    HAVING max(temp_lo) < 40;
-- HAVING can contain aggregate functions, but WHERE cannot
-- filtering for cities that start with S, we can use where
-- because the aggregate is done outside/after the where filter
SELECT city, count(1), max(temp_lo)
    FROM weather
    WHERE city LIKE 'S%'
    GROUP BY city;
-- another filtering method is to use FILTER, per aggregate
SELECT city, count(1) FILTER (WHERE temp_lo < 45), max(temp_lo)
    FROM weather
    GROUP BY city;
-- FILTER only effects the item it's applied to in SELECT
-- which is why the 46 degree SF entry still makes it through
-- could use a better column name than 'count' in the example

-- updating rows 2.7
UPDATE weather
    SET temp_hi = temp_hi - 2, temp_lo = temp_lo - 2
    WHERE date > '1994-11-28';
-- look at the result
SELECT * FROM weather;
-- deleting rows
DELETE FROM weather WHERE city = 'Hayward';
SELECT * FROM weather;
-- DELETE FROM tablename; would remove all the rows in tablename
```

## Advanced Features - 3
Chapter 3 in the turorial [here](https://www.postgresql.org/docs/current/tutorial-views.html)
```sql
-- Views
-- a view gives a table-like name to a query result
CREATE VIEW myview AS
    SELECT name, temp_lo, temp_hi, prcp, date, location
    FROM weather, cities
    WHERE city = name;
SELECT * FROM myview;

-- Foreign Keys
-- to only allow inserts of listed cities
-- define the tables like this
CREATE TABLE cities (
    name        varchar(80) primary key,
    location    point
);
-- this errors as we already have the table, so we need to drop the
--  table first
DROP TABLE cities;
-- since we just created a view using cities, we need to them both
DROP TABLE cities CASCADE;
-- what is left
\d
-- just weather, but we will need to redo that also, so...
DROP TABLE weather;
-- now remake both tables
CREATE TABLE cities (
    name        varchar(80) primary key,
    location    point
);
CREATE TABLE weather (
    city        varchar(80) references cities(name),
    temp_lo     int,
    temp_hi     int,
    prcp        real,
    date        date
);
\d
-- they are back
INSERT INTO weather VALUES ('Berkeley', 45, 53, 0.0, '1994-11-28');
-- will not insert, as designed, since Berkeley is not in the cities table

-- Transactions
-- the tutorial does not give us any tables to run the example code against
--  so let's try to create some ourselves
CREATE TABLE branches (
    name        varchar(80) primary key,
    balance     real
);
CREATE TABLE accounts (
    name        varchar(80),
    balance     real,
    branch_name varchar(80) references branches(name)
);
-- we used the relationship method we just learned about with references
-- then we need to put some entries in these to match the example code
INSERT INTO branches VALUES ('A branch', 200.0);
INSERT INTO branches VALUES ('B branch', 0.0);
INSERT INTO accounts VALUES ('Alice', 200.00, 'A branch');
INSERT INTO accounts VALUES ('Bob', 0.00, 'B branch');
INSERT INTO accounts VALUES ('Wally', 0.00, 'B branch');
-- use \d and/or SELECT * to be sure things are OK
-- now we should be able to try a transaction
BEGIN;
UPDATE accounts SET balance = balance - 100.00
    WHERE name = 'Alice';
UPDATE branches SET balance = balance - 100.00
    WHERE name = (SELECT branch_name FROM accounts WHERE name = 'Alice');
UPDATE accounts SET balance = balance + 100.00
    WHERE name = 'Bob';
UPDATE branches SET balance = balance + 100.00
    WHERE name = (SELECT branch_name FROM accounts WHERE name = 'Bob');
COMMIT;
SELECT * FROM accounts;
-- seems to have worked
-- now let's try the SAVEPOINT
BEGIN;
UPDATE accounts SET balance = balance - 100.00
    WHERE name = 'Alice';
UPDATE branches SET balance = balance - 100.00
    WHERE name = (SELECT branch_name FROM accounts WHERE name = 'Alice');
SAVEPOINT my_savepoint;
UPDATE accounts SET balance = balance + 100.00
    WHERE name = 'Bob';
-- oops ... should go to Wally
ROLLBACK TO my_savepoint;
UPDATE accounts SET balance = balance + 100.00
    WHERE name = 'Wally';
UPDATE branches SET balance = balance + 100.00
    WHERE name = (SELECT branch_name FROM accounts WHERE name = 'Wally');
COMMIT;
SELECT * FROM accounts;
SELECT * FROM branches;
-- everything seems to have gone as it should

-- Window Functions
-- again, we are going to have to do some setup to try the examples and
--  this time I looked for included tutorial files, and could not find any
-- I think we just need a single table
CREATE TABLE empsalary (
    empno       int,
    salary      int,
    depname     varchar(80)
);
INSERT INTO empsalary VALUES (11, 5200, 'develop');
INSERT INTO empsalary VALUES (7, 4200, 'develop');
INSERT INTO empsalary VALUES (9, 4500, 'develop');
INSERT INTO empsalary VALUES (8, 6000, 'develop');
INSERT INTO empsalary VALUES (10, 5200, 'develop');
INSERT INTO empsalary VALUES (5, 3500, 'personnel');
INSERT INTO empsalary VALUES (2, 3900, 'personnel');
INSERT INTO empsalary VALUES (3, 4800, 'sales');
INSERT INTO empsalary VALUES (1, 5000, 'sales');
INSERT INTO empsalary VALUES (4, 4800, 'sales');
SELECT * FROM empsalary;
-- looks good, let's try the window function
SELECT depname, empno, salary, avg(salary) OVER (PARTITION BY depname)
    FROM empsalary;
-- works and we see the OVER makes it a window function
--  with the 'frame' set by the PARTITION BY
-- and with ordering
SELECT depname, empno, salary,
    rank() OVER (PARTITION BY depname ORDER BY salary DESC)
    FROM empsalary;
-- what rank() does with ties is interesting
-- another example without the partition
SELECT salary, sum(salary) OVER () FROM empsalary;
-- and ordered
SELECT salary, sum(salary) OVER (ORDER BY salary) FROM empsalary;
-- now the sum is cumulative with grouped behavior when salaries are
--  the same
-- I didn't include an 'enroll_date' when creating the table, so let's
--  see if the next example works without it as it looks like it should
SELECT depname, empno, salary
    FROM (SELECT depname, empno, salary,
        rank() OVER (PARTITION BY depname ORDER BY salary DESC, empno) AS pos
        FROM empsalary
    ) AS ss
    WHERE pos < 3;
-- works and shows the highest two paid employees in each department
-- naming a window
SELECT sum(salary) OVER w, avg(salary) OVER w
    FROM empsalary
    WINDOW w AS (PARTITION BY depname ORDER BY salary DESC);
-- I see what it is doing, but it's an odd example
-- also strange, the name seems to come before the AS, weird

-- Inheritance
-- first drop the existing cities table that we have from earlier and
--  we will have to cascade it to get rid of weather that depends on it
DROP TABLE cities CASCADE;
\d
\d weather
-- the cities table is gone as is the dependency from the weather table
-- now we can try the inheritance example
CREATE TABLE cities (
    name        text,
    population  real,
    elevation   int     -- (in ft)
);
CREATE TABLE capitals (
    state       char(2) UNIQUE NOT NULL
) INHERITS (cities);
\d capitals
-- seems to have worked. We will need some cities to do the rest of the
--  examples, just guessing populations
INSERT INTO cities VALUES ('Mariposa', 2000.0, 1953);
INSERT INTO capitals VALUES ('Madison', 150000.0, 845, 'WI');
INSERT INTO cities VALUES ('Las Vegas', 350000.0, 2174);
INSERT INTO cities VALUES ('San Francisco', 400000, 50);
-- now we can try the examples
SELECT name, elevation
    FROM cities
    WHERE elevation > 500;
SELECT name, elevation
    FROM ONLY cities
    WHERE elevation > 500;
```
