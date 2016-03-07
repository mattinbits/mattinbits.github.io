---
layout: post
title: Spark caching by example
---

This post is a WIP

[Apache Spark](http://spark.apache.org/) is a powerful tool for distributed
processing of data. Data, whether represented as [RDDs](http://spark.apache.org/docs/latest/quick-start.html#more-on-rdd-operations)
or as [DataFrames](http://spark.apache.org/docs/latest/sql-programming-guide.html#dataframes)
is processed using transformations (such as `map` and `filter`) which are performed
lazily, meaning that they are only executed when the results are needed for some output.
In some cases, a particular RDD or DataFrame may be used as the basis for
multiple different downstream computations, or a computation performed multiple times.

In these cases, it may be useful to cache the data of an RDD which is used more
than once, to prevent it being computed multiple times.

In this post, the procedure of caching data is explored, along with the limitations
and options that apply to caching.

## Environment setup

Similar examples to these could be run on any Spark cluster, but the caching
results may differ depending on the version of Spark used, and the resources
assigned to the Spark components.

These examples were run using Spark 1.5.2, specifically the [binary distribution
built against Hadoop 2.6](http://www.apache.org/dyn/closer.lua/spark/spark-1.5.2/spark-1.5.2-bin-hadoop2.6.tgz).

The following `spark-defaults.conf` was used:


    spark.driver.memory 1g
    spark.executor.memory 1g
    spark.master spark://<your machine name>:7077


The following `spark-env.sh` was used:

    export SPARK_WORKER_MEMORY=1g
    export SPARK_WORKER_CORES=1
    export SPARK_DRIVER_MEMORY=1g

The environment variable `SPARK_HOME` is set to the location that the spark
download archive was extracted to. Spark is started with:

    $SPARK_HOME/sbin/start-master.sh

    $SPARK_HOME/sbin/start-slave.sh spark://<your machine name>:7077

You should be able to navigate to `http://localhost:8080/` to see
the Web UI for the Spark Master.

## Examples

The interactive Spark REPL can be started by running `$SPARK_HOME/bin/spark-shell`.

A small RDD can be created an cached in the REPL with:

{% highlight Scala %}
val smallRdd = sc.parallelize(List("a", "b", "c")).setName("small RDD")
smallRdd.cache().foreach(_ => {})
{% endhighlight %}

The `foreach` is necessary since even caching is performed lazily. An RDD
is cached the first time it is used, then that cached data is available to any
subsequent usages. Using `foreach` in this manner avoids bringing any data to
the driver, which can be costly in terms of performance.

After running this example in the spark shell (leaving the REPL open to keep
the application alive), you should be able to navigate to the Application's
Web UI from the Master Web UI. The Application UI will probably be at
`http://localhost:4040`.

On the Storage tab, you should see an entry such as:

RDD Name | Storage Level | Cached Partitions | Fraction Cached | Size in Memory |	Size in ExternalBlockStore | Size on Disk
------|----------------|----|----|----|------|--------------
small RDD | Memory Deserialized 1x Replicated | 2 |	100% | 192.0 B | 0.0 B | 0.0 B

Calling `cache()` on an `RDD` is simply a special case of calling the `unpersist()`
method with a storage type of `MEMORY_ONLY`. Another option is `MEMORY_ONLY_SER`
which serialized the contents of the RDD into byte buffers rather than caching the
Java objects themselves. This generally leads to significantly smaller memory usage.

For example:

{% highlight Scala %}
import org.apache.spark.storage.StorageLevel

val mediumRdd = sc.parallelize(List(1 to 5000)).
  map(i => s"Some data $i").
  setName("med RDD")

mediumRdd.persist(StorageLevel.MEMORY_ONLY).
foreach(_ => {})

val mediumRddSer = sc.parallelize(List(1 to 5000)).
  map(i => s"Some data $i").
  setName("med RDD Serialized")

mediumRddSer.persist(StorageLevel.MEMORY_ONLY_SER).
foreach(_ => {})
{% endhighlight %}

The Storage tab for application UI should show entries such as:

RDD Name | Storage Level | Cached Partitions | Fraction Cached | Size in Memory |	Size in ExternalBlockStore | Size on Disk
------|----------------|----|----|----|------|--------------
med RDD | Memory Deserialized 1x Replicated | 2 |	100% | 4.9 KB | 0.0 B | 0.0 B
med RDD Serialized | Memory Serialized 1x Replicated | 2 |	100% | 2.4 KB | 0.0 B | 0.0 B

{% highlight Scala %}
val hugishRdd =sc.parallelize(0 to 4999).
  flatMap(i => (0 to 999).
  map(x => i*1000 + x)).
  map(i => s"data value of $i")

hugishRdd.persist(StorageLevel.MEMORY_ONLY).foreach(_ => {})

val otherHugishRdd =sc.parallelize(0 to 4999).
  flatMap(i => (0 to 999).
  map(x => i*1000 + x)).
  map(i => s"data value of $i")

otherHugishRdd.persist(StorageLevel.MEMORY_ONLY_SER).foreach(_ => {})
{% endhighlight %}
