# Working with Apache Spark

In this tutorial, we will use Apache Spark for `WebLog` analysis.
We will be using the `dataframe` API as it is the most common Structured API in
Spark. A dataframe represents a table of data with rows and columns. A dataframe
always has a `schema` defining the column data types and some additional `metadata`
like `nullable` indicating if the column accepts `nulls`.

In this tutorial, we will use [Apache Zeppelin](https://zeppelin.apache.org/)
notebook as our interactive development environment.

You can follow the instructions in the [install](https://zeppelin.apache.org/docs/0.7.3/install/install.html)
guide to install Zeppelin on your machine. Zeppelin comes with Spark binaries
so there is no need to install Spark separately!

## Load the web logs

The `Apache server log` is a text based format with custom structure (similar to
tabular format). It can't be directly loaded into Apache Spark.
We need to parse it line by line.

This can be done by reading a text file into an `RDD`, mapping every line into a
`pyspark.sql.Row` and transform the `RDD` into a Spark `dataframe`.

An Apache server log line can be parsed using:

```
import re # regular expression
from pyspark.sql import Row

# regular expression to parse an Apache Server log line
# It is constructed as follows
# IP_Address client_id user_id date_time method endpoint protocol response_code content_size
LOG_PATTERN = '^(\S+) (\S+) (\S+) \[([\w:/]+\s[+\-]\d{4})\] "(\S+) (\S+) (\S+)" (\d{3}) (\d+)'

# Returns a dataframe Row including named arguments. The fields in a Row are
# sorted by name. The fields of the returned row correspond to the Apache Server
# Log fields.
def log_parser(line):
  # Now match the given input line with the regular expression
  match = re.search(LOG_PATTERN, line)

  # If there is no match, then the input line is invalid: report an error
  if match is None:
    raise Error("Invalid input line: %s" % line)

  # return a pyspark.sql.Row
  return Row(
    ip_address    = match.group(1), # IP address of the client (IPv4 or IPv6)
    client_id     = match.group(2), # Clientid: mostly '-'. This info should never be used as it is unreliable.
    user_id       = match.group(3), # The userid as defined in HTTP authentication
                                    # If the endpoint is not password protected, it will be '-'
    date_time     = match.group(4), # The time the server finished processing the request
                                    # date_time format: day/month/year:hour:minute:second zone
    method        = match.group(5), # The HTTP method
    endpoint      = match.group(6), # The requested endpoint
    protocol      = match.group(7), # The protocol in use, usually HTTP/version
    response_code = int(match.group(8)), # one of HTTP's response codes (< 599).
    content_size  = int(match.group(9)) # content size as reported in the HTTP
  )
```

Now load the Weblog data

```
# path to weblog data
logfile = 'REPLACE_WITH_PATH_TO_WEBLOG.LOG'

df = sc.textFile(logfile) # reads the Weblog as a text file and returns it as RDD of lines
  .map(log_parser) # applies the log_parser function to every line in the RDD
  .toDF() # Transforms the RDD into a Spark Dataframe
```

In the next section, we will apply multiple transformations to the weblog dataframe.
In such case, it is beneficial to `cache` the dataframe in order to accelerate
future access.

```
df = df.cache()
```

## Explore the Weblog data

Now that the log in cached in a dataframe, let's explore some of its content:

```
df.show() # prints up to 20 rows (default) of the dataframe
```

Zeppelin provides a better way to explore dataframes as tables or using charts
as follows:

```
z.show(df)
```

The schema of the dataframe was defined by our custom parser. We can check it using:

```
df.printSchema()
```

When exploring a data set, it is always interesting to get per column statistics
as count, min, max, average and standard deviation values. This can be done using:

```
# In plain Spark
df.describe().show()

# OR in Zeppelin
z.show(df.describe())
```

## Cleaning up Weblog data

### Removing empty columns

Some of the values in Apache Log are optional, this is the case of `client_id`
and `user_id`. Check if these 2 columns are reported in the logs, and remove the
 corresponding columns if this is not the case.

> Hint you might want to check unique values of these columns.

```
# check distinct user and client ids
ids = df.select('user_id', 'client_id').distinct().collect()

print ids
# In Zeppelin
z.show(ids)
```

As you can see, both columns have just one unique value: `-`. Let's now remove
the columns from the dataframe:

```
# Dropping user_id and client_id as they have no actual values, just the '-'.
ndf = df.drop('user_id', 'client_id')

ndf.show()
# In Zeppelin
z.show(ndf)
```

### Casting from string to time

As you could see in the datframe Schema, `date_time` column is in String format.
as `dd/MMM/yyyy:HH:mm:ss Z`. When manipulating dates or date time data, it is
always better to have it in the corresponding type.

Spark provides utility functions to work with dates and times. Let's create two columns:
* `ts`: with `date_time` column converted into a timestamp (`DateType`)
* `day`: with `date_time` column converted into `dd-MM-yyy` format.

We will use the following function (assuming Spark version >= v2.2):
* `to_timestamp`: casts a String formatted date time into a `DateType` object
  using an optionally provided format. This function is new in version 2.2.
* `date_format`: converts a `DateType` into a String in the specified format.

```
from pyspark.sql import functions as F

ndf = ndf.withColumn('ts', F.to_timestamp(ndf.date_time, 'dd/MMM/yyyy:HH:mm:ss Z'))
ndf = ndf.withColumn('day', F.date_format(ndf.ts, 'dd-MM-yyyy'))

ndf.show()
# In Zeppelin
z.show(ndf)
```

For Spark version 2.1 or earlier, we can use:

```
from pyspark.sql import functions as F

ndf = ndf.withColumn('ts', F.unix_timestamp(ndf.date_time, 'dd/MMM/yyyy:HH:mm:ss Z'))
ndf = ndf.withColumn('day', F.from_unixtime(ndf.ts, 'dd-MM-yyyy'))

ndf.show()
# In Zeppelin
z.show(ndf)
```

## Analysing Weblog data

Let's now analyse the Weblog data to produce some reports. We will try to identify:
* Top IP addresses: IP addresses with the largest number of queries. Remember a
  Web query corresponds to a Weblog data entry. For every IP address, report the
  queries count and the content size.
* Top Endpoints: Endpoints with the largest number of requests. For every Endpoint,
  report the queries count and the total content size.
* Top Methods: number of queries and total content per Method.
* Requests count per day: Number of requests per day.
* Distribution of response codes.

### Top IP addresses

In order to get the top IP addresses with respect to queries count and content size,
we need to `groupBy` IP addresse then `aggregate` by row `count` and by `sum` of
content size. This can be done as follows:

```
from pyspark.sql import functions as F

top_ips = ndf.groupBy('ip_address').agg(F.count(ndf.method).alias('count'), F.sum(ndf.content_size).alias('size'))

top_ips.sort(F.desc('count')).limit(10).show()
## In Zeppelin
z.show(top_ips.sort(F.desc('count')).limit(10))
```

In this report we used the following:
* [pyspark.sql.DataFrame.groupBy](http://spark.apache.org/docs/2.1.0/api/python/pyspark.sql.html#pyspark.sql.DataFrame.groupBy)
  to group the Weblog data using the `ip_address` column.
* [pyspark.sql.GroupedData.agg](http://spark.apache.org/docs/2.1.0/api/python/pyspark.sql.html#pyspark.sql.GroupedData.agg)
  to aggregate grouped data according to the specified aggregation functions.
* [pyspark.sql.function.count](http://spark.apache.org/docs/2.1.0/api/python/pyspark.sql.html#pyspark.sql.functions.count)
  to return the number of items in a group.
* [pyspark.sql.function.sum](http://spark.apache.org/docs/2.1.0/api/python/pyspark.sql.html#pyspark.sql.functions.sum)
  to return the sum of all values, in our case the content size.
* [pyspark.sql.Column.alias](http://spark.apache.org/docs/2.1.0/api/python/pyspark.sql.html#pyspark.sql.Column.alias)
  to rename a column.

### Top Endpoints

Similar to top IP addresses, we can get top endpoints as follows:

```
from pyspark.sql import functions as F

top_endpoints = ndf.groupBy('endpoint').agg(F.count(ndf.method).alias('count'))

top_endpoints.sort(F.desc('count')).limit(10).show()
## In Zeppelin
z.show(top_endpoints.sort(F.desc('count')).limit(10))
```

### Top Methods

Top methods can be computed as follows:

```
from pyspark.sql import functions as F

top_methods = ndf.groupBy('method').agg(F.count(ndf.method).alias('count'), F.sum(ndf.content_size).alias('size'))

top_methods.sort(F.desc('count')).limit(10).show()
## In Zeppelin
z.show(top_methods.sort(F.desc('count')).limit(10))
```

### Queries per day

Number of queries per day can be calculated by grouping row count by `day` column:

```
from pyspark.sql import functions as F

daily_requests = ndf.groupBy('day').agg(F.count(ndf.method).alias('count'))

daily_requests.show()
## In Zeppelin
z.show(daily_requests)
```

Zeppelin provides interesting capabilities to graphically represent data. This
allows to create graphical report to visualize and explore the data while interactively
creating our data processing pipelines.

### Response code distribution

When analyzing Web server logs, it is always important to analyze response codes.
The number or ratio of `4xx` and `5xx` response codes can be indicators of problems
affecting the services. We can calculate the distribution of response codes as
follows:

```
from pyspark.sql import functions as F

codes = ndf.groupBy('response_code').agg(F.count(ndf.method).alias('count'))

codes.show()
## In Zeppelin
z.show(codes)
```

## Writing Data to File System

Spark defines an `interface`, `DataFrameWriter`, to write data to storage systems.
Storage systems can be HDFS, local file system, key-value stores, etc.

The `DataFrame.write()` method can be used to access the implemented storage systems.
Out of the box, Spark can write local file system and HDFS.

> Writing to Local File system: use this only on your single instance cluster or
for development. Writing to Local File system is a very bad idea on a Spark cluster.
You should rather use HDFS or alternative distributed and durable stores.

### Writing in CSV format

You can write to CSV format as follows:

```
ndf.write \
  .format('csv') \
  .save('PATH_TO_FOLDER/myweblog.csv')
```

This is equivalent to:

```
ndf.write \
  .csv('PATH_TO_FOLDER/myweblog.csv')
```

Try to run the write operation again! You should get an error `path already exists`.
By default, the behaviour of Spark Writer is to raise an error if it tries
writing to existing destination. You can override the default behavior by setting
the mode option. You can select from:
* `append`: appends content of the dataframe to write to existing data.
* `overwrite`: Replaces existing data by the content of the dataframe to write.
* `ignore`: Do nothing if the destination already exist.
* `error`: this is the default mode. It throws an error if destination already exists.

To override the destination with new data:

```
ndf.write \
  .mode("overwrite") \
  .csv('PATH_TO_FOLDER/myweblog.csv')
```

The override mode is idempotent, writing multiple times will lead to the same
data on the file system. This is not the case of `append` mode.
Change your code to write with append mode, run it multiple times and see how
data changes on disk.

> Selecting the right write mode is use case specific. You should take good care
when setting the write mode.

Open the csv file (or any of them) in a text editor and see how it was saved without
the column names. To include the header line containing the column names, you need
to set the `Header` option:

```
ndf.write \
  .mode("overwrite") \
  .csv('PATH_TO_FOLDER/myweblog.csv', header=True)
```

By default, writing to csv will not compress data. You can add compression by
setting the `compression` option:

```
ndf.write \
  .mode("overwrite") \
  .option("compression", "gzip") \
  .csv('PATH_TO_FOLDER/myweblog.csv', header=True)
```

### Writing in Parquet format

Writing to parquet (or any other format) is almost identical to writing to csv
format if we don't consider the csv specific serialization options. You can write
your dataframe in Parquet format as follows:

```
ndf.write \
  .format('parquet') \
  .save('PATH_TO_FOLDER/myweblog.parquet')
```

This is equivalent to:

```
ndf.write \
  .parquet('PATH_TO_FOLDER/myweblog.parquet')
```

Check your local file system where data was written and check the size of the
Parquet data. Compare it to CSV file size on disk to see how Parquet optimizes
data thanks to its columnar format and optimization techniques.

Now use `parquet-tools` to inspect the file and answer the following questions:
* What is the size of every column on disk?
* What type of statistics Parquet stores in columns metadata?
* Does Parquet stores the data schema? if yes what is the schema? Compare it to
  the schema of Spark dataframe.
* Play with the compression mode to test `none`, `snappy` and `gzip`. What can
  you say about the compression of Parquet data?

### Writing in ORC format

Writing to ORC format is similar to writing to Parquet format. The only thing
that changes if the format option:

```
ndf.write \
  .format('orc') \
  .save('PATH_TO_FOLDER/myweblog.orc')
```

This is equivalent to:

```
ndf.write \
  .orc('PATH_TO_FOLDER/myweblog.orc')
```

Now use `orc-tools` to inspect the file and answer the following questions:
* What is the size of every column on disk?
* What type of statistics ORC stores in columns metadata?
* Does ORC stores the data schema? if yes what is the schema? Compare it to
  the schema of Spark dataframe.
* Play with the compression mode to test `none`, `snappy` and `gzip`. What can
  you say about the compression of ORC data? How does it compare to Parquet?

### Writing in Avro format

Avro is a major binary row based format in the Hadoop ecosystem. However, Spark
does not provide a native Avro connector. Avro connector is available though as
a third party package provided by Databricks. You need to explicitly import the
Avro connector.

Writing to Avro can be done as follows:

```
## Set the compression codec
spark.conf.set("spark.sql.avro.compression.codec", "uncompressed")

ndf.write \
  .format('com.databricks.spark.avro') \
  .save('PATH_TO_FOLDER/myweblog.avro')
```

Avro format does not have a shorthand as it is the case for csv, parquet and orc.

#df.coalesce(1).write.option("compression", "none").format('parquet').save('/FileStore/tables/weblog/log_data.parquet')
#df.coalesce(1).write.option("compression", "none").format('orc').save('/FileStore/tables/weblog/log_data.orc')
#spark.conf.set("spark.sql.avro.compression.codec", "uncompressed")
#df.coalesce(1).write.option("compression", "none").format('com.databricks.spark.avro').save('/FileStore/tables/weblog/log_data3.avro')
Writing data with Spark is

## Organizing Data on File System

Organizing stored data is a key point to consider when creating a data lake.
How data is organized on file system affects the performance of both the reads
and writes operations. It has also a direct impact on the volume of stored data.

### Partition by Columns

Partitioning data by columns allows to physically divide data on file system
according to the columns values. The advantage of partitioning by columns is a
much faster data access speeds. The drawback is the danger of having too much
files if the columns have a large number of unique values.

Therefore, partitioning by columns should be based on the use case requirements.

Let's see how partitioning by column works:

We will first try to partition our data by `method` column. The objective is to
have data files per `method`:

```
ndf.write \
  .partitionBy("method") \
  .mode("overwrite") \
  .parquet("PATH_TO_FOLDER/myweblog_partitioned.parquet")
```

Open a terminal and check the data structure in your destination output. You
should have something similar to:

```
bachwehbi@bachwehbi-VirtualBox:~/dev/data/weblogs$ ls -all -R myweblog_partitioned.parquet/
myweblog_partitioned.parquet/:
total 24
drwxrwxr-x  5 bachwehbi bachwehbi 4096 mai   13 11:15 .
drwxrwxr-x 10 bachwehbi bachwehbi 4096 mai   13 18:02 ..
drwxrwxr-x  2 bachwehbi bachwehbi 4096 mai   13 11:15 method=GET
drwxrwxr-x  2 bachwehbi bachwehbi 4096 mai   13 11:15 method=HEAD
drwxrwxr-x  2 bachwehbi bachwehbi 4096 mai   13 11:15 method=POST
-rw-r--r--  1 bachwehbi bachwehbi    0 mai   13 11:15 _SUCCESS
-rw-rw-r--  1 bachwehbi bachwehbi    8 mai   13 11:15 ._SUCCESS.crc

myweblog_partitioned.parquet/method=GET:
total 168
drwxrwxr-x 2 bachwehbi bachwehbi   4096 mai   13 11:15 .
drwxrwxr-x 5 bachwehbi bachwehbi   4096 mai   13 11:15 ..
-rw-r--r-- 1 bachwehbi bachwehbi 157692 mai   13 11:15 part-00000-e3ed5195-e9ac-4462-b911-053fb8084e4d.snappy.parquet
-rw-rw-r-- 1 bachwehbi bachwehbi   1240 mai   13 11:15 .part-00000-e3ed5195-e9ac-4462-b911-053fb8084e4d.snappy.parquet.crc

myweblog_partitioned.parquet/method=HEAD:
total 16
drwxrwxr-x 2 bachwehbi bachwehbi 4096 mai   13 11:15 .
drwxrwxr-x 5 bachwehbi bachwehbi 4096 mai   13 11:15 ..
-rw-r--r-- 1 bachwehbi bachwehbi 2176 mai   13 11:15 part-00000-e3ed5195-e9ac-4462-b911-053fb8084e4d.snappy.parquet
-rw-rw-r-- 1 bachwehbi bachwehbi   28 mai   13 11:15 .part-00000-e3ed5195-e9ac-4462-b911-053fb8084e4d.snappy.parquet.crc

myweblog_partitioned.parquet/method=POST:
total 1312
drwxrwxr-x 2 bachwehbi bachwehbi    4096 mai   13 11:15 .
drwxrwxr-x 5 bachwehbi bachwehbi    4096 mai   13 11:15 ..
-rw-r--r-- 1 bachwehbi bachwehbi 1319478 mai   13 11:15 part-00000-e3ed5195-e9ac-4462-b911-053fb8084e4d.snappy.parquet
-rw-rw-r-- 1 bachwehbi bachwehbi   10320 mai   13 11:15 .part-00000-e3ed5195-e9ac-4462-b911-053fb8084e4d.snappy.parquet.crc
```

See how the file system layout reflects perfectly the content of the `method`
column. Now, any query with a condition on the method column will only read the
corresponding partitions.

Try now to partition by `ip_address`, check the destination folder and answer to
the following questions:
* Why in our example partitioning by IP address is a bad idea?
* What columns in our data set you consider good choices to partition data on?

It is also possible to partition by multiple columns. How would you partition
data based on `method` and `response_code`?

```
## partition by method and response_code
## Your code goes here
```

Check the output destination on your file system.
* Does the column order in `partitionBy` has any impact?
* In the case of `method` and `response_code` columns, what would be the good order?
* Assume you have the following columns: `year`, `month`, `day`, how would you
  partition based on these columns? Why?

### Writing Sorted data

Saving sorted data to the file system can have double impact:
* reduce the required disk space when using columnar based formats
* improve the read performance of requests with conditions on the sorted columns

```
from pyspark.sql import functions as F

df \
  .sort(F.desc('date_time')) \
  .repartition(1) \
  .write \
  .mode("overwrite") \
  .parquet("PATH_TO_FOLDER/myweblog_sorted.parquet")
```

Compare now the size on disk of sorted and non sorted parquet data.
* What is the impact of sorting on `date_time` column on the file size?
  Why in your opinion?

Sorting based on *poorly selected* columns might have negative impact. Sort your
data now based on `endpoint` column then write it to parquet.
* What is the impact of sorting on `endpoint` column on the file size?
  Why in your opinion?

You can also sort data on multiple columns. Consider `endpoint`, `data_time` and
`ip_address` columns. You are asked to find the best sorting strategy that minimizes
disk space requirements.
* Does sort order have an impact?
* What is the sorting order that minimizes disk space? Why?

> Hint: you can use `parquet-tools` to debug the output files.

## Spark Streaming with Kafka

So far we have been working with batch processing with Spark. Spark is a powerful
data processing engine with streaming capabilities. Spark streaming is `mico-batching`.
This means, Spark will process data in short time windows instead of processing
on a per event basis as Kafka.

Spark streaming period is defined based on the application or use case needs.
It can be a few seconds for latency sensitive applications.

In this section, we will work with Spark streaming with Kafka.

### Install Kafka Connector

Spark does not come shipped with a Kafka connector. This dependency should be
manually added.

Spark streaming and Kafka integration guide can be found [here](https://spark.apache.org/docs/2.1.0/streaming-kafka-integration.html).
As you can see, there are two libraries with different maturity. We will be using
[spark-streaming-kafka-0-8](https://spark.apache.org/docs/2.1.0/streaming-kafka-0-8-integration.html)
in this tutorial.

There are different ways to install the Kafka connector for Spark:

**Manual Download**
Just go to [Maven Central](https://mvnrepository.com/artifact/org.apache.spark/spark-streaming-kafka-0-8-assembly_2.11/2.3.0)
locate the corresponding [jar](http://central.maven.org/maven2/org/apache/spark/spark-streaming-kafka-0-8-assembly_2.11/2.3.0/) then:

```
wget http://central.maven.org/maven2/org/apache/spark/spark-streaming-kafka-0-8-assembly_2.11/2.3.0/spark-streaming-kafka-0-8-assembly_2.11-2.3.0.jar
```

This is the only `jar` file you'll need to work with Kafka in Spark. Move the
file to the Spark dependency folder and you're ready to go. If you are using
Zeppelin with Spark integrated, run the following:

```
cd zeppelin # or whatever you named your folder
cp spark-streaming-kafka-0-8-assembly_2.11-2.3.0.jar ./interpreter/spark/dep
```

**Spark Dependency Interpreter in Zeppelin**

You can load the Kafka dependency right from the Spark dependency interpreter in
Zeppelin.

In order to load a new dependency with the Spark dependency interpreter, you need
first to `restart` the Spark interpreter. Then, in any notebook, create a
paragraph at the top of the notebook and run the following:

```
%spark.dep

z.load("org.apache.spark:spark-streaming-kafka-0-8-assembly_2.11:2.3.0")
```

Now you have Kafka connector ready.

### Working with Kafka

Before going further, you need to start the Kafka broker, create a new topic,
let's name it `mytopic`, and start a producer console. You can refer to
`working_with_kafka` tutorial for more details.

**Start the streaming context**

You can start the Spark streaming context as follows:

```
from pyspark import SparkContext
from pyspark.streaming import StreamingContext

ssc = StreamingContext(sc, 5)
```

This starts a streaming context with a window period of 5 seconds. That means,
all messages arriving in a 5 seconds window, will be processed together.

**Connect to Kafka topic**

```
from pyspark.streaming.kafka import KafkaUtils

zk = 'localhost:2181' ## Zookeeper
mytopic = 'mytopic' ## topic name

## create a kafka stream documentation at: https://spark.apache.org/docs/2.1.0/api/python/pyspark.streaming.html#pyspark.streaming.kafka.KafkaUtils
kafka_dstream = KafkaUtils.createStream(ssc, zk, 'sparkit', {mytopic: 1})
```

**Register an output operation**

When working with streams, you need to register an output operation to be executed
window period. This is basically the computation flow we want to make on arriving
data. For the moment, let's simply register an operation that will simply write
arriving messages to a text file:

```
kafka_dstream.saveAsTextFiles(prefix='PATH_TO_FOLDER/some_prefix', suffix='log')
```

**Start the streaming context**

Now that the output operation is registered, you can start the streaming context:

```
ssc.start()
```

Now go to your Kafka producer console and send few messages for about 15 seconds.

**Stop the streaming context**

Now let's stop the streaming context and check what has been done:

```
## stop the streaming context by keep the Spark context
ssc.stop(stopSparkContext=False, stopGraceFully=True)
```

Now check the destination folder where data has been written to and check the content.
You will see some folders (one per window period) containing text files with
content similar to this:

```
g-1526239245000.log/part-00000
::::::::::::::
(None, u'message1')
(None, u'message2')
...
```

These are the contents of the messages received by the Kafka connector. These are
`key:value` pairs with `key` set to null as we are not sending keyed messages.

In order to send keyed messages, check the Kafka tutorial last section. Repeat
the procedure and see how you will be getting something like:

```
g-1526239645000.log/part-00000
::::::::::::::
(u'key1', u'message1')
(u'key2', u'message2')
...
```

**Word Count**

Now let's add some processing to received messages. We will implement a `word count`.
Change your code as follows:

```
kafka_dstream = KafkaUtils.createStream(ssc, zk, 'sparkit', {mytopic: 1})

## get only the value part of message. Remember messages are key:value pairs
values = kafka_dstream.map(lambda x: x[1])

## count words in received messages
counts = values.flatMap(lambda value: value.split(' ')).map(lambda word: (word, 1)).reduceByKey(lambda a, b: a + b)

counts.saveAsTextFiles(prefix='PATH_TO_FOLDER/some_prefix', suffix='log')
```

Go over the whole process and see how you will be getting word counts now:

```
f-1526232420000.log/part-00000
::::::::::::::
(u'messge', 1)
(u'message', 2)
(u'abc', 1)
...
```

**Web log analysis**

Now combine the Weblog line parser at the beginning of this tutorial to calculate
the `method` and `endpoint` requests count.
