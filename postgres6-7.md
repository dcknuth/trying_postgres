# Data Manipulation and Queries - Chapters 6 and 7

## Data Manipulation - Chapter 6
### Inserting Data - 6.1
```sql
-- we already did some of this, let's see where we were
SELECT * FROM products;
\d
-- looks like we deleted the table from earlier, so we will redo it
CREATE TABLE products (
    product_no  integer,
    name        text,
    price       numeric
);
INSERT INTO products VALUES (1, 'Cheese', 9.99);
-- if you need to change the order, skip values or conform to a standard
--  list the column names first
INSERT INTO products (name, price, product_no) VALUES ('Carrot', 1.25, 2);
-- PostgreSQL will fill in defaults for missing values on the right, but
--  this is not SQL standard
INSERT INTO products VALUES (3, 'Potatoes');
SELECT * FROM products;
-- specifically request a default value
INSERT INTO products VALUES (4, 'Spinach', DEFAULT);
SELECT * FROM products WHERE product_no = 4;
-- set a default so we can see the row in the next example
ALTER TABLE products
    ALTER COLUMN  price SET DEFAULT 999.99;
-- request to set a new row to all defaults
INSERT INTO products DEFAULT VALUES;
SELECT * FROM products;
-- insert multiple rows
INSERT INTO products (product_no, name, price) VALUES
    (5, 'Bread', 2.75),
    (6, 'Milk', 4.00),
    (7, 'THC gummies', 27.99);
SELECT * from products;
-- insert from a selection
CREATE TABLE new_products (
    product_no      integer,
    name            text,
    price           numeric,
    release_date    date
);
INSERT INTO new_products VALUES
    (8, 'Leg warmers', 4.99, '1982-01-01'),
    (9, 'Ponzi crypto coin', 9999.99, 'today');
SELECT * FROM new_products;
INSERT INTO products (product_no, name, price)
    SELECT product_no, name, price FROM new_products
    WHERE release_date = 'today';
SELECT * from products;
-- note that COPY might be more efficient if it can be used
```

### Updating Data - 6.2
```sql
-- note, you can only reliably address a single row with a primary key
-- update all product prices with a null to the default
UPDATE products SET price = DEFAULT WHERE price IS NULL;
-- note that the docs here used an actual price and '='. One has to
--  use 'IS' to compare with null
SELECT * FROM products;
-- we need to adjust all prices for the post-covid inflation
UPDATE products SET price = price * 1.10;
SELECT * FROM products;
-- fractional cents will confuse people
UPDATE products SET price = ROUND(price, 2);
SELECT * FROM products;
-- update multiple columns, my_table seems to have some stuff
SELECT * FROM my_table;
UPDATE my_table SET a = 5, my_data = 'stuff' WHERE a > 0;
SELECT * FROM my_table;
```

### Deleting Data - 6.3
```sql
SELECT * FROM products;
-- we never should have stocked crypto
DELETE FROM products WHERE name = 'Ponzi crypto coin';
SELECT * FROM products;
-- this will delete all rows, beware!
DELETE FROM my_table;
SELECT * FROM my_table;
-- guess we can drop that one now (which would have been faster overall)
DROP TABLE my_table;
```

### Returning Data from Modified Rows - 6.4
```sql
-- returning can be useful with insert with a computed default, like serial
DROP TABLE IF EXISTS users CASCADE;
CREATE TABLE users (
    firstname   text,
    lastname    text,
    id          SERIAL PRIMARY KEY
);
INSERT INTO users (firstname, lastname) VALUES ('Joe', 'Cool') RETURNING id;
-- increased handling cost of low priced items, let's see the new price
UPDATE products SET price = ROUND(price * 1.1, 2)
    WHERE price <= 10.0
    RETURNING name, price AS new_price;
-- delete and see what was deleted
SELECT * FROM new_products;
DELETE FROM new_products
    WHERE release_date = 'today'
    RETURNING *;
-- returning is also useful for seeing columns computed by triggers
```

