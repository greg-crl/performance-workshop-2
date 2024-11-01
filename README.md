## CockroachDB Workshop: Schema and Query Techniques for optimizing performance

This is the second in a series of workshops, and consists of a series of self-paced exercises that allow the student to demonstrate and explore concepts we present in the CEA enablement on Schema Design and Query Best Practices for optimizing performance of CockroachDB.

### Setup: Preparing the data

To do this workshop, we're going to load some data files into a database on a `cockroach` cluster.  Fetch the following CSV files from the **/data** directory of this repository to your local data store:
* users.csv
* users_colfam.csv
* orders.csv
* orderitems.csv
* products.csv

### Setup: Creating a test cluster

Exercises can be run on any cluster, though the performance costs are easier to see when executed against a geographically dispersed cluster with small nodes, such as a GCE machine type `n1-highcpu-2`.  

A cluster for exercising this workshop can be set up with roachprod:
```
export CLUSTER=my_cluster
roachprod create $CLUSTER -n 3 --geo
roachprod stage $CLUSTER release latest
roachprod start $CLUSTER
```

Bring up the DB Console with the command

```
roachprod adminui $CLUSTER:1 --open
```

The exercises can be performed from a SQL client that can connect to the cluster.  The student can use a local copy of `cockroach` if they have one, or they can remote login to one of the nodes of the cluster and communicate with the local process.

The student will need to load the accompanying CSV into the database.  We’ll be loading it from `nodelocal` on one of the cluster nodes, so we’ll need the CSV placed wherever the SQL client is running.  If the student will be remoting into one of the cluster nodes, execute the following commands:

```
roachprod put users.csv $CLUSTER:1
roachprod put users_colfam.csv $CLUSTER:1
roachprod put orders.csv $CLUSTER:1
roachprod put orderitems.csv $CLUSTER:1
roachprod put products.csv $CLUSTER:1
roachprod ssh $CLUSTER:1
```

(For a more realistic setup, we could set up another server with haproxy and a copy of the cockroach SQL shell binary, but that’s unnecessary for the exercises we’re going to do.)

### Setup: Creating the test database 

Put the data files where the cluster can find them:

```
$ cockroach nodelocal upload users.csv workshop/users.csv –insecure
$ cockroach nodelocal upload users_colfam.csv workshop/users_colfam.csv –insecure
$ cockroach nodelocal upload orders.csv workshop/orders.csv –insecure
$ cockroach nodelocal upload order_items.csv workshop/order_items.csv –insecure
$ cockroach nodelocal upload products.csv workshop/products.csv –insecure
```

Connect to the cluster with the SQL client, and create a `workshop` database.  Within it, use the following command to create the table `book`:

```
$ cockroach sql –insecure

> create database workshop;
 
> use workshop;

workshop> create table users (
	userid INT8 PRIMARY KEY,
        password STRING NULL,
        access_count INT8 NULL,
        cv JSONB NULL,
        bio STRING NULL,
        notes STRING NULL
);

workshop> create table users_colfam (
        userid INT8 PRIMARY KEY,
        password STRING NULL,
        access_count INT8 NULL,
        cv JSONB NULL,
        bio STRING NULL,
        notes STRING NULL,
        FAMILY f1 (password, access_count),
        FAMILY f2 (cv, bio, notes)
);

workshop> IMPORT INTO users (title, author_name, price, format, publish_date) CSV DATA ('nodelocal://node/workshop/users.csv');

workshop> IMPORT INTO users_colfams (title, author_name, price, format, publish_date) CSV DATA ('nodelocal://node/workshop/users_colfam.csv');
```

Check your work.  You should have two user tables with 25K entries each:

```
workshop> show tables;
  schema_name |  table_name  | type  | owner | estimated_row_count | locality
--------------+--------------+-------+-------+---------------------+-----------
  public      | users        | table | root  |            25000     | NULL
  public      | users_colfam | table | root  |            25000     | NULL
```

Now that we're set up, let's get started with the exercises.

[Next Section](exercises/denormalization.md)
