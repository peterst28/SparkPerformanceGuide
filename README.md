# Spark Performance Guide

This guide focuses on Databricks, but may be relevant to other versions of Spark as well.

## Setting Session Configurations

In Python:

```python
spark.conf.set("<config>", "<value>")
```

In SQL

```sql
SET <config> = <value>
```

## Shuffle Partitions

#### How to configure

In SQL:
```sql
SET spark.sql.shuffle.partitions = <num partitions>
```

#### Summary

Shuffle partitions are the parts a dataframe is broken up to in a shuffle.  Too few partitions, and the size of the data that each task has to operate on is too big and spills out of memory to the disk.  Too many partitions, and the management overhead slows Spark to a crawl. 

#### Deep Dive

Certain operations require cross-worker communication.  A join is an example of this.  One way to implement a join is to make sure matching rows are sent to the same worker so they can be joined together.  To do this, a shuffle is performed.  The dataframe is broken up into shuffle partitions, and those partitions are mapped to a worker.  How a particular row ends up in a particular partition depends on the operation, but the number of partitions is of serious interest for performance optimization.  Too few partitions, and the size of the data that each task has to operate on is too big and spills out of memory to the disk.  Too many partitions, and the management overhead slows Spark to a crawl.  This is an important balance.  

Ideally you want your shuffle partitions to be somewhere in the range of 100MB to 1GB.  You achieve this by this simple formula:

(ideal number of partitions) = (dataframe size) / (target partition size)

Keep in mind that once you go over 30,000 partitions, the overhead of managing those partitions may not be worth the spill avoidance.

Another important concept is shuffle blocks.  Shuffle blocks are the smallest units of data that are transferred between workers.

The steps of a shuffle are:

1. Scan a dataframe
2. For each dataframe partition, write one shuffle block per shuffle partition and save it to the local disk
3. When a task needs a shuffle block, it will download it from the worker that has it and save it to its local disk

The number of shuffle blocks can get very large very quickly.  The formula is:

(number of shuffle blocks) = (number of partitions of the source) x (number of shuffle partitions)

#### Personal Experience

Too many shuffle partitions:

I had a customer with a 3.5 TB table that needed to be joined with about a dozen fact tables in one SQL statement.  Our initial thought was that we should set the shuffle partitions to 40,000 in order to get a reasonable partitions size.  However, when we ran the job we found that the CPU on the driver was maxed out, and the workers were practically idle.  The job never finished, slowing down as the job progressed until it stopped making progress and we saw FetchFailedException errors in the logs.  After about 6 hours and little progress, we killed the job.  On investigation, we realized we were generating 1.6 billion shuffle blocks (40,000 x 40,000) for each join!  This completely swamped the driver.  We reduced the number of shuffle partitions to 3,000, and the job completed in under 3 hours.

#### Delta Caching

Databricks only!  This is only compatible with the i3-series of VMs on AWS and Ls-series on Azure.  On these VM types, Delta caching is on by default.

#### How to configure

In SQL:
```sql
SET spark.databricks.io.cache.enabled = true/false
```

#### Summary

Delta caching makes use of fast SSD drives on the workers to cache Delta tables that have been recently queried.  Only Delta tables are cached, not results, so if many operations are performed and the results queried repeatedly, the operations will be re-performed on each query.  To improve performance, save the results to a Delta table and query from the results table.  The results will then be cached and performant.