## Queries - Chapter 7
### Table Expressions - 7.2
```sql
-- let's look at the default if you list multiple tables
-- two unrelated tables, no join type specified
\d products
\d users
SELECT * FROM users;
SELECT * FROM products, users;
-- some equivalents
SELECT * FROM products CROSS JOIN users;
SELECT * FROM products INNER JOIN users ON TRUE;
-- what happens if there is more than one row in users?
INSERT INTO users VALUES ('John', 'Doe');
SELECT * FROM products, users;
-- so every row in the first table gets a line for every row in the
--  second table with all of its columns
-- we need some test tables, will use the ones in the docs
CREATE TABLE t1 (
    num     integer,
    name    text
);
CREATE TABLE t2 (
    num     integer,
    value   text
);
INSERT INTO t1 VALUES
    (1, 'a'),
    (2, 'b'),
    (3, 'c');
INSERT INTO t2 VALUES
    (1, 'xxx'),
    (3, 'yyy'),
    (5, 'zzz');
-- a default or CROSS JOIN will be like above and should give 9 rows
SELECT * FROM t1, t2;
-- basically flattens multiple tables into one rows = t1 rows * t2 rows
-- inner join on num, should give only matching 2 rows, all columns
SELECT * FROM t1 INNER JOIN t2 ON t1.num = t2.num;
-- we have to use the fully qualified name at the end (like t1.num) since
--  both tables are using the same column name. Or, we can use
SELECT * FROM t1 INNER JOIN t2 USING (num);
-- USING assumes they will have the same name. Or, we can use
SELECT * FROM t1 NATURAL INNER JOIN t2;
-- NATURAL will assume compatible, shared column names
-- left join should list all rows from the first (left) table filling
--  in any nulls needed for missing rows in the second (right) table
SELECT * FROM t1 LEFT JOIN t2 ON t1.num = t2.num;
SELECT * FROM t1 LEFT JOIN t2 USING (num);
SELECT * FROM t1 NATURAL LEFT JOIN t2;
-- all work and fill in the missing value with null
-- right join will do the same for the second (right) table
SELECT * FROM t1 RIGHT JOIN t2 ON t1.num = t2.num;
-- the other forms still work, but will only list the right num column
SELECT * FROM t1 RIGHT JOIN t2 USING (num);
SELECT * FROM t1 NATURAL RIGHT JOIN t2;
-- full join
SELECT * FROM t1 FULL JOIN t2 ON t1.num = t2.num;
-- lists all rows from both tables and fills in nulls where needed
SELECT * FROM t1 FULL JOIN t2 USING (num);
SELECT * FROM t1 NATURAL FULL JOIN t2;
-- again, the other forms work, but only use 1 num column
-- ON can contain additional conditions, but with pre filter results
SELECT * FROM t1 LEFT JOIN t2 ON t1.num = t2.num AND t2.value = 'xxx';
-- notice this is very different than using WHERE to filter
SELECT * FROM t1 LEFT JOIN t2 ON t1.num = t2.num
    WHERE t2.value = 'xxx';

-- Table and Column Aliases
-- column aliases can be used to reduce typing or confusion
SELECT t1.num AS t1_num, name, t2.num AS t2_num, value
    FROM t1 FULL JOIN t2 ON t1.num = t2.num;
-- now it is easier to tell which num we are looking at
-- I like to use AS, but it is optional
-- above, we use the old name during the table creation process
-- we need to use the alias name for a table after it has been aliased
SELECT * FROM t1 AS t3
    WHERE t3.num = 1;
-- table aliases come in handy for sub selections and are needed for
--  self joins
CREATE TABLE people (
    id          integer,
    name        text,
    mother_id   integer
);
INSERT INTO people VALUES
    (1, 'Jan', NULL),
    (2, 'Jill', 1),
    (3, 'Joe', 2),
    (4, 'Bob', 1);
SELECT * FROM people AS mother
    JOIN people AS child ON mother.id = child.mother_id;
SELECT * FROM people AS mother
    INNER JOIN people AS child ON mother.id = child.mother_id;
-- JOIN without anything else is an INNER JOIN, but just a list defaults
--  to a cartesian join
-- you can alias a table and columns at the same time
SELECT * FROM people AS mother (my_id, mother_name, my_mother)
    JOIN people AS child (child_id, child_name, childs_mother)
    ON mother.my_id = child.childs_mother;

-- Subqueries
-- subquery as a values list
SELECT * FROM (VALUES
    ('anne', 'smith'),
    ('bob', 'jones'),
    ('joe', 'blow')) AS names(first, last);
-- aggregation of another select
SELECT mother_name, count(mother_name) AS num_children FROM
    (SELECT * FROM people AS mother (my_id, mother_name, my_mother)
        JOIN people AS child (child_id, child_name, childs_mother)
        ON mother.my_id = child.childs_mother)
    GROUP BY mother_name;
-- with a table alias also
SELECT moms.mother_name, count(moms.mother_name) AS num_children FROM
    (SELECT * FROM people AS mother (my_id, mother_name, my_mother)
        JOIN people AS child (child_id, child_name, childs_mother)
        ON mother.my_id = child.childs_mother) AS moms
    GROUP BY moms.mother_name;

-- Table Functions
CREATE TABLE foo (
    fooid       int,
    foosubid    int,
    fooname     text
);
CREATE FUNCTION getfoo(int) RETURNS SETOF foo AS $$
    SELECT * FROM foo WHERE fooid = $1;
$$ LANGUAGE SQL;
INSERT INTO foo VALUES
    (1, 4, 'name1'),
    (2, 5, 'name2'),
    (3, 6, 'name3');
SELECT * FROM getfoo(1) AS t1;
SELECT * FROM foo
    WHERE foosubid IN (
        SELECT foosubid
        FROM getfoo(foo.fooid) z
        WHERE z.fooid = foo.fooid
    );
-- not very exciting
CREATE VIEW vw_getfoo AS SELECT * FROM getfoo(1);
SELECT * FROM vw_getfoo;
-- ok, but still not exciting

SELECT *
    FROM dblink('dbname=mydb', 'SELECT proname, prosrc FROM pg_proc')
        AS t1(proname name, prosrc text)
    WHERE proname LIKE 'bytea%';
-- I guess the dblink module is not installed or we need a superuser
-- well, a superuser alone is not enough, next example should work
SELECT *
    FROM ROWS FROM (
        json_to_recordset('[{"a":40,"b":"foo"},{"a":"100","b":"bar"}]')
            AS (a INTEGER, b TEXT),
        generate_series(1, 3)
    ) AS x (p, q, s)
ORDER BY p;
-- and one more ROWS FROM example so I can get how it works
SELECT * FROM ROWS FROM (
    generate_series(1, 3), -- Generates numbers from 1 to 3
    generate_series(101, 103) -- Generates numbers from 101 to 103 in parallel
) AS t(col1, col2);
-- K, I think I get it

-- Lateral Subqueries
-- didn't like the doc examples or a ChatGPT one, so will just use the
--  trivial one from the docs
DROP TABLE foo CASCADE;
CREATE TABLE foo (
    id      int,
    bar_id  int
);
INSERT INTO foo VALUES
    (1, 11),
    (2, 22),
    (3, 33),
    (4, 44);
CREATE TABLE bar (
    id      int,
    name    text
);
INSERT INTO bar VALUES
    (22, 'b1'),
    (44, 'b2'),
    (15, 'b3');
SELECT * FROM foo, LATERAL
    (SELECT * FROM bar WHERE bar.id = foo.bar_id) ss;
-- the docs note this case is easier with the default, cartesian join
SELECT * FROM foo, bar WHERE bar.id = foo.bar_id;

-- WHERE
-- we will use the products table we already have
SELECT * FROM products WHERE price > 5;
SELECT * FROM products WHERE name LIKE '%e%';
SELECT * FROM products WHERE price BETWEEN 10 AND 50;
SELECT * FROM products WHERE name IS NOT NULL;
SELECT * FROM products WHERE name = 'Milk';
SELECT * FROM products WHERE product_no IN (1, 2, 3);
-- the docs don't provide sample tables, so these are simplified and don't
--  have sub-selects

-- GROUP BY
CREATE TABLE test1 (
    x   text,
    y   int
);
INSERT INTO test1 VALUES
    ('a', 3),
    ('c', 2),
    ('b', 5),
    ('a', 1);
SELECT x FROM test1 GROUP BY x;
-- with aggregate expression
SELECT x, sum(y) FROM test1 GROUP BY x;
SELECT x, array_agg(y) FROM test1 GROUP BY x;
-- a variation of the mom table from before
SELECT moms.mother_name, array_agg(child_name) AS children FROM
    (SELECT * FROM people AS mother (my_id, mother_name, my_mother)
        JOIN people AS child (child_id, child_name, childs_mother)
        ON mother.my_id = child.childs_mother) AS moms
    GROUP BY moms.mother_name;

-- HAVING
-- mostly used like a WHERE filter after a GROUP BY
SELECT x, sum(y) FROM test1 GROUP BY x HAVING sum(y) > 3;
SELECT x, sum(y) FROM test1 GROUP BY x HAVING x < 'c';
SELECT moms.mother_name, array_agg(child_name) AS children FROM
    (SELECT * FROM people AS mother (my_id, mother_name, my_mother)
        JOIN people AS child (child_id, child_name, childs_mother)
        ON mother.my_id = child.childs_mother) AS moms
    GROUP BY moms.mother_name HAVING count(moms.mother_name) > 1;

-- GROUPING SETS, CUBE, and ROLLUP
CREATE TABLE items_sold (
    brand   text,
    size    text,
    sales   integer
);
INSERT INTO items_sold VALUES
    ('Foo', 'L', 10),
    ('Foo', 'M', 20),
    ('Bar', 'M', 15),
    ('Bar', 'L', 5);
SELECT brand, size, sum(sales) FROM items_sold
    GROUP BY GROUPING SETS ((brand), (size), ());
-- GROUPING SETS gets you sub-set grouping. The last, empty () gets you
--  the sum over everything without a sub-group
SELECT brand, size, sum(sales) FROM items_sold
    GROUP BY ROLLUP (brand, size);
-- is equivalent to GROUPING SETS ((brand, size), (brand), ())
SELECT brand, size, sum(sales) FROM items_sold
    GROUP BY CUBE (brand, size);
-- gives all sub-sets, including the empty (total)
```

