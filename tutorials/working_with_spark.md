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

Use the previous report's code to produce this report!

### Top Methods

Use the `Top IPs` report's code to produce this report!

### Queries per day
Use the `Top IPs` report's code to produce this report!

### Response code distribution
Use the `Top IPs` report's code to produce this report!
