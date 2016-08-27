---
layout: post
title: Spark Structured Streaming - The Absolute Basics
---

Spark 2.0 introduces a new mode of computation for Spark,
[Structured Streaming](https://databricks.com/blog/2016/07/28/structured-streaming-in-apache-spark.html).
This post gives a very basic fully working example of using Structured Streaming.
The full code is available [here on github](https://github.com/mattinbits/blog_examples/tree/master/spark-structured-streaming-example).

First, we need a source of streaming data. Like traditional Spark streaming,
Spark Structured Streaming can read from a simple socket server, so we can write
one in Scala to output the stream we want:

{% highlight scala %}
object SocketServer extends App {

  val sentences = Array(
    "the cat sat on the mat",
    "to be or not to be",
    "what's the story morning glory"
  )
  val iterator = Iterator.continually(sentences).flatten

  val server = new ServerSocket(9999)
  val s = server.accept()
  val out = new PrintStream(s.getOutputStream())
  while (true) {
    out.println(iterator.next())
    Thread.sleep(100)
  }
}
{% endhighlight %}

This code continuously writes the sentences to the client which connects to the socket.
So, we can run a basic word count example using Spark Structured Streaming:

{% highlight scala %}
object StructuredStreaming extends App {

  /*
    This is Spark 2.0, so  construct a
    SparkSession rather than Context.
   */
  val spark = SparkSession
    .builder
    .master("local[*]")
    .appName("StructuredNetworkWordCount")
    .getOrCreate()

  import spark.implicits._
  val lines = spark.readStream
    .format("socket")
    .option("host", "localhost")
    .option("port", 9999)
    .load()

  // Split the lines into words
  val words = lines.as[String].flatMap(_.split(" "))
  words.explain(true)
  // Generate running word count
  val wordCounts = words.groupBy("value").count()
  val query = wordCounts.writeStream
    .outputMode("complete")
    .format("console")
    .start()

  query.awaitTermination()
}
{% endhighlight %}