### Select Lists - 7.3
```sql
-- we will skip some of the common usage we have already covered
CREATE TABLE test2 (
    a   integer,
    b   integer,
    c   integer
);
INSERT INTO test2 VALUES
    (1, 1, 1),
    (1, 2, 3),
    (1, 4, 5);
SELECT a, b + c AS sum FROM test2;
-- don't do it, but in case you see it, a keyword can be named as a column
--  label using AS or with quotes
SELECT a AS from, b + c AS sum FROM test2;
-- from and sum are a keyword and a function

-- DISTINCT
-- removes duplicates
SELECT * FROM people;
SELECT DISTINCT mother_id FROM people;
-- if we had multiple nulls only one would show
```

### Combining Queries (UNION, INTERSECT, EXCEPT)
```sql
CREATE TABLE test3 (
    a   integer,
    b   integer,
    c   integer
);
INSERT INTO test3 VALUES
    (1, 1, 1),
    (1, 2, 3),
    (1, 6, 7);
SELECT * FROM test2 UNION
    SELECT * FROM test3;
-- all rows, no duplicates
SELECT * FROM test2 UNION ALL
    SELECT * FROM test3;
-- all rows, with duplicates
SELECT * FROM test2 INTERSECT
    SELECT * FROM test3;
-- only rows in common
SELECT * FROM test2 INTERSECT ALL
    SELECT * FROM test3;
INSERT INTO test2 VALUES (1, 1, 1);
SELECT * FROM test2 INTERSECT ALL
    SELECT * FROM test3;
INSERT INTO test3 VALUES (1, 1, 1);
SELECT * FROM test2 INTERSECT ALL
    SELECT * FROM test3;
-- ALL will include duplicates, but only if they are duplicated in
--  both queries for INTERSECT
SELECT * FROM test2 EXCEPT
    SELECT * FROM test3;
INSERT INTO test2 VALUES (1, 4, 5);
SELECT * FROM test2 EXCEPT ALL
    SELECT * FROM test3;
-- include dups
-- same table
SELECT * FROM test2 EXCEPT
    SELECT * FROM test2 WHERE c = 1;
```

