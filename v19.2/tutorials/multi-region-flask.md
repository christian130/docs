---
title: Developing and Deploying a Multi-Region Web Application with CockroachDB
summary: Learn how to build and deploy a multi-region web application on CockroachDB, using Flask, SQLAlchemy, CockroachCloud, and Google Cloud services.
toc: true
---

This guide walks you through developing and deploying a multi-region web application built on CockroachDB.

Throughout the guide, we reference the source code for an example web application that we built for the fictional vehicle-sharing company [MovR](../v19.2/movr.html). The application is written in Python, using Flask and SQLAlchemy. The source code for this application is open source and available on GitHub, in the [movr-flask repo](https://github.com/cockroachlabs/movr-flask). The code is well-commented, with docstrings defined at the beginning of each class and function definition.

The repo's [README](https://github.com/cockroachlabs/movr-flask/README.md) also includes instructions on debugging and deploying the application. For production deployment, we use Google Cloud services.

## Overview

MovR offers users a platform for sharing vehicles, like scooters, bicycles, and skateboards. MovR is currently available in select cities in the United States and Europe. To serve users in these two continents, they need a globally-available application.

When building a global application, it's important to consider *resilience* and *latency* in design and deployment decisions. Both of these factors affect the health of the application, the integrity of its data, and the individual user experience.

### Resilience in global applications

For an application to be resilient to system failures, the application's server and database need to be deployed in multiple locations.

Cloud services and deployment orchestration tools facilitate the deployment of applications in multiple regions. Later in this guide, we provide detailed steps for [deploying the example multi-region application](multi-region-flask.html#multi-region-application-deployment-gke).

CockroachDB is built for distributed deployments. Each deployment of CockroachDB replicates and distributes its data across all instances of the database, making the deployment resilient to system failures. Later in this guide, we discuss [multi-region database deployment](multi-region-flask.html#multi-region-database-deployment) in the context of our example application. For more information about replication and resilience in CockroachDB, see [Life of Distributed Transaction](../v19.2/architecture/life-of-a-distributed-transaction.html).

### Latency in global applications

Limiting latency improves user experience, and can also help avoid problems with data integrity, like [transaction contention](../v19.2/performance-best-practices-overview.html#understanding-and-avoiding-transaction-contention).

If MovR's application and database servers are deployed in a single region, latency can be a serious problem for users located in cities outside the deployment region. A simple distributed deployment might not improve latency over a single-region deployment, especially if client requests are sent to any server in the deployment, without consideration for the client's location.

#### Database latency

To limit database latency in a distributed database deployment, the data should be [geo-partitioned](../v19.2/topology-geo-partitioned-replicas.html). Geo-partitioning enables you to control where data is stored. Requests for data in a particular partition never leave the region to which a partition is constrained. For more about geo-partitioning in the context of our example application, see [Geo-partitioining](multi-region-flask.html#geo-partitioning).

#### Application latency

If you are building an application, it's likely that the end user will not be making requests to the database directly. Instead, the user makes requests to the application, and the application makes requests to the database on behalf of the user. To limit database latency on the application side, you need to write your application so that it only communicates with relevant, geo-partitioned data located in relative proximity to the application. To limit application latency for global applications, the application should be locality-aware, preferably behind a global load balancer. Later in the guide, we cover setting up a multi-cluster ingress and configuring custom HTTP headers for client location, as a part of the [multi-region application deployment steps](multi-region-flask.html#multi-region-application-deployment-gke). We also discuss [handling client location server-side](multi-region-flask.html#client-location).


In the sections that follow, we cover some best practices in database schema creation and database deployment with an example database and deployment. We also cover some basics for developing a locality-aware, global application, with detailed deployment steps.

## Before you begin

Before you begin, make sure that you have installed the following on your local machine:

- [CockroachDB](../v19.2/install-cockroachdb-mac.html)
- [Python 3](https://www.python.org/downloads/)
- [`pipenv`](https://docs.pipenv.org/en/latest/install/#installing-pipenv) (for debugging)
- [Google Cloud SDK](https://cloud.google.com/sdk/install) (for production deployments)
- [Docker](https://docs.docker.com/v17.12/docker-for-mac/install/) (for production deployments)
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) (for production deployments)

There are a number of Python libraries that you also need for this tutorial, including `flask`, `sqlalchemy`, and `cockroachdb`. Rather than downloading these dependencies directly from PyPi to your machine, you should list them in dependency configuration files (e.g. a `requirements.txt` file), which we will go over later in the tutorial.

Next, clone the [movr-flask](https://github.com/cockroachlabs/movr-flask) repo. After configuring just a few files, you'll be able to test out the application locally.

## The database

The MovR application is built on a multi-region deployment of CockroachDB that uses a database named `movr`. This database is a slightly simplified version of the [`movr` database](../v19.2/movr.html) that is built into the `cockroach` binary. The `movr` database for the application contains the following tables:

- `users`
- `vehicles`
- `rides`

Here's a diagram of the database schema, generated with [DBeaver](../v19.2/dbeaver.html):

<img src="{{ 'images/v19.2/movr_v2.png' | relative_url }}" alt="MovR database schema" style="border:1px solid #eee;max-width:100%" />

These tables contain information about the users and vehicles registered with MovR, and the rides associated with those users and vehicles.

The database initialization, `CREATE`, and `PARTITION` statements are all defined in `dbinit.sql`, a SQL file that you use later in this tutorial to load the database to a running cluster.

### Geo-partitioning

Distributed CockroachDB deployments consist of multiple regional deployments of CockroachDB nodes that communicate as a single, logical database (in CockroachDB terminology, a **cluster**). CockroachDB splits table data into ranges, and then replicates the ranges and distributes them to individual nodes.

At startup, each node in a cluster is assigned a [locality](../v19.2/cockroach-start.html#locality). After startup, nodes with the same locality can be assigned to the same [replication zone](](../v19.2/configure-replication-zones.html), a piece of metadata that controls the number and location of replicas for a specific set of data. When you partition data, you break up tables into segments of rows, based on a common value or characteristic. When you geo-partition the data, you constrain a partition to a specific replication zone.

Now suppose that the `movr` database is loaded to a multi-region CockroachDB cluster, with each node assigned a cloud provider `region` locality at startup.

Each table in the `movr` database contains a `city` column, which signals a location for each row of data. If a user is registered in New York, their row in the `users` table will have a `city` value of `new york`. If that user takes a ride in Seattle, that ride's row in the `rides` table has a `city` value of `seattle`.

You can [partition the tables](../v19.2/partition-by.html) of the `movr` database into regions that correspond to the replication zones, with their rows assigned to partitions, based on the row's `city` value.

For example:

~~~ sql
PARTITION BY LIST (city) (
             |     PARTITION us_west VALUES IN (('seattle'), ('san francisco'), ('los angeles')),
             |     PARTITION us_east VALUES IN (('new york'), ('boston'), ('washington dc')),
             |     PARTITION europe_west VALUES IN (('amsterdam'), ('paris'), ('rome'))
             | );
~~~

After you define a partition, you can constrain it to a replication zone, using a [zone constraint](../v19.2/configure-zone.html). For the `users` table, this looks like:

~~~ sql
ALTER PARTITION europe_west OF INDEX movr.public.users@primary CONFIGURE ZONE USING
             |     constraints = '[+region=gcp-europe-west1]';
             | ALTER PARTITION us_east OF INDEX movr.public.users@primary CONFIGURE ZONE USING
             |     constraints = '[+region=gcp-us-east1]';
             | ALTER PARTITION us_west OF INDEX movr.public.users@primary CONFIGURE ZONE USING
             |     constraints = '[+region=gcp-us-west1]'
~~~

For full partitioning statements for each table and secondary index, see `dbinit.sql`. See below for the create statements for the tables.

### The `users` table

Here is the create statement for the `users` table:

~~~
CREATE TABLE users (
      id UUID NOT NULL,
      city STRING NOT NULL,
      first_name STRING NULL,
      last_name STRING NULL,
      email STRING NULL,
      username STRING NULL,
      password_hash STRING NULL,
      is_owner BOOL NULL,
      CONSTRAINT "primary" PRIMARY KEY (city ASC, id ASC),
      UNIQUE INDEX users_username_key (username ASC),
      FAMILY "primary" (id, city, first_name, last_name, email, username, password_hash, is_owner),
      CONSTRAINT check_city CHECK (city IN ('amsterdam':::STRING, 'boston':::STRING, 'los angeles':::STRING, 'new york':::STRING, 'paris':::STRING, 'rome':::STRING, 'san francisco':::STRING, 'seattle':::STRING, 'washington dc':::STRING))
  );
~~~

Note the following:

- There is a composite primary key index on the `city` and `id` columns. We want to partition this table on the `city` column. In order to partition on a column value, the column must be indexed. For performance, the `city` column precedes the `id` column, guaranteeing that scans on the `users` table are ordered by the `city` first, and then by `id`. [Primary keys](../v19.2/primary-key.html) also imply [unique](../v19.2/unique.html) and [`NOT NULL`](../v19.2/not-null.html) constraints on the constrained column pair.
- There is a [secondary index](../v19.2/indexes.html) on the `username` column, which optimizes scans with a filter on the `username` column. Although explicitly stated here, CockroachDB automatically applies a [unique](../v19.2/unique.html) constraint to columns that are indexed.
- There is a [check constraint](../v19.2/check.html) on the `city` column, which verifies that the value of the `city` column is valid. When [querying partitions](../v19.2/partitioning.html#filtering-on-an-indexed-column), check constraints also optimize scans that have filters on the constrained columns.

### The `vehicles` table

~~~
CREATE TABLE vehicles (
      id UUID NOT NULL,
      city STRING NOT NULL,
      type STRING NULL,
      owner_id UUID NULL,
      date_added DATE NULL,
      status STRING NULL,
      last_location STRING NULL,
      color STRING NULL,
      brand STRING NULL,
      CONSTRAINT "primary" PRIMARY KEY (city ASC, id ASC),
      CONSTRAINT fk_city_ref_users FOREIGN KEY (city, owner_id) REFERENCES users(city, id),
      INDEX vehicles_auto_index_fk_city_ref_users (city ASC, owner_id ASC, status ASC) PARTITION BY LIST (city) (
          PARTITION us_west VALUES IN (('seattle'), ('san francisco'), ('los angeles')),
          PARTITION us_east VALUES IN (('new york'), ('boston'), ('washington dc')),
          PARTITION europe_west VALUES IN (('amsterdam'), ('paris'), ('rome'))
      ),
      FAMILY "primary" (id, city, type, owner_id, date_added, status, last_location, color, brand),
      CONSTRAINT check_city CHECK (city IN ('amsterdam':::STRING, 'boston':::STRING, 'los angeles':::STRING, 'new york':::STRING, 'paris':::STRING, 'rome':::STRING, 'san francisco':::STRING, 'seattle':::STRING, 'washington dc':::STRING))
  );
~~~

Note the following:

- Like the `users` table, the `vehicles` table has a composite primary key on `city` and `id`.
- The `vehicles` table has a foreign key constraint on the `users` table, for the `city` and `owner_id` columns. This guarantees that a vehicle is registered to a particular user (i.e. an "owner") in the city where that user is registered.
- The table has a secondary index (`vehicles_auto_index_fk_city_ref_users`) on the `city`, `owner_id`, and `status`. By default, CockroachDB creates secondary indexes for all foreign key constraints, to optimize scans on the foreign key columns for foreign key enforcement. Here, we add `status` to the secondary index, because reading and writing the status of a vehicle is a common query for the application.
- The `vehicles_auto_index_fk_city_ref_users` is partitioned. When geo-partitioning a database, it's important to geo-partition all indexes containing partition values. In this case, the index includes `city`, so we can partition the index. After defining this partition, you also need to add a zone constraint to the partition, so that it is truly geo-partitioned. For example, for the `us_east` partition, we use the following statement to configure the zone:

    ~~~ sql
    ALTER PARTITION us_east OF INDEX movr.public.vehicles@vehicles_auto_index_fk_city_ref_users CONFIGURE ZONE USING
    constraints = '[+region=gcp-us-east1]';
    ~~~

    See `dbinit.sql` for full zone configuration statements.
- Like `users`, the table also has a `check` constraint on the `city` row, to optimize table scans in filtered queries.

### The `rides` table

~~~
  CREATE TABLE rides (
      id UUID NOT NULL,
      city STRING NOT NULL,
      vehicle_id UUID NULL,
      rider_id UUID NULL,
      rider_city STRING NOT NULL,
      start_location STRING NULL,
      end_location STRING NULL,
      start_time TIMESTAMPTZ NULL,
      end_time TIMESTAMPTZ NULL,
      length INTERVAL NULL,
      CONSTRAINT "primary" PRIMARY KEY (city ASC, id ASC),
      CONSTRAINT fk_city_ref_users FOREIGN KEY (rider_city, rider_id) REFERENCES users(city, id),
      CONSTRAINT fk_vehicle_city_ref_vehicles FOREIGN KEY (city, vehicle_id) REFERENCES vehicles(city, id),
      INDEX rides_auto_index_fk_city_ref_users (rider_city ASC, rider_id ASC) PARTITION BY LIST (rider_city) (
          PARTITION us_west VALUES IN (('seattle'), ('san francisco'), ('los angeles')),
          PARTITION us_east VALUES IN (('new york'), ('boston'), ('washington dc')),
          PARTITION europe_west VALUES IN (('amsterdam'), ('paris'), ('rome'))
      ),
      INDEX rides_auto_index_fk_vehicle_city_ref_vehicles (city ASC, vehicle_id ASC) PARTITION BY LIST (city) (
          PARTITION us_west VALUES IN (('seattle'), ('san francisco'), ('los angeles')),
          PARTITION us_east VALUES IN (('new york'), ('boston'), ('washington dc')),
          PARTITION europe_west VALUES IN (('amsterdam'), ('paris'), ('rome'))
      ),
      FAMILY "primary" (id, city, rider_id, rider_city, vehicle_id, start_location, end_location, start_time, end_time, length),
      CONSTRAINT check_city CHECK (city IN ('amsterdam':::STRING, 'boston':::STRING, 'los angeles':::STRING, 'new york':::STRING, 'paris':::STRING, 'rome':::STRING, 'san francisco':::STRING, 'seattle':::STRING, 'washington dc':::STRING))
  );
~~~


Note the following:

- Like the `users` and `vehicles` table, the `rides` table has a composite primary key on `city` and `id`.
- Like the `vehicles` table, the `rides` table has foreign key constraints. These constraints are on the `users` and the `vehicles` table.
- The foreign key indexes are partitioned. These partitions need to be zone-constrained. For full zone configuration statements, see `dbinit.sql`.
- Like `users` and `vehicles`, the table has a `check` constraint on the `city` row, to optimize table scans in filtered queries.

## Setting up the development environment

At this point, you should be familiar with the `movr` database schema. Let's set up the CockroachDB cluster, and initialize the database.

### Setting up a virtual, multi-region CockroachDB cluster

In production, you want to start a secure CockroachDB cluster, with nodes on machines located in different areas of the world. For debugging and development purposes, you can just use the [`cockroach demo`](../v19.2/cockroach-demo.html) command. This command starts up an insecure, virtual nine-node cluster. After you are done debugging the application, you can move on to a [multi-region database deployment](multi-region-flask.html#multi-region-database-deployment).

To set up the virtual, multi-region cluster:

1. Run `cockroach demo`, with the `--nodes` and `--demo-locality` flags. The localities specified below assume GCP region names.

    ~~~ shell
    $ cockroach demo \
    --nodes=9 \
    --demo-locality=region=gcp-us-east1:region=gcp-us-east1:region=gcp-us-east1:region=gcp-us-west1:region=gcp-us-west1:region=gcp-us-west1:region=gcp-europe-west1:region=gcp-europe-west1:region=gcp-europe-west1
    ~~~

    ~~~
    root@127.0.0.1:<some_port>/movr>
    ~~~

    Keep this terminal window open. Closing it will shut down the virtual cluster.

1. Copy the connection string at the prompt (e.g., `root@127.0.0.1:<some_port>/movr`).

1. In a separate terminal window, run the following command to load `dbinit.sql` to the demo database:

    ~~~ shell
    $ cockroach sql --insecure --url='postgresql://root@127.0.0.1:<some_port>/movr' < dbinit.sql
    ~~~

    This file contains the `movr` database definition, and SQL instructions to geo-partition the database.

1. Verify that the database schema loaded properly:

    ~~~ sql
    > SHOW TABLES;
    ~~~
    ~~~
      table_name
    +------------+
      rides
      users
      vehicles
    (3 rows)
    ~~~


### Setting up a virtual environment

In production, you want to containerize your application and deploy it with k8s. For debugging, use [`pipenv`](https://docs.pipenv.org/en/latest/install/#installing-pipenv), a tool that manages dependencies with `pip` and creates virtual environments with `virtualenv`.

1. Run the following command to initialize the project's virtual environment:

    ~~~ shell
    $ pipenv --three
    ~~~

    `pipenv` creates a `Pipfile` in the current directory. Open this `Pipfile`, and edit it to read as follows:

    ~~~ toml
    [[source]]
    name = "pypi"
    url = "https://pypi.org/simple"
    verify_ssl = true

    [dev-packages]

    [packages]
    cockroachdb = "*"
    psycopg2-binary = "*"
    SQLAlchemy = "*"
    SQLAlchemy-Utils = "*"
    Flask = "*"
    Flask-SQLAlchemy = "*"
    Flask-WTF = "*"
    Flask-Bootstrap = "*"
    Flask-Login = "*"
    WTForms = "*"
    gunicorn = "*"
    geopy = "*"

    [requires]
    python_version = "3.7"
    ~~~

1. Run the following command to install the packages listed in the `Pipfile`:

    ~~~ shell
    $ pipenv install
    ~~~

1. Pipenv automatically sets any variables defined in a `.env` file as environment variables in a Pipenv virtual environment. To connect to a SQL database (like CockroachDB) from a client, you need a [SQL connection string](https://en.wikipedia.org/wiki/Connection_string).

    So, open the `.env`, and then define the connection string in that file as the `DB_URI` environment variable. Note that for SQLAlchemy, the connection string protocol needs to be specific to the CockroachDB dialect.

    For example:

    ~~~
    DB_URI = 'cockroachdb://root@127.0.0.1:52382/movr'
    ~~~

    You can also specify other variables in this file that you'd rather not hard-code in the application, like API keys and secret keys used by the application. For debugging purposes, you should leave these variables as they are.


1. Activate the virtual environment:

    ~~~ shell
    $ pipenv shell
    ~~~

    The prompt should now read `~bash-3.2$`. From this shell, you can run any Python3 application with the required dependencies that you listed in the `Pipfile`, and the environment variables that you listed in the `.env` file. You can exit the shell subprocess at any time with a simple `exit` command.

1. To test out the application, you can simply run the server file:

    ~~~ shell
    $ python3 server.py
    ~~~

    You can alternatively use [gunicorn](https://gunicorn.org/) for the server.

    ~~~ shell
    $ gunicorn -b localhost:8000 server:app
    ~~~

1. Navigate to the URL provided to test out the application.


## The application

Now that we have initialized the database, and set up the database and development environment, let's look at the example application source code.

### Project structure

Our application needs to handle requests from clients, namely web browsers. To translate these kinds of requests into database transactions, the application stack consists of the following components:

- A multi-node, geo-distributed CockroachDB cluster, with each node's locality corresponding to cloud provider regions.
- A geo-partitioned database schema that defines the tables and indexes for user, vehicle, and ride data.
- Python class definitions that map to the tables in our database.
- A backend API that defines the application's connection to the database and the database transactions.
- A Flask server that handles requests from client mobile applications and web browsers.
- HTML files that define web pages that the Flask server can host.

Here is the application's project directory structure:

~~~ shell
movr
├── Dockerfile  ## Defines the Docker container build
├── LICENSE  ## A license file for your application
├── Pipfile ## Lists PyPi dependencies for pipenv
├── Pipfile.lock
├── README.md  ## Contains instructions on running the application
├── __init__.py
├── dbinit.sql  ## Initializes the database and partitions the tables by region
├── mcingress.yaml ## Defines the multi-cluster ingress K8s object, for multi-region deployment
├── movr
│   ├── __init__.py
│   ├── models.py  ## Defines classes that map to tables in the movr database
│   ├── movr.py  ## Defines the primary backend API
│   └── transactions.py ## Defines callback functions
├── movr.yaml  ## Defines service and deployment K8s objects for multi-region deployment
├── requirements.txt  ## Lists PyPi dependencies for Docker container
├── server.py  ## Defines a Flask web application that to handle requests from clients and render HTML files
├── static  ## Static resources for the web frontend
│   ├── movr-lockup.png
│   └── movr-symbol.png
├── templates  ## HTML templates for the web frontend
│   ├── _nav.html
│   ├── home.html
│   ├── layout.html
│   ├── login.html
│   ├── register.html
│   ├── rides.html
│   ├── user.html
│   ├── users.html
│   ├── vehicles-add.html
│   └── vehicles.html
└── web
    ├── __init__.py
    ├── config.py   ## Contains Flask configuration settings
    ├── forms.py  ## Defines FlaskForm classes
    ├── geoutils.py  ## Defines utility functions for the web application
    └── gunicorn.py  ## Contains gunicorn configuration settings
~~~

In the sections that follow, we go over each of the files and folders in the project, with a focus on the backend and database components.

### Using SQLAlchemy with CockroachDB

Object Relational Mappers (ORM's) map classes to tables, class instances to rows, and class methods to transactions on the rows of a table. The `sqlalchemy` package includes some base classes and methods that you can use to connect to your database's server from a Python application, then map tables in that database to Python classes. In this tutorial, you'll use SQLAlchemy's [Declarative](https://docs.sqlalchemy.org/en/13/orm/extensions/declarative/) extension, which is built on the `mapper()` and `Table` functions. You'll also use the [`cockroachdb`](https://github.com/cockroachdb/cockroachdb-python) Python package, which includes some functions that help you handle [transactions](../v19.2/transactions.html) in a running CockroachDB cluster.

#### Declaring mappings

Let's start by mapping Python classes to the tables in the `movr` database. By now, you should be familiar with the `movr` database and each of the tables in the database (`users`, `vehicles`, and `rides`).

Open `models.py`, under the `movr` folder, and look at the first 10 lines of the file:

~~~ python
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy import Column, Index, String, DateTime, Integer, Boolean, Float, Interval, ForeignKey, CheckConstraint
from sqlalchemy.types import DECIMAL, DATE
from sqlalchemy.dialects.postgresql import UUID, ARRAY
import datetime
from werkzeug.security import generate_password_hash, check_password_hash
from flask_login import UserMixin

Base = declarative_base()
~~~

The first thing the file imports is `declarative_base`, a constructor for the [declarative base class](https://docs.sqlalchemy.org/en/13/orm/extensions/declarative/api.html#sqlalchemy.ext.declarative.declarative_base), built on SQLAlchemy's Declarative extension. All mapped objects inherit from this object. We assign a declarative base object to the `Base` variable below the imports.

`models.py` also imports some other standard SQLAlchemy data structures that represent database objects (like columns and indexes), data types, and constraints, in addition to a standard Python library to help with default values (i.e. `datetime`).

Since the application handles user logins and authentication, we also need to import some security libraries. We'll cover the web development libraries in more detail when we talk about the Flask implementation of this application.

Recall that each instance of a table class represents a row in the table, so we name our table classes as if they were individual rows of their parent table, since that's what they'll represent when we construct class objects.

Now take a look at the `User` class definition:

~~~ python
class User(Base):
    __tablename__ = 'users'
    id = Column(UUID, primary_key=True)
    city = Column(String, primary_key=True)
    first_name = Column(String)
    last_name = Column(String)
    email = Column(String)
    username = Column(String, unique=True)
    password_hash = Column(String)
    is_owner = Column(Boolean)

    def __repr__(self):
        return "<User(city='%s', id='%s', name='%s')>" % (self.city, self.id, self.first_name + ' ' + self.last_name)
~~~

The `User` class holds the following attributes:

- `__tablename__`, which holds the stored name of the table in the database. SQLAlchemy requires this attribute for all classes that map to tables.
- All of the other attributes of the `User` class (`id`, `city`, `first_name`, etc.) are stored as `Column` objects. These attributes represent columns of the `users` table. The constructor for each `Column` takes the column data type as its first argument, and any constraints as additional arguments. To help define column objects, SQLAlchemy also includes classes for SQL data types and column constraints. For the columns in this table, we just use `UUID` and `String` data types. The `primary_key` constraint on the first two columns defines a composite primary key with the two columns.

The `__repr__` function defines the string representation of the object.

After you define the `User` table class, you can go ahead and define classes for the rest of the tables in the `movr` database, which are only slightly more complex.

Next, look at a definition for the `Vehicle` class:

~~~ python
class Vehicle(Base):
    __tablename__ = 'vehicles'
    id = Column(UUID, primary_key=True)
    city = Column(String, primary_key=True)
    type = Column(String)
    owner_id = Column(UUID, ForeignKey('users.id'))
    date_added = Column(DATE, default=datetime.date.today)
    status = Column(String)
    last_location = Column(String)
    color = Column(String)
    brand = Column(String)

    def __repr__(self):
        return "<Vehicle(city='%s', id='%s', type='%s', status='%s')>" % (self.city, self.id, self.type, self.status)
~~~

The `vehicles` table contains more columns and data types than the `users` table. It also contains a [foreign key constraint](../v19.2/foreign-key.html) (on the `users` table), and a default value.

Lastly, there's the `Ride` class:

~~~ python
class Ride(Base):
    __tablename__ = 'rides'
    id = Column(UUID, primary_key=True)
    city = Column(String, ForeignKey('vehicles.city'), primary_key=True)
    rider_id = Column(UUID, ForeignKey('users.id'))
    rider_city = Column(String, ForeignKey('users.city'))
    vehicle_id = Column(UUID, ForeignKey('vehicles.id'))
    start_location = Column(String)
    end_location = Column(String)
    start_time = Column(DateTime)
    end_time = Column(DateTime)
    length = Column(Interval)

    def __repr__(self):
        return "<Ride(city='%s', id='%s', rider_id='%s', vehicle_id='%s')>" % (self.city, self.id, self.rider_id, self.vehicle_id)
~~~

The `rides` table has four foreign key constraints, two on the `users` table and two on the `vehicles` table.

### Defining transactions

After you create a class for each table in the database, you can start defining functions that bundle together common SQL operations as atomic [transactions](../v19.2/transactions.html).

#### Using the `cockroachdb` Python library

The `cockroachdb` Python library handles transactions in SQLAlchemy with the `run_transaction()` function. This function takes a "`transactor`", which can be an [`Engine`](https://docs.sqlalchemy.org/en/13/core/connections.html#sqlalchemy.engine.Engine), [`Connection`](https://docs.sqlalchemy.org/en/13/core/connections.html#sqlalchemy.engine.Connection), or [`sessionmaker`](https://docs.sqlalchemy.org/en/13/orm/session_api.html#sqlalchemy.orm.session.sessionmaker) object, and a callback function. It then uses the `transactor` to connect to the database, and then executes the callback as a database transaction.

We recommend that you pass a `sessionmaker` object, bound to an existing `Engine`, to `run_transaction()`, as we do in the MovR example application. Every time `run_transaction()` is called, it uses the `sessionmaker` object to create a new [`Session`](https://docs.sqlalchemy.org/en/13/orm/session.html) object.

`run_transaction()` abstracts the details of [transaction retries](../v19.2/transactions.html#transaction-retries) away from your application code. Transaction retries are more frequent in CockroachDB than in some other databases because it uses [optimistic concurrency control](https://en.wikipedia.org/wiki/Optimistic_concurrency_control) rather than locking. Because of this, a CockroachDB transaction may have to be tried more than once before it can commit. This is part of how we ensure that our transaction ordering guarantees meet the ANSI [SERIALIZABLE](https://en.wikipedia.org/wiki/Isolation_(database_systems)#Serializable) isolation level.

`run_transaction()` has the following additional benefits:

- When passed a [sqlalchemy.orm.session.sessionmaker](https://docs.sqlalchemy.org/en/latest/orm/session_api.html#session-and-sessionmaker) object (*not* a `session`), it ensures that a new session is created exclusively for use by the callback, which protects you from accidentally reusing objects via any sessions created outside the transaction.
- It abstracts away the [client-side transaction retry logic](../v19.2/transactions.html#client-side-intervention) from your application, which keeps your application code portable across different databases. For example, the sample code given on this page works identically when run against Postgres (modulo changes to the prefix and port number in the connection string).
- It will run your transactions to completion much faster than a naive implementation that created a fresh transaction after every retry error. Because of the way the CockroachDB dialect's driver structures the transaction attempts (using a [`SAVEPOINT`](../v19.2/savepoint.html) statement under the hood, which has a slightly different meaning in CockroachDB than in some other databases), the server is able to preserve some information about the previously attempted transactions to allow subsequent retries to complete more easily.

 Because all callback functions are passed to `run_transaction()`, the `Session` method calls within those callback functions are written a little differently than the typical SQLAlchemy application. Most importantly, those functions must not change the session and/or transaction state. This is in line with the recommendations of the [SQLAlchemy FAQs](https://docs.sqlalchemy.org/en/latest/orm/session_basics.html#session-frequently-asked-questions), which state (with emphasis added by the original author) that

 > As a general rule, the application should manage the lifecycle of the session *externally* to functions that deal with specific data. This is a fundamental separation of concerns which keeps data-specific operations agnostic of the context in which they access and manipulate that data.

 and

 > Keep the lifecycle of the session (and usually the transaction) **separate and external**.

 In keeping with the above recommendations from the official docs, we **strongly recommend** avoiding any explicit mutations of the transaction state inside the callback passed to `run_transaction()`, since that will lead to breakage. Specifically, we do not make calls to the following functions from inside `run_transaction()`:

 - [`sqlalchemy.orm.Session.commit()`](https://docs.sqlalchemy.org/en/latest/orm/session_api.html?highlight=commit#sqlalchemy.orm.session.Session.commit) (or other variants of `commit()`): This is not necessary because `run_transaction()` handles the savepoint/commit logic for you.
 - [`sqlalchemy.orm.Session.rollback()`](https://docs.sqlalchemy.org/en/latest/orm/session_api.html?highlight=rollback#sqlalchemy.orm.session.Session.rollback) (or other variants of `rollback()`): This is not necessary because `run_transaction()` handles the commit/rollback logic for you.
 - `Session.flush()`: This will not work as expected with CockroachDB because CockroachDB does not support nested transactions, which are necessary for `Session.flush()` to work properly. If the call to `Session.flush()` encounters an error and aborts, it will try to rollback. This will not be allowed by the currently-executing CockroachDB transaction created by `run_transaction()`, and will result in an error message like the following: `sqlalchemy.orm.exc.DetachedInstanceError: Instance <FooModel at 0x12345678> is not bound to a Session; attribute refresh operation cannot proceed (Background on this error at: http://sqlalche.me/e/bhk3)`.

 In the example application provided, all calls to `run_transaction()` are found within the methods of the `MovR` class (defined in `movr.py`), which represents the connection to the running database. Requests to the web application frontend (defined in `server.py`), are routed to the `MovR` class methods. We discuss `movr.py` and `server.py` in more detail in later sections.

#### Defining transaction callback functions

 To separate concerns, we define all callback functions passed to `run_transaction()` calls in a separate file, called  `transactions.py`. These callback functions wrap `Session` method calls, like [`session.query()`](https://docs.sqlalchemy.org/en/13/orm/session_api.html#sqlalchemy.orm.session.Session.query) and [`session.add()`](https://docs.sqlalchemy.org/en/13/orm/session_api.html#sqlalchemy.orm.session.Session.add), to perform database operations within a transaction.

This file imports all of the table classes that we defined in `models.py`, in addition to some SQLAlchemy and standard Python data structures needed to generate correctly-typed row values that the ORM can write to the database.

~~~ python
from movr.models import Vehicle, Ride, User
from sqlalchemy import cast, Numeric
import datetime
import uuid
~~~

##### Reading

A common query that a client might want to run is a read of the `rides` table.

~~~ python
def get_rides_txn(session, rider_id):
    rides = session.query(Ride).filter(
        Ride.rider_id == rider_id).order_by(Ride.start_time).all()
    return list(map(lambda ride: {'city': ride.city, 'id': ride.id, 'vehicle_id': ride.vehicle_id, 'start_time': ride.start_time, 'end_time': ride.end_time, 'rider_id': ride.rider_id, 'length': ride.length}, rides))
~~~

The `get_rides_txn()` function takes a `Session` object and a `rider_id` string as its inputs, and it outputs a list of dictionaries containing the columns-value pairs of a row in the `rides` table. To retrieve the data from the database bound to a particular `Session` object, we use `Session.query()`, a method of the `Session` class. This method returns a `Query` object, with methods for filtering and ordering query results.

Note that the `get_rides_txn()` function returns the rides for a specific rider, which can be in more than one city. The function does not filter the query by `city`, and therefore does not limit the query to ride data stored in a particular region.

Another common query would be to read the registered vehicles in a particular city, to see which vehicles are available for riding. Unlike `get_rides_txn()`, the `get_vehicles_txn()` function takes the `city` string as an input.

~~~ python
def get_vehicles_txn(session, city):
    vehicles = session.query(Vehicle).filter(
        Vehicle.city == city, Vehicle.status != 'removed').all()
    return list(map(lambda vehicle: {'city': vehicle.city, 'id': vehicle.id, 'owner_id': vehicle.owner_id, 'type': vehicle.type, 'last_location': vehicle.last_location + ', ' + vehicle.city, 'status': vehicle.status, 'date_added': vehicle.date_added, 'color': vehicle.color, 'brand': vehicle.brand}, vehicles))
~~~

This function filters the query call by city. Because the data in the vehicles table is partitioned by city, the function only queries a specific partition of data, constrained to a particular region.

##### Writing

Let's move on to some write transactions. There are two basic types of write operations: creating new rows and updating existing rows. In SQL terminology, these are `INSERT`'s and `UPDATE`'s.  All transaction callback functions that update existing rows include a `session.query()` call. All functions adding new rows call `session.add()`. Some functions do both.

`start_ride_txn()`, which is called when a user starts a ride, adds a new row to the `rides` table and then updates a row in the `vehicles` table.

~~~ python
def start_ride_txn(session, city, rider_id, rider_city, vehicle_id):
    v = session.query(Vehicle).filter(Vehicle.city == city,
                                      Vehicle.id == vehicle_id).first()
    r = Ride(city=city, id=str(uuid.uuid4()), rider_id=rider_id,
             rider_city=rider_city, vehicle_id=vehicle_id, start_location=v.last_location, start_time=datetime.datetime.now(datetime.timezone.utc))
    session.add(r)
    v.status = "unavailable"
~~~

The function takes the `city` string, `rider_id` UUID, `rider_city` string, and `vehicle_id` UUID as inputs. It queries the `vehicles` table for all vehicles of a specific ID, and in a specific city. It also creates a `Ride` object, representing a row of the `rides` table. To add the ride to the table in the database bound to the `Session`, use `Session.add()`. To update the `vehicles` table, just modify the object attribute, like any other python variable. `run_transaction()`, called by each method in `movr.py`, commits the changes to the database.

Be sure to review the other callback functions in `transactions.py` before moving on to the next section.

Now that we've created the table classes and some transaction functions, we can look at the interface that connects web requests to the running CockroachDB cluster.

### Connecting to the database

The `MovR` class, defined in `movr.py`, handles connections to CockroachDB using SQLAlchemy's [`Engine`](https://docs.sqlalchemy.org/en/13/core/connections.html#sqlalchemy.engine.Engine) and [`Session`](https://docs.sqlalchemy.org/en/13/orm/session_api.html#sqlalchemy.orm.session.Session) classes.

The `MovR` class, defined in `movr.py`, handles connections to CockroachDB using SQLAlchemy's `Engine` class.

~~~ python
from movr.transactions import start_ride_txn, end_ride_txn, add_user_txn, add_vehicle_txn, get_users_txn, get_user_txn, get_vehicles_txn, get_rides_txn, remove_user_txn, remove_vehicle_txn
from cockroachdb.sqlalchemy import run_transaction
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from sqlalchemy.dialects import registry
registry.register(
    "cockroachdb", "cockroachdb.sqlalchemy.dialect", "CockroachDBDialect")


class MovR:
    def __init__(self, conn_string):
        self.engine = create_engine(conn_string, convert_unicode=True)
~~~

This file imports the transaction callback functions that we defined in the `transactions.py` file. The `MovR` class methods use the `run_transaction()` function, imported from the `cockroachdb` Python library, to execute the callbacks as transactions.

The file also imports some `sqlalchemy` libraries to create instances of the `Engine` and `Session` classes, and to [register the CockroachDB as a dialect](https://docs.sqlalchemy.org/en/13/core/connections.html#registering-new-dialects).

When called, the constructor creates an instance of the `Engine` class using an input connection string that specifies the database dialect and connection arguments. The constructor then creates a `Session` object that binds to the active `Engine` object.

### Creating a backend API

The `MovR` class methods function as the "backend API" for the application. Frontend requests get routed to the these methods.

We've already defined the transaction logic in the callback functions in `transactions.py`. We can now wrap calls to the `run_transaction()` function in the `MovR` class methods.

For example, look at the `start_ride()` function:

~~~ python
def start_ride(self, city, rider_id, rider_city, vehicle_id):
        return run_transaction(sessionmaker(bind=self.engine), lambda session: start_ride_txn(session, city, rider_id, rider_city, vehicle_id))
~~~

The function takes some keyword arguments, and then returns the `run_transaction()` call, using a new session and the callback function `start_ride_txn()`. `start_ride()` passes `sessionmaker(bind=self.engine)` as the first argument in `run_transaction()`, creating a new `Session` object that binds to the `Engine` instance initialized by the `MovR` constructor.

The function also takes `city`, `rider_id`, `rider_city`, and `vehicle_id` as inputs. These are the same keyword arguments, and should be of the same types, as the inputs for `start_ride_txn()`.

Be sure to review the other functions in `movr.py` before moving on to the next section.

### Building the web application

By now you should have an idea of what components we need to connect to a running database, map our database objects to objects in Python, and then interact with the database transactionally. All the application needs now is a front end. We'll use Flask for the web server, routing, and forms, and some basic Bootstrapped HTML templates for the web UI.

Most of the web application components are found in the `web` and `templates` folders, and in the `server.py` file.

#### Configuring the server environment

Flask offers some pretty robust configuration options, and you can take several different approaches to setting those options. We store the configuration in simple `object`-based classes.

Open `web/config.py`. This file imports the `os` library,

~~~ python
import os


class Config(object):
    DEBUG = os.environ['DEBUG']
    SECRET_KEY = os.environ['SECRET_KEY']
    API_KEY = os.environ['API_KEY']
    DB_URI = os.environ['DB_URI']
    PREFERRED_URL_SCHEME = ('https', 'http')[DEBUG == 'True']
~~~

Rather than rewrite any lines of code when we want to debug certain functionality in the application, let's use a global Flask configuration variable, and then use the variable in the application's control flow.

#### Creating the web forms

Forms make up an important part of a web UI. We define these in `web/forms.py`, using some data structures from the `flask_wtf` and `wtforms` libraries.

We won't go into too much detail about these forms. The important thing to know is that they help handle `POST` requests from the web UI.

The `CredentialForm` class, for example, defines the fields of the login form that users interface with to send a login request to the server:

~~~ python
class CredentialForm(FlaskForm):
    username = StringField('Username: ', validators=[data_required()])
    password = PasswordField('Password: ', validators=[data_required()])
    submit = SubmitField('Sign In')
~~~

#### Initializing the application

`server.py` defines the main process (the "`main()`") of the application: the web server. After initializing the database, we run `python server.py` to start up the web server.

Let's look at the first ten lines of `server.py`:

~~~ python
from flask import Flask, __version__, render_template, session, redirect, flash, url_for, Markup, request
from flask_bootstrap import Bootstrap
from flask_login import LoginManager, current_user, login_user, logout_user, login_required
from werkzeug.security import check_password_hash
from movr.movr import MovR
from web.forms import CredentialForm, RegisterForm, VehicleForm, StartRideForm, EndRideForm, RemoveUserForm, RemoveVehicleForm
from web.config import Config
from web.geoutils import get_region
from sqlalchemy.exc import DBAPIError
~~~

The first line imports standard Flask libraries for connecting, routing, and rendering web pages. The next three lines import some libraries from the Flask ecosystem, for bootstrapping, authentication, and security.

The next four lines import other web resources that we've defined separately in our project:
- The `MovR` class, which handles the connection and interaction with the running CockroachDB cluster.
- Several `FlaskForm` superclasses, which define the structure of web forms.
- The `Config` class, which holds configuration information for the Flask application.
- `get_region()`, which matches a city to a cloud provider region, so the load balancer knows which deployment to route requests to.

Finally, we import the `DBAPIError` type from `sqlalchemy`, for error handling.

The next five or so lines initialize the application:

~~~ python
app = Flask(__name__)
app.config.from_object(Config)
Bootstrap(app)
login = LoginManager(app)
protocol = ('https', 'http')[app.config.get('DEBUG') == 'True']
~~~

Calling the `Flask()` constructor initializes a WSGI application (our Flask web server). By assigning this to a variable (`app`), we can then configure the application, store variables in its attributes, and call it with functions from other libraries to add features and functionality to the application.

For example, we can bootstrap the application for enhanced HTML and form generation (`Bootstrap(app)`). We can also add authentication (`LoginManager(app)`).

After initializing the WSGI application, we can connect to the database:

~~~ python
conn_string = app.config.get('DB_URI')
movr = MovR(conn_string)
~~~

Here, the application retrieves the connection string that is stored as a configuration variable of the `app`, and then calls the `MovR()` constructor to establish a connection to the database at the location we provided to the `Config` object.


#### User authentication

User authentication is handled with the `Flask-Login` library. This library manages user logins with the `LoginManager`, and some other functions that help verify if a user has been authenticated or not.

~~~ python
# Define user_loader function for LoginManager
@login.user_loader
def load_user(user_id):
    return movr.get_user(user_id=user_id)
~~~

By defining a `user_loader()` function, we can control whether certain routes are accessible to a client session. To restrict access to a certain page to users that are logged in with the `LoginManager`, we add the `@login_required` decorator function to the route. We'll go over some examples in the section below.

#### Routing

We define all Flask routing functions directly in `server.py`.

Flask applications use `@app.route()` decorators to handle client requests to specific URL's. When a request is sent to a URL served by the Flask app instance, the server calls the function defined within the decorator (the routing function).

Flask provides a few useful callbacks to use within a routing function definition:
- `redirect()`, which redirects a request to a different URL
- `render_template()`, which renders an HTML page, with Jinja2 templating, into a static webpage
- `flash()`, which sends messages from the application output to the webpage

In addition to calling these functions, and some other standard Flask and Python libraries, the application's routing functions need to call some of the methods that we defined in `movr.py` (the "backend API").

Let's look at a few examples, starting with the `login()` route:

~~~ python
# Login page
@app.route('/login', methods=['GET', 'POST'])
def login():
    if current_user.is_authenticated:
        return redirect(url_for('home_page', _external=True, _scheme=protocol))
    else:
        form = CredentialForm()
        if form.validate_on_submit():
            try:
                user = movr.get_user(username=form.username.data)
                if user is None or not check_password_hash(user.password_hash, form.password.data):
                    flash(Markup('Invalid user credentials.<br>If you aren\'t registered with MovR, go <a href="{0}">Sign Up</a>!').format(
                        url_for('register', _external=True, _scheme=protocol)))
                    return redirect(url_for('login', _external=True, _scheme=protocol))
                login_user(user)
                return redirect(url_for('home_page', _external=True, _scheme=protocol))
            except Exception as error:
                flash('{0}'.format(error))
                return redirect(url_for('login', _external=True, _scheme=protocol))
        return render_template('login.html', title='Log In', form=form, available=session['region'])
~~~

#### Client Location

In an earlier section ([Latency in global applications](multi-region-flask.html#latency-in-global-applications)), we discussed the importance of reducing application latency by making our application location-aware.

When a user arrives at the website, their request needs to be routed to the application deployment closest to the location from which they made the request. This step is handled outside of the application logic, by a [cloud-hosted, global load balancer](https://en.wikipedia.org/wiki/Cloud_load_balancing). In our [example production deployment](multi-region-flask.html#multi-region-application-deployment-gke), we use a [GCP multi-cluster ingress](https://cloud.google.com/kubernetes-engine/docs/how-to/multi-cluster-ingress).

For the purposes of this section of the tutorial, let's assume that the application has been deployed behind a global load balancer that routes requests to deployments, based on the location of the request. Even if the application request is routed to the correct regional deployment of the application, the application still needs to know where the client located, so that requests can be translated into database operations on the partitions relevant to the client's location (city, in this case). For example, if a user logs onto your website in New York, you want them to only see the vehicles available in New York, and you want to them to only be able to start rides in New York.

Requests include all of the information passed from a client to a server. All HTTP requests include standard HTTP headers. Client location information is not included in [standard HTTP header fields](https://en.wikipedia.org/wiki/List_of_HTTP_header_fields), but some non-standard HTTP header fields include information that can be used to derive client location, using client-side logic. For example, you can reverse geo-locate an IP address delivered through the [`X-Forwarded-For`](https://en.wikipedia.org/wiki/X-Forwarded-For) field.

Most load-balancing services require you to enable header forwarding, or the original request headers will be lost behind the load balancer. Some services, like GCP Network Services, allow you to define non-standard HTTP headers to include location information directly with [user-defined request headers](https://cloud.google.com/load-balancing/docs/user-defined-request-headers).

We discuss the details of configuring custom request header fields in our [example production deployment](multi-region-flask.html#multi-region-application-deployment-gke). For the purposes of this tutorial, let's assume that you receive the location of your client from a request header field named "`X-City`".

Now suppose that a user is coming to the website for the first time. They type in the address of the website, let's say `https://movr.cloud`, into their browser, and hit enter. The following example routing function sets gathers the client's location, stores it in a persistent Flask `session` variable, and then renders the home page:

~~~ python
@app.route('/', methods=['GET'])
@app.route('/home', methods=['GET'])
def home_page():
    if app.config.get('DEBUG') == 'True':
        session['city'] = 'new york'
    else:
        try:
            session['city'] = request.headers.get("X-City").lower()
            # This header attribute is passed by the HTTP load balancer, to its configured backend. The header must be configured manually in the cloud service provider's console to include this attribute. See README for more details.
        except Exception as error:
            session['city'] = 'new york'
            flash('{0} {1}'.format(
                error, '\nUnable to retrieve client city information.\n Application is now assuming you are in New York.'))
    session['region'] = get_region(session['city'])
    session['riding'] = None
    return render_template('home.html', available=session['region'], city=session['city'])
~~~

Perhaps the first thing that you'll notice is that the function's control flow sequence evaluates the `DEBUG` configuration variable first. If the application is in `DEBUG` mode, it greatly simplifies the work that the application needs to do with the request information by just setting the city to `new york`. This variable should be set to `True` if you are running the application against [a virtual CockroachDB cluster](multi-region-flask.html#setting-up-a-virtual-multi-region-cockroachdb-cluster).

Let's say that we are *not* in `DEBUG` mode. The request defines a new session variable, `city`, as the value in the request header field `X-City`. If the process fails for some reason (e.g., the `X-City` header field wasn't properly forwarded) then the application sets `city` to `new york`, like the `DEBUG` case, and prints the error to the rendered webpage, using Flask's `flash()` function.

In all cases, the session variables `city`, `region`, and `riding` are initialized, and then the `home.html` web page is rendered, with the `region` and `city` variables passed to the HTML template as variables. We'll discuss what those webpages do with the variables in the next section.

#### Creating a Web UI

For the purposes of this tutorial, we limit the web UI to some static HTML web pages, rendered using Flask's built-in Jinja-2 engine. We won't spend too much time covering the web UI. Just note that the forms take input from the user, and that input is usually passed to the backend where it is translated into and executed as a database transaction.

We've also added some Bootstrap syntax and Google Maps, for UX purposes. As you can see, the Google Maps API requires a key. For debugging, you can define this key in the `.env` file. If you decide to use an embedded service like Google Maps in production, you should restrict your Google Maps API key to a specific hostname or IP address from within the cloud provider's console, as the API key will be publicly visible in the HTML. In production, we use Kubernetes secrets to the store the API keys.

## Production deployment

After you finish developing and debugging a multi-region application in your local development environment, you are ready to deploy the application.

### Multi-region database deployment

In production, you want to start a secure CockroachDB cluster, with nodes on machines located in different areas of the world. To deploy CockroachDB in multiple regions, using [CockroachCloud](https://cockroachlabs.cloud):

1. Create a CockroachCloud account.

1. Request a multi-region CockroachCloud cluster on GCP, in regions us-west1, us-east1, and europe-west1.

1. After the cluster is created, open the console, and select the cluster.

1. Select **SQL Users** from the side panel, select **Add user**, give the user a name and a password, and then add the user. You can use any user name except "root".

1. Select **Networking** from the side panel, and then select **Add network**. Give the network any name you'd like, select either a **New network** or a **Public network**, check both **UI** and **SQL**, and then add the network. In this example, we use a public network.

1. Select **Connect** at the top-right corner of the cluster console.

1. Select the **User** that you created, and then **Continue**.

1. Copy the connection string, with the user and password specified.

1. **Go back**, and retrieve the connection strings for the other two regions.

1. Download the cluster cert to your local machine (it's the same for all regions).

1. Open a new terminal, and run the `dbinit.sql` file on the running cluster to initialize the database. You can connect to the database from any node on the cluster for this step.

    ~~~ shell
    $ cockroach sql --url any-connection-string < dbinit.sql
    ~~~

    {{site.data.alerts.callout_info}}
    You need to specify the password in the connection string!
    {{site.data.alerts.end}}

    e.g.,
    ~~~ shell
    $ cockroach sql --url \ 'postgresql://user:password@region.cockroachlabs.cloud:26257/defaultdb?sslmode=verify-full&sslrootcert=certs-dir/movr-app-ca.crt' < dbinit.sql
    ~~~

{{site.data.alerts.callout_info}}
You can also deploy CRDB manually. For instructions, see the [Manual Deployment](../v19.2/manual-deployment.html) page of the Cockroach Labs documentation site.
{{site.data.alerts.end}}

### Multi-region application deployment (GKE)

To deploy an application in multiple regions in production, we recommend that you use a [managed Kubernetes engine](https://kubernetes.io/docs/setup/#production-environment), like [Amazon EKS](https://aws.amazon.com/eks/), [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine/), or [Azure Kubernetes Service](https://azure.microsoft.com/en-us/services/kubernetes-service/). To route requests to the container cluster deployed closest to clients, you should also set up a multi-cluster [ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/). In this tutorial, we use [kubemci](https://cloud.google.com/kubernetes-engine/docs/how-to/multi-cluster-ingress) to configure a Cloud HTTP Load Balancer to container clusters on GKE.

To serve a secure web application, you need a public domain name!

To deploy the application globally, we recommend that you use a major cloud provider with a global load-balancing service and a Kubernetes engine. For our deployment, we use GCP. To serve a secure web application, you also need a public domain name!

1. If you don't have a glcoud account, create one at https://cloud.google.com/.

1. Create a gcloud project on the [GCP console](https://console.cloud.google.com/).

1. Enable the [Google Maps Embed API](https://console.cloud.google.com/apis/library), create an API key, restrict the API key to all subdomains of your domain name (e.g. `https://site.com/*`), and retrieve the API key.

1. Configure/authorize the `gcloud` CLI to use your project and region.

    ~~~ shell
    $ gcloud init
    $ gcloud auth login
    $ gcloud auth application-default login
    ~~~

1. If you haven't already, install `kubectl`.

    ~~~ shell
    $ gcloud components install kubectl
    ~~~

1. Build and run the Docker image locally.

    ~~~ shell
    $ docker build -t gcr.io/<gcp_project>/movr-app:v1 .
    ~~~

    If there are no errors, the container built successfully.

1. Push the Docker image to the project’s gcloud container registry.

    e.g.,
    ~~~ shell
    $ docker push gcr.io/<gcp_project>/movr-app:v1
    ~~~

1. Create a K8s cluster for all three regions.

    ~~~ shell
    $ gcloud config set compute/zone us-east1-b && \
      gcloud container clusters create movr-us-east
    $ gcloud config set compute/zone us-west1-b && \
      gcloud container clusters create movr-us-west
    $ gcloud config set compute/zone europe-west1-b && \
      gcloud container clusters create movr-europe-west
    ~~~

1. Add the container credentials to `kubeconfig`.

    ~~~ shell
    $ KUBECONFIG=~/mcikubeconfig gcloud container clusters get-credentials --zone=us-east1-b movr-us-east
    $ KUBECONFIG=~/mcikubeconfig gcloud container clusters get-credentials --zone=us-west1-b movr-us-west
    $ KUBECONFIG=~/mcikubeconfig gcloud container clusters get-credentials --zone=europe-west1-b movr-europe-west
    ~~~

1. For each cluster context, create a secret for the connection string, Google Maps API, and the certs, and then create the k8s deployment and service using the `movr.yaml` manifest file. To get the context for the cluster, run `kubectl config get-contexts -o name`.

    ~~~ shell
    $ kubectl config use-context <context-name> && \
    kubectl create secret generic movr-db-cert --from-file=cert=<full-path-to-cert> && \
    kubectl create secret generic movr-db-uri --from-literal=DB_URI="connection-string" && \
    kubectl create secret generic maps-api-key --from-literal=API_KEY="APIkey" \
    kubectl create -f ~/movr-flask/movr.yaml
    ~~~

    {{site.data.alerts.callout_info}}
    You need to do this for each cluster context!
    {{site.data.alerts.end}}

1. Reserve a static IP address for the ingress.

    ~~~ shell
    $ gcloud compute addresses create --global movr-ip
    ~~~

1. Download [`kubemci`](https://github.com/GoogleCloudPlatform/k8s-multicluster-ingress), and then make it executable:

    ~~~ shell
    $ chmod +x ~/kubemci
    ~~~

1. Use `kubemci` to make the ingress.

    ~~~ shell
    $ ~/kubemci create movr-mci \
    --ingress=<path>/movr-flask/mcingress.yaml \
    --gcp-project=<gcp_project> \
    --kubeconfig=<path>/mcikubeconfig
    ~~~

    {{site.data.alerts.callout_info}}
    `kubemci` requires full paths.
    {{site.data.alerts.end}}

1. In GCP's **Load balancing** console (found under **Network Services**), select and edit the load balancer that you just created.

    1. Edit the backend configuration.
        - Expand the advanced configurations, and add [a custom header](https://cloud.google.com/load-balancing/docs/user-defined-request-headers): `X-City: {client_city}`. This forwards an additional header to the application telling it what city the client is in. The header name (`X-City`) is hardcoded into the example application.

    1. Edit the frontend configuration, and add a new frontend.
        - Under "**Protocol**", select HTTPS.
        - Under "**IP address**", select the static IP address that you reserved earlier (e.g., "`movr-ip`").
        - Under "**Certificate**", select "**Create a new certificate**".
        - On the "**Create a new certificate**" page, give a name to the certificate (e.g., "`movr-ssl-cert`"), check "**Create Google-managed certificate**", and then under "Domains", enter a domain name that you own and want to use for your application.
    1. Review and finalize the load balancer, and then "**Update**".

    {{site.data.alerts.callout_info}}
    It will take several minutes to provision the SSL certificate that you just created for the frontend.
    {{site.data.alerts.end}}

1. Check the status of the ingress.

    ~~~ shell
    $ ~/kubemci list --gcp-project=<gcp_project>
    ~~~

1. In the **Cloud DNS** console (found under **Network Services**), create a new zone. You can name the zone whatever you want. Enter the same domain name for which you created a certificate earlier.

1. Select your zone, and copy the nameserver addresses (under "**Data**") for the recordset labeled "**NS**".

1. Outside of the GCP console, through your domain name provider, add the nameserver addresses to the authorative nameserver list for your domain name.

    {{site.data.alerts.callout_info}}
    It can take up to 48 hours for changes to the authorative nameserver list to take effect.
    {{site.data.alerts.end}}

1. Navigate to the domain name and test out your application.

1. Clean up (at your leisure).

## Upgrading your deployment

After you deploy your application, you will likely need to push changes that you've made locally to the deployed code. When pushing changes, be aware that you defined the database separate from the application. If you change a datatype, for example, in your application, you will also need to modify the database schema to be compatible with your application's requests. We recommend that you used a supported schema migration tool.

## Developing your own application

This tutorial, and the example application source code, should help you get started developing multi-region applications on CockroachDB. Although we use Flask for the web framework, there are many other web frameworks compatible with SQLAlchemy. CockroachDB is also fully compatible with other popular ORM's.

## See also

- The [SQLAlchemy](https://docs.sqlalchemy.org/en/latest/) docs
- [Transactions](../v19.2/transactions.html)
