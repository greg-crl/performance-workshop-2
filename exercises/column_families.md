## Column Families

Recall that in CockroachDB, a table is represented as a set of key-value (KV) pairs, where every row has a Key (the primary index) and a Value (all the columns of the row).  Recall, too, that a basic secondary index on that table is also stored as a KV pair, where the index has a Key (the index column or columns) and a Value (the primary key of the row that's being indexed).  And finally, recall that a STORING index is a KV entry where the Key is the index column (or columns) + the PK value, and the Value is the STORED collection of columns.  It's often said that the Primary Key is essentially just a STORING index with **all** the column tables specified in STORING clause.

In actuality, the columns stored in each row are the columns specified in a single column family.  Column families are a mechanism for grouping grouping columns in a table, for whatever reason.  By default, CockroachDB defines a table as consisting of a single column family, containing all the columns of the table.  But it is possible to specify additional column families to contain specific subsets of a table's columns. 

### Exercise 1

Your test database has a ```users``` table that includes the ```userid``` and hashed ```password```, along with a integer ```access_count```, and some TEXT and JSON columns containing a biography, a CV, and other documents.  In a loop, run the following  query 10000 times:

```
UPDATE users
SET access_count = access_count + 1
WHERE userid = 12345;
```

How long does it take?  Why? 

### Explanation

We know from our discussion of AOST queries that CockroachDB maintains MVCC data for each row.  that is, when you UPDATE a row, CockroachDB writes a new, updated copy of the entire row, with a new timestamp.  
This is terrifically useful for all sorts of reasons, but it does come at a cost.  Every update to a row - even if it's just a small change, like incrementing an integer value - causes a rewrites of the entire row.  

How big is the row accessed in the query above?  

```
> SHOW CREATE TABLE users;

  table_name |                  create_statement
-------------+-----------------------------------------------------
  users      | CREATE TABLE users (
             |     userid INT8 NOT NULL,
             |     password STRING NULL,
             |     access_count INT8 NULL,
             |     cv JSONB NULL,
             |     bio STRING NULL,
             |     notes STRING NULL,
             |     CONSTRAINT users_pkey PRIMARY KEY (userid ASC)
             | )
             

> SELECT userid, LENGTH(bio) AS b, LENGTH(notes) AS n, LENGTH(CV::TEXT) AS cv FROM users WHERE userid = 12345;

  userid |    b    |    n   |    cv 
---------+---------+--------+----------
  12345  | 100,000 | 50,000 | 100,000
```

The ```CV``` column is a JSON column that contains 100,000 characters.  The ```bio``` column is a STRING column that contains 100,000 characters.  And ```notes``` is a STRING column that contains another 50,000 characters.  Every update to the ```access_count``` column of this user record causes CRDB to rewrite a new record that's a quarter of a megabyte in size.  That's a lot of data to write.  And it's not data that changes very often.  

### Exercise 2

Let's try an exercise where we're just accessing the data, not updating it.  Query the database for the sum of all user accesses.  How long does it take?  How much memory does it require?  Why?

### Explanation

Summing the accesses by all our users is simple:
```
SELECT SUM(access_count) AS TotalAccessCount
FROM users;
```

There's no filter in this query, so it will execute a full table scan.  There are only 25,000 records in the ```users``` table, so we might expect it to complete quickly.  

But this query against our test database takes a surprisingly long time (several seconds) to return.  And you can check the memory usage for this query by checking the DB Console Hardware dashboard, or the SQL Activity page.  The query produces a signficant spike in memory usage.  Both of these effects are a result of the fact that each record in the table is roughly 250K in size.  Each row must be read into memory in order to parse out the single value the query is interested in.  That means the query read about 6.25MB of data.

### Exercise 3

What if we could split this row into two pieces, one that contains the very large and very seldom updated STRING and JSON columns, and another that contains the rest?

We can.  That's what column families are for.

It is possible to move the ```CV```, ```bio```, and ```notes``` columns into a separate column family, but it's a complex multi-step process, and outside the scope of this workshop.  For simplicity's sake, we've included a second table ```users_colfam```, in your test database, with these 3 columns defined as a separate family.  Here's what that looks like:

```
> SHOW CREATE TABLE users_colfam;

  table_name |                  create_statement
-------------+-----------------------------------------------------
  users      | CREATE TABLE users_colfam (
             |     userid INT8 NOT NULL,
             |     password STRING NULL,
             |     access_count INT8 NULL,
             |     cv JSONB NULL,
             |     bio STRING NULL,
             |     notes STRING NULL,
             |     CONSTRAINT users_pkey PRIMARY KEY (userid ASC)
             |     FAMILY f1 (password, access_count),
             |     FAMILY f2 (cv, bio, notes)
             | )
```             

Re-run the previous exercise 1, incrementing the ```access_count```  10000 times for user 12345, but this time in the ```users_colfam``` database.  How does this run compare to the last?  Why?

### Explanation

You should see a marked improvement in query throughput in this exercise.  As before, we're updating a single record 10,000 times, but each updatye requires that we rewrite only the column family that includes the ```access_count``` column, which measures a few dozen bytes.

### Exercise 4

Re-run the previous exercise 2, summing the ```access_count``` for all users, but this time in the ```users_colfam``` database.  How does this run compare to the last?  Why?

### Explanation

Similarly, you should see a marked improvement in query this exercise over exercise 2.  True, this query reads the same number of records as in the previous run, but the total amount of data read is a small fraction of the data we had to read before.  

### Conclusion

A good use case for column families in CockroachDB is when you have a table with a mix of hot and cold data. "Hot" data refers to columns that are frequently accessed or updated, while "cold" data refers to columns that are rarely accessed.  One tradeoff here is that reading all the data for a single row requires accessing both column families.  Since each column family is represented as a separate KV entry, a single row requires two read operations. 

While this technique can be used to optimize both read and write speed, but the improvement is a function of both the data stored in the database and the queries used to access it.  As always, optimizations are applied to improve specific queries, but it's important to consider both the benefits ```and``` the costs of any optimization.

[Next Section](end.md)