### Sorting Rows (ORDER BY)
```sql
SELECT * FROM test1 ORDER BY x;
SELECT * FROM test1 ORDER BY 1;
SELECT * FROM test1 ORDER BY x, y;
SELECT * FROM test1 ORDER BY x DESC, y;
SELECT * FROM test1 ORDER BY x DESC, y DESC;
SELECT * FROM people ORDER BY mother_id;
SELECT * FROM people ORDER BY mother_id NULLS FIRST;
```

### LIMIT and OFFSET
```sql
-- limit is nice to just get a couple rows to see what a table looks like
SELECT * FROM test1 LIMIT 1;
SELECT * FROM people LIMIT 2;
-- if you need a beginning or end few, you need to combine with ORDER BY
SELECT * FROM test1 ORDER BY x LIMIT 2;
-- using OFFSET probably only makes sense with ORDER BY
-- skip the null mother_id
SELECT * FROM people ORDER BY mother_id NULLS FIRST OFFSET 1;
-- what if we don't know the number of nulls?
SELECT * FROM people ORDER BY mother_id NULLS FIRST
    OFFSET (SELECT count(1) FROM people WHERE mother_id IS NULL);
```

### VALUES Lists
```sql
VALUES (1, 'one'), (2, 'two'), (3, 'three');
-- different versions of SQL may assign different default column names
-- you can override the defaults like this
SELECT * FROM
    (VALUES (1, 'one'), (2, 'two'), (3, 'three')) AS t (num, letter);
-- can't rename without the select
```

