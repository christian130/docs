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
3. [Use the right join type, maybe](#use-the-right-join-type) for the tables you are querying.

For example, let's look at a query using the [Employees data set](https://github.com/datacharmer/test_db).  Import it with:

{% include copy-clipboard.html %}
~~~ sql
CREATE DATABASE IF NOT EXISTS employees;
USE employees;
IMPORT MYSQLDUMP 'https://s3-us-west-1.amazonaws.com/cockroachdb-movr/datasets/employees-db/mysqldump/employees-full.sql.gz';
~~~

The question we want to answer is "who are my top 25 highest-paid employees?"

To answer this we will write a simple join query against the employees and salaries tables, something like (conceptually):

```sql
SELECT
	e.last_name, s.salary
FROM
	employees AS e
    JOIN
    salaries AS s
    ON e.emp_no = s.emp_no
WHERE
	s.salary > ?
ORDER BY
	s.salary DESC
    LIMIT 25;
```

This will make more sense soon: read on.

### Scan at most a few dozen rows

#### Study the schema

First, spend some time studying the schema so you understand the relationships between the tables.

In this case you have already been told you will be joining the `employees` and `salaries` tables, so look at the relationsips between them using `SHOW COLUMNS FROM`

TODO: show create table, etc.

#### Get row counts

First, get the row counts for the tables involved in the query.  You need to understand which tables are large, and which are small by comparison.

In this data set, there are about 300k employee records, and 2.8 million salary records.

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

#### Write the queries and see how they run

Now let's analyze the actual queries.  For each query, find out how many rows it may need to scan.  This can be done using one of the following techniques:

1. Look at the output of [`EXPLAIN`](explain.html).  This can work if you know how to [read `EXPLAIN` output](sql-tuning-with-explain.html) and already know how much data is in each table.

2. Break apart the query and and run each piece to understand how many rows are being scanned.  This is easier, but requires you to have a test environment available.

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

{% include copy-clipboard.html %}
~~~ sql
SELECT stats.key                AS product_key,
       SUM(product_stats.total) AS total,
       p.id                     AS product_id
  FROM
    product_stats AS stats
    JOIN
    products AS p
    ON p.id = stats.id
  WHERE
    p.category = ?
    AND p.brand = ?
  GROUP BY stats.key,
           p.id
  ORDER BY max(p.created_on) DESC
  LIMIT ?
~~~

### Use the right index

Or really, use any index at all.

TODO: add 2 indexes to salaries and to_date

### Use the right join type

Note: you shouldn't mess with this unless you KNOW you need to override the join type selected by the CBO (link).  As time goes on, this becomes a worse and worse idea.



General rules for join types:

1. If one side A is much smaller than the other side B, use a lookup join with A on the left side.  This will ensure that the query touches only the smaller number of rows in A.

`SELECT * FROM A LOOKUP JOIN B ON A.id = B.id ...`

2. Hash join and merge join both iterate over each row on the left hand side, and then over each row on the right hand side.  They should be avoided in cases when A is much smaller than B.

For more details, see the [join reference documentation](joins.html).

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

- [CockroachDB Performance](cockroachdb-performance.html)
- [SQL Performance Best Practices](sql-performance-best-practices.html)
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
