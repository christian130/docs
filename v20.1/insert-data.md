---
title: Insert Data
summary: How to insert data into CockroachDB during application development
toc: true
---

This page has instructions for getting data into CockroachDB using various programming languages using the [`INSERT`](insert.html) SQL statement.

The instructions assume that you already have a cluster up and running using the instructions for a [manual deployment][manual] or an [orchestrated deployment][orchestrated].

The instructions assume you are already [connected to the database](connect-to-the-database.html).

{{site.data.alerts.callout_danger}}
If you are just getting started and need to get a lot of data into a new CockroachDB cluster quickly, you should use the [`IMPORT`](import.html) statement using the instructions in the [Migration Overview](migration-overview.html).  It will be much faster because it bypasses the higher-level SQL layers altogether and writes to the data store using low-level commands.
{{site.data.alerts.end}}

<div class="filters filters-big clearfix">
  <button class="filter-button" data-scope="sql">SQL</button>
  <button class="filter-button" data-scope="go">Go</button>
  <button class="filter-button" data-scope="java">Java</button>
  <button class="filter-button" data-scope="python">Python</button>
</div>

<section class="filter-content" markdown="1" data-scope="sql">

## SQL

</section>

<section class="filter-content" markdown="1" data-scope="go">

## Go

</section>

<section class="filter-content" markdown="1" data-scope="java">

## Java

TIP: be sure to enable the rewritebatchedinserts=true on your JDBC driver (in the connection parameters) to get up to a 3x speedup.

</section>

<section class="filter-content" markdown="1" data-scope="python">

## Python

</section>

## See also

Reference information related to this task:

- [Migration Overview](migration-overview.html)
- [`IMPORT`](import.html)
- [Import performance](import.html#performance)
- [`INSERT`](insert.html)
- [`UPSERT`](upsert.html)


Other common tasks:

- [Connect to the Database](connect-to-the-database.html)
- [Query Data](query-data.html)
- [Update Data](update-data.html)
- [Delete Data](delete-data.html)
- [Make Queries Fast](make-queries-fast.html)
- [Run Multi-Statement Transactions](run-multi-statement-transactions.html)
- [Error Handling and Troubleshooting](error-handling-and-troubleshooting.html)
- [Hello World Example apps](hello-world-example-apps.html)
