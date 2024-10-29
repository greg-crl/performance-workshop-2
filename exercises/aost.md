## Time Travel queries, or Follower Reads

Because CockroachDB maintains MVCC values for every record, the system has a complete picture of the database as it existed at every moment from "now" back to the beginning of the garbage collection window.  

Remember, old MVCC values are discarded - that is, "garbage collected" - when they age out of the configured garbage collection window.  Remember, too, that there may be multiple different garbage collection windows, as those are specified with the "time to live" value in a ```ZONE CONFIGURATION```.

A "time travel" query is a query that specifies you want the data as of a specific past point in time.  In CockroachDB this is known as an ```AS OF SYSTEM TIME``` query (abbreviated AOST) because you issue the query with a clause specifying that you want the data ```AS OF SYSTEM TIME x```, where ```x``` specifies a timestamp in the past.  An AOST query can (and should) specify a time in the past that's older than the closed timestamp for the range.  That is, the query is requesting data old enough that no new write to that data could possibly have an earlier timestamp.  That means that an AOST query cannot be subject to contention, because the data it's looking for can't be updated at that timestamp. 

Remember, too, that CockroachDB replicates its data (organized into "ranges"), and that one replica of every range is the "raft leader" and "leaseholder", responsible for coordinating writes to and serving current reads from that data.  All the other replicas of that range are called "follower replicas".   Follower replicas might not have the latest and greatest value, but they do have values older than the closed timestamp.  So CockroachDB can serve AOST queries from follower replicas, instead of having to request the data from the leaseholder.  That's why these queries are also called "follower reads".  So follower reads can improve query performance a second way; the CockroachDB optimizer is locality-aware, and will serve a follwer read from the geographically closest replica (not necessarily the leaseholder), improving latency due to decreased network communications time.

Today, the closed timestamp for a given range is almost never more than 4.8 seconds in the past.  So CockroachDB provides a convenience function ```follower_read_timestamp()``` that returns a timestamp just far enough in the past to (virtually) guarantee a follower read.

The tradeoff, of course, is that follower reads return data that maybe slightly out of date.  So follower reads / time travel queries have specific uses.

### Exercise

In this exercise, we'll "accidentally" delete the line items from an Order, then recover them from a time in the past.

First, read the line items from an order.
```
SELECT * from OrderItems where OrderID = 12345;
```

Then delete those line items:
```
DELETE from OrderItems where OrderID = 12345;
```

And verify the line items are gone:
```
SELECT * from OrderItems where OrderID = 12345;
```

Now issue a time travel query to get those line items as they existed before you deleted them.

### Explanation

```
SELECT * from OrderItems where OrderID = 12345 AS OF SYSTEM TIME '-2m';
```

Note that we're querying the database for records as they existed 2 minutes ago.  That's because we need to look at the database as it was long enough ago that the records were still there.  If you use  ```follower_read_timestamp()```, then you're looking at the database from just 5 seconds ago, and (unless you're doing this exercise very quickly) the line items were already gone that short a time ago.

### Exercise

While a range is being frequently updated, a query for that data will contend with the writes, and can be significantly delayed.

In your testbed, run the program ```fast_writes.sh``` to generate a constant stream of updates to the ```Products``` table.  Then run a query to get a list of Products with prices > $50.  How long does it take the query to return?  Try it a few times.  Is the query latency consistent, or pretty variable?  Review the DB Console SQL Activity page, find your query, and see how much contention it suffers.

### Explanation

The query is a simple one:
```
SELECT FROM Products WHERE Price > 50;
```
But it consistently takes longer than a simple full table scan.  That's because the ```fast_writes`` script is updating Products records at high speed, and CockroachDB has to restart the query a number of times before it can succeed.  When it finally does succeed, it's because the query's priority has been raised to a level that preempts the competing writes... which mean that the query is having a negative effect on the write-heavy workload.

### Exercise

Assume that the list you want is meant to be a published price list.  It isn't the prices that are changing in the database; the workload is upating other columns in the Product records. Rewrite your query as a follower read.  How quickly does it run?  Does it run that fast all the time?

### Explanation

Here's the updated query:
```
SELECT FROM Products WHERE Price > 50 AS OF SYSTEM TIME follower_read_timestamp();
```

This query consistetly runs as fast as the system can perform a full table scan of the Products table, and does not suffer any contention with the write workload.


### Conclusion

Time travel queries can be useful for a variety of reasons:
* **Auditing and compliance**: You can query historical data to verify what the data looked like at a certain point, useful for audit trails and legal compliance.
* **Debugging**: If a bug or data issue occurred at a specific time, you can retrieve data as it existed just before or at the time of the issue.
* **Accidental data loss**: If someone mistakenly deleted or altered data, you can retrieve what the data looked like before the change.
* **Analytics & reporting**: You can run reports on past data without needing to keep manual snapshots, and without suffering from contention with a running workload.
* **Dashboards**: If the data you need isn't really expected to be current, as when you populate a dashboard that won't refresh for another 30 seconds, then you can populate it with data that might have been a few seconds old when it first displays.

[Next Section](column_families.md)
