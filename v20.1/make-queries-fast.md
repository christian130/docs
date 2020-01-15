---
title: Make Queries Fast
summary: How to make your queries run faster during application development
toc: true
---

This page describes how to optimize OLTP application queries for CockroachDB.  To get good performance from CockroachDB, you need to look at how you're using the database through several lenses:

- [SQL Query performance](#sql-query-performance)
- [Schema Design](#schema-design)
- [Cluster Topology](#cluster-topology)

## SQL query performance

To get good SQL query performance, follow these rules (in rough order of importance):

1. Make sure your query [scans at most a few dozen rows](#scan-at-most-a-few-dozen-rows) (several hundred rows at the absolute maximum).
2. [Use the right index](#use-the-right-index).
3. If needed, check that your query is [using the right join type](#use-the-right-join-type) for the table sizes you are querying.  (This should rarely be necessary.)

For example, let's optimize a query against the [Employees data set](https://github.com/datacharmer/test_db).  To import the data set, run:

{% include copy-clipboard.html %}
~~~ sql
CREATE DATABASE IF NOT EXISTS employees;
USE employees;
IMPORT MYSQLDUMP 'https://s3-us-west-1.amazonaws.com/cockroachdb-movr/datasets/employees-db/mysqldump/employees-full.sql.gz';
~~~

The question we want to answer about this data is: "Who are the top 25 highest-paid employees?"

### Scan at most a few dozen rows

#### Study the schema

First, spend some time studying the schema so you understand the relationships between the tables.

To figure out the lay of the land, run [`SHOW TABLES`](show-tables.html):

{% include copy-clipboard.html %}
~~~ sql
SHOW TABLES;
~~~

~~~
   table_name
+--------------+
  departments
  dept_emp
  dept_manager
  employees
  salaries
  titles
(6 rows)
~~~

Let's look at the schema for the `employees` table:

{% include copy-clipboard.html %}
~~~ sql
SHOW CREATE TABLE employees;
~~~

~~~
  table_name |                                    create_statement
+------------+-----------------------------------------------------------------------------------------+
  employees  | CREATE TABLE employees (
             |     emp_no INT4 NOT NULL,
             |     birth_date DATE NOT NULL,
             |     first_name VARCHAR(14) NOT NULL,
             |     last_name VARCHAR(16) NOT NULL,
             |     gender STRING NOT NULL,
             |     hire_date DATE NOT NULL,
             |     CONSTRAINT "primary" PRIMARY KEY (emp_no ASC),
             |     FAMILY "primary" (emp_no, birth_date, first_name, last_name, gender, hire_date),
             |     CONSTRAINT imported_from_enum_gender CHECK (gender IN ('M':::STRING, 'F':::STRING))
             | )
(1 row)
~~~

There's no salary information there, but luckily there is a `salaries` table.  Let's look at it.

{% include copy-clipboard.html %}
~~~ sql
SHOW CREATE TABLE salaries;
~~~

~~~
  table_name |                                          create_statement
+------------+-----------------------------------------------------------------------------------------------------+
  salaries   | CREATE TABLE salaries (
             |     emp_no INT4 NOT NULL,
             |     salary INT4 NOT NULL,
             |     from_date DATE NOT NULL,
             |     to_date DATE NOT NULL,
             |     CONSTRAINT "primary" PRIMARY KEY (emp_no ASC, from_date ASC),
             |     CONSTRAINT salaries_ibfk_1 FOREIGN KEY (emp_no) REFERENCES employees(emp_no) ON DELETE CASCADE,
             |     FAMILY "primary" (emp_no, salary, from_date, to_date)
             | )
(1 row)
~~~

OK, there's a salary field, and a foreign key on `emp_no`, which is also in the primary key of the `employees` table.

This means that to get the information we want, we'll need to do a join on `employees` and `salaries`.

#### Get row counts

Next, let's get the row counts for the tables that we'll be using in this query.  We need to understand which tables are large, and which are small by comparison.

As shown below, this data set contains about 300k employee records, and 2.8 million salary records.  This matters because later we will want to verify that we're [using the right kind of join](#use-the-right-join-type) for the relative table sizes in our query.

```sql
SELECT COUNT(*) FROM employees;
```

```
   count
+--------+
  300024
```

```sql
SELECT COUNT(*) FROM salaries;
```

```
   count
+---------+
  2844047
```

### Use the right index

Below is a query that fetches the right answer to our question: "Who are the 25 highest-paid employees?"  Note that because the `salaries` table includes historical salaries for each employee, we needed to check that the salary is current by ensuring the salary record's `to_date` column is in the future in the `WHERE` clause.

Building this query also required exploring the data a bit with some test queries and discovering that $145k was the right lower bound to use on `salary` so we weren't scanning too many records.

{% include copy-clipboard.html %}
~~~ sql
SELECT
	e.last_name, s.salary
FROM
	employees AS e
    JOIN
    salaries AS s
    ON e.emp_no = s.emp_no
WHERE
	s.salary > 145000 AND s.to_date > now()
ORDER BY
	s.salary DESC LIMIT 25;
~~~

Unfortunately, this query is pretty slow.

~~~
   last_name  | salary
+-------------+--------+
  Pesch       | 158220
  Mukaidono   | 156286
  Whitcomb    | 155709
  Luders      | 155513
  Alameldin   | 155190
  Baca        | 154459
  Meriste     | 154376
  Griswold    | 153715
  Chenoweth   | 152710
  Hatcliff    | 152687
  Birdsall    | 152412
  Stanfel     | 152220
  Moehrke     | 150740
  Junet       | 150345
  Kambil      | 150052
  Thambidurai | 149440
  Minakawa    | 148820
  Birta       | 148212
  Sudbeck     | 147480
  Unni        | 147469
  Chinin      | 146882
  Gruenwald   | 146755
  Teitelbaum  | 146719
  Kobara      | 146655
  Worfolk     | 146507
(25 rows)

Time: 2.341862s
~~~

We can see why if we look at the output of `EXPLAIN`.

~~~
            tree            |       field       |               description
+---------------------------+-------------------+-----------------------------------------+
                            | distributed       | true
                            | vectorized        | false
  render                    |                   |
   └── limit                |                   |
        │                   | count             | 25
        └── sort            |                   |
             │              | order             | -salary
             └── merge-join |                   |
                  │         | type              | inner
                  │         | equality          | (emp_no) = (emp_no)
                  │         | left cols are key |
                  │         | mergeJoinOrder    | +"(emp_no=emp_no)"
                  ├── scan  |                   |
                  │         | table             | employees@primary
                  │         | spans             | ALL
                  └── scan  |                   |
                            | table             | salaries@primary
                            | spans             | ALL
                            | filter            | (salary > 145000) AND (to_date > now())
(19 rows)
~~~

There are 2 problems:

1. We are doing full table scans on both the `employees` and `salaries` tables (see `scan > spans=ALL`).  That tells us that we don't have indexes on all of the columns in our `WHERE` clause, which is [an indexing best practice](indexes.html#best-practices).
2. We are using a merge join even though a lookup join is better when one of the tables is much smaller than the other, as it is in this case.

The first problem is more urgent.  We need indexes on all of the columns in our `WHERE` clause.  Specifically, we need to create indexes on:

- `salaries.salary`
- `salaries.to_date`

Let's verify what indexes exist on the `salaries` table by running [`SHOW INDEXES`](show-index.html):

{% include copy-clipboard.html %}
~~~ sql
SHOW INDEXES FROM salaries;
~~~

~~~
  table_name | index_name | non_unique | seq_in_index | column_name | direction | storing | implicit
+------------+------------+------------+--------------+-------------+-----------+---------+----------+
  salaries   | primary    |   false    |            1 | emp_no      | ASC       |  false  |  false
  salaries   | primary    |   false    |            2 | from_date   | ASC       |  false  |  false
(2 rows)
~~~

As we suspected, there are no indexes on `salary` or `to_date`, so we'll need to create indexes on those columns.

First, we create the index on the `salaries.salary` column.

{% include copy-clipboard.html %}
~~~ sql
CREATE INDEX ON salaries (salary);
~~~

Depending on your environment, this may take a few seconds, since the salaries table has ~3 million rows.

~~~
CREATE INDEX

Time: 10.589462s
~~~

Next, let's create the index on `salaries.to_date`:

{% include copy-clipboard.html %}
~~~ sql
CREATE INDEX ON salaries (to_date);
~~~

This will also take a few seconds, again due to the table size.

~~~
CREATE INDEX

Time: 10.681537s
~~~

Now that we have indexes on all of the columns in our `WHERE` clause, let's run the query again.

{% include copy-clipboard.html %}
~~~ sql
 SELECT
	e.last_name, s.salary
FROM
	employees AS e
    JOIN
    salaries AS s
    ON e.emp_no = s.emp_no
WHERE
	s.salary > 145000 AND s.to_date > now()
ORDER BY
	s.salary DESC LIMIT 25;
~~~

~~~
        last  | salary
+-------------+--------+
  Pesch       | 158220
  Mukaidono   | 156286
  Whitcomb    | 155709
  Luders      | 155513
  Alameldin   | 155190
  Baca        | 154459
  Meriste     | 154376
  Griswold    | 153715
  Chenoweth   | 152710
  Hatcliff    | 152687
  Birdsall    | 152412
  Stanfel     | 152220
  Moehrke     | 150740
  Junet       | 150345
  Kambil      | 150052
  Thambidurai | 149440
  Minakawa    | 148820
  Birta       | 148212
  Sudbeck     | 147480
  Unni        | 147469
  Chinin      | 146882
  Gruenwald   | 146755
  Teitelbaum  | 146719
  Kobara      | 146655
  Worfolk     | 146507
(25 rows)

Time: 14.37ms
~~~

Wow, that's about 140x faster than before! (2000ms/14ms)  At this point we could be pretty satisfied with this query.

### Use the right join type

{{site.data.alerts.callout_danger}}
Out of the box, the optimizer will select the right join type for your query in the majority of cases.  This statement becomes more and more true with every new release of CockroachDB.  Therefore, you should only provide [join hints](cost-based-optimizer.html#join-hints) in your query if you can **prove** to yourself through experimentation that the optimizer should be using a different join type than it is selecting.
{{site.data.alerts.end}}

In most cases, you should not need to worry about this.

Having said all of the above, here are some general guidelines for which types of joins should be used in which situations:

1. If one of the tables being joined is much smaller than the other, a [lookup join](joins.html#lookup-joins) is best.  This will ensure that the query reads rows from the smaller table and matches them against the larger table.

2. Merge joins are used for tables that are roughly similar in size.  They offer better performance than hash joins, but [have some additional requirements](joins.html#merge-joins).

3. [Hash joins](joins.html#hash-joins) are used when tables are roughly similar in size, if the requirements for a merge join are not met.

For more details, see the [join reference documentation](joins.html).

Given the example query above, we can run an [`EXPLAIN`](explain.html) and verify that it is using a lookup join as expected, given the difference in size between these two tables:

{% include copy-clipboard.html %}
~~~ sql
EXPLAIN SELECT
 e.last_name, s.salary
FROM
 employees AS e
    JOIN
    salaries AS s
    ON e.emp_no = s.emp_no
WHERE
 s.salary > 145000 AND s.to_date > now()
ORDER BY
 s.salary DESC LIMIT 25;
~~~

We can see that: yes, this query is indeed using the lookup join as expected.  No need to provide any hints, now that we've added the indexes in the previous section.

~~~
                tree               |         field         |         description
+----------------------------------+-----------------------+------------------------------+
                                   | distributed           | true
                                   | vectorized            | false
  render                           |                       |
   └── limit                       |                       |
        │                          | count                 | 25
        └── lookup-join            |                       |
             │                     | table                 | employees@primary
             │                     | type                  | inner
             │                     | equality              | (emp_no) = (emp_no)
             │                     | equality cols are key |
             └── filter            |                       |
                  │                | filter                | to_date > now()
                  └── index-join   |                       |
                       │           | table                 | salaries@primary
                       └── revscan |                       |
                                   | table                 | salaries@salaries_salary_idx
                                   | spans                 | /145001-
(17 rows)
~~~

## Schema design

2. **Will your schema design perform well under your workload?** Even if you write good SQL, you may be querying a schema that will lead to contention under your workload.  For example, you may be trying to run a write-heavy workload against a schema that uses sequential primary keys, which leads to write hotspots.  See [Schema design](#schema-design) below.

The most important factor to consider when designing your schema is your workload's data access patterns.  In particular:

- Keep records that are likely to be accessed together near each other
- Keep records that are not likely to be accessed together far from each other
- Make sure you have indexes you need for your application

For more information about how to put these rules into practice, see:

- [Choosing Index Keys for CockroachDB](https://www.cockroachlabs.com/blog/how-to-choose-db-index-keys/)
- [Unique ID Best Practices](performance-best-practices-overview.html#unique-id-best-practices)

## Cluster topology

3. **Does the cluster topology you are using match your use case?**  Are you picking the right point in the latency vs. resiliency tradeoff? See [Cluster topology](#cluster-topology) below.

The most important factor to consider in your choice of cluster topology is the latency vs. resiliency tradeoff.  In general, more resiliency will cost you in higher latency, since it requires more nodes in your cluster, spread over more availability zones.

Depending on your use case, there are a number of design options in this space to choose from.  To find the cluster topology that meets the needs of your use case, see [this list of topology patterns](topology-patterns.html).

## Foo

3 pronged approach, listed in order of most to least control by application developer (probably):
Write good SQL
Follow Rules of thumb from Optimizing OLTP Application Queries:
Queries should scan no more than dozens of rows (couple of hundred at most).
Create the right index, use the right join so that DB has to touch a handful of rows to get the results.
Pointers to SQL tuning with EXPLAIN, etc.
Use good schema design
This can list the highlights and point the user elsewhere for details
Especially important to avoid transaction contention if at all possible.
Developer may not have control of schema depending on organization / application maturity / etc.
Use the right cluster topology
Give the highlights, then punt to Topology Patterns.
Ideally, Ops team should have already handled this so you don’t have to care, but be aware as it affects query performance.  Link to ‘Troubleshooting cluster problems’ doc in this guide.
TODO: look at SQL best practices page to see if something is missing
TODO: talk about secondary indexes - does your query have covering indexes

## See also

Reference information:

- [CockroachDB Performance](performance.html)
- [SQL Performance Best Practices](performance-best-practices-overview.html)
- [Topology Patterns](topology-patterns.html)
- [SQL Tuning with `EXPLAIN`](sql-tuning-with-explain.html)
- [Joins](joins.html)

Specific tasks:

- [Connect to the Database](connect-to-the-database.html)
- [Insert Data](insert-data.html)
- [Query Data](query-data.html)
- [Update Data](update-data.html)
- [Delete Data](delete-data.html)
- [Run Multi-Statement Transactions](run-multi-statement-transactions.html)
- [Error Handling and Troubleshooting](error-handling-and-troubleshooting.html)
- [Hello World Example apps](hello-world-example-apps.html)