### WITH Queries (Common Table Expressions)
```sql
-- WITH allows the breakdown and reorder of complex queries
CREATE TABLE departments (
    id SERIAL PRIMARY KEY,
    department_name TEXT
);
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name TEXT,
    department_id INTEGER REFERENCES departments(id)
);
INSERT INTO departments (department_name) VALUES
    ('HR'),
    ('Development'),
    ('Marketing');
INSERT INTO employees (name, department_id) VALUES
    ('Alice', 1),
    ('Bob', 2),
    ('Charlie', 3);
-- now our with query
WITH d AS (
    SELECT id, department_name
    FROM departments
)
SELECT e.name AS employee_name, d.department_name
    FROM employees AS e
    JOIN d ON e.department_id = d.id;
-- this one is pretty simple, but I do like moving a sub-select up into
--  a with statement with a name as it can be tested and named before
--  using it

-- Recursive Queries
-- sum 1 to 100
WITH RECURSIVE t(n) AS (
    VALUES (1)
    UNION ALL
        SELECT n+1 FROM t WHERE n < 100
)
SELECT sum(n) FROM t;
-- the form is: non-recursive term, UNION[ALL], recursive term
-- following sub-tables as far as they go is a common use

-- Search Order
-- going to skip examples here, but it covers how to final order results
--  by breath or depth. Results are not directly returned in any
--  pre-defined order

-- Cycle Detection
-- one can add columns to the recursive statement and look for a loop
--  with a WHERE test... or just set a limit
WITH RECURSIVE t(n) AS (
    SELECT 1
    UNION ALL
        SELECT n+1 FROM t
)
SELECT n FROM t LIMIT 100;

-- the MATERIALIZED section seems rare, so going to skip

-- data modifying statement are mostly allowed within WITH, however it
--  seems like separate statements within a transaction block might be
--  better for most uses
```
