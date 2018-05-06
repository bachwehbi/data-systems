# Working with Window Functions in Spark

In this tutorial, we will work with Window function in Spark. Window Functions
were introduced in Spark v1.4 and are explained in this awesome Databricks
[blog post](https://databricks.com/blog/2015/07/15/introducing-window-functions-in-spark-sql.html).

For this tutorial, we will use the following sample data:

```
wdf = spark.createDataFrame([
  (1234, '2018-03-01', 40),
  (1234, '2018-03-15', 25),
  (1234, '2018-04-01', 25),
  (1234, '2018-04-30', 30),
  (4567, '2018-03-01', 200),
  (4567, '2018-03-13', 221),
  (4567, '2018-04-01', 700),
  (4567, '2018-04-06', 400),
  (4567, '2018-04-29', 500),
  (9999, '2018-03-01', 150),
  (8888, '2018-03-01', 15),
  (8888, '2018-03-30', 20)
]).toDF('id', 'start', 'nb')

wdf.createOrReplaceTempView('wdftable')

## To work with SQL, let's create a temporary view over our dataframe. A view
## can be used exactly as a table (i.e. Hive Table).
wdf.createOrReplaceTempView('wdftable')
```

In the following we will create windows over the dataframe partitioned by `id`
column.

## Define a Window

A Window over a dataframe can be defined as follows:

```
from pyspark.sql import Window
## create a window partitioned by id and order windows data by start column
win = Window.partitionBy('id').orderBy(wdf.start.desc())
```

## Window Operations

In the following we will review a large set of window functions.

### Rank

`pyspark.sql.functions.rank()` is a window function that returns the rank of
rows within a window partition.

```
from pyspark.sql import functions as F
rankfct = F.rank().over(win)
rankdf = wdf.select('*', rankTest.alias('rank'))

rankdf.show()
# In Zeppelin
z.show(rankdf)
```

In `SQL`, the same operation can be achieved with:

```
rankdf = spark.sql("SELECT id, start, nb, RANK() OVER (PARTITION BY id ORDER BY start desc) as rank FROM wdftable")
rankdf.show()
```
### Sum

When using the `sum` function over a window, we will have the cumulative `sum`
as we move over the Window. Check the results given by this sample code:

```
cumulsum = F.sum(wdf.nb).over(win)

sumdf = wdf.select('*', cumulsum.alias('sum'))

sumdf.show()
```

In `SQL`, the same operation can be achieved with:

```
sumdf = spark.sql("SELECT id, start, nb, sum(nb) OVER (PARTITION BY id ORDER BY start desc) as sum FROM wdftable")
sumdf.show()
```

### Lead

`lead` is a window function that returns the value that is one (by default) row
after the current row.

```
leadfct = F.lead(wdf.start, 1).over(win)
leaddf = wdf.select('*', leadfct.alias('previous_start'))

leaddf.show()
```

In `SQL`, the same operation can be achieved with:

```
spark.sql('SELECT id, start, nb, LEAD(start) OVER (PARTITION BY id ORDER BY start DESC) as previous_start from wdftable')
```

### Lag

`lag` is a window function that returns the value that is one (by default) row
before the current row.

```
lagfct = F.lag(wdf.start, 1).over(win)
lagdf = wdf.select('*', lagfct.alias('end'))

lagdf.show()
```

In `SQL`, the same operation can be achieved with:

```
spark.sql('SELECT id, start, nb, LAG(start) OVER (PARTITION BY id ORDER BY start DESC) as end from wdftable')
```
### First

`first` returns the first value in a window partition.

```
firstfct = F.first(wdf.start).over(win)
firstdf = wdf.select('*', firstfct.alias('first'))

firstdf.show()
```

In `SQL`, the same operation can be achieved with:

```
spark.sql('SELECT id, start, nb, FIRST(start) OVER (PARTITION BY id ORDER BY start DESC) as first from wdftable')
```

### Last

`last` returns the last value in a window partition.

```
lastfct = F.last(wdf.start).over(win)
lastdf = wdf.select('*', lastfct.alias('last_not_really'))

lastdf.show()
```

In `SQL`, the same operation can be achieved with:

```
spark.sql('SELECT id, start, nb, LAST(start) OVER (PARTITION BY id ORDER BY start DESC) as last_not_really from wdftable')
```

As you can see in the results, the newly created column by `last` window function
do not contain what we really expected. It contains exactly the same values as
the content of `start` column. This is however normal, the reason is `range` of
the Window being by default from [`Window.unboundedPreceding`, `Window.currentRow`]
which practically means from all preceding row to the current row. Using such range
it is not surprising that the `last` row will always be the current row.

In order to consider the complete range of the window partitions, we need to
specify the range:

```
## range from all preceding rows to all following rows
win = Window.partitionBy('id').orderBy(wdf.start.desc()).rowsBetween(-Window.unboundedFollowing, Window.unboundedFollowing)

lastfct = F.last(wdf.start).over(win)

lastdf = wdf.select('*', lastfct.alias('last'))

lastdf.show()
```

In `SQL`, the same operation can be achieved with:

```
spark.sql('SELECT id, start, nb, LAST(start) OVER (PARTITION BY id ORDER BY start DESC ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING) as last from wdftable')
```

> It is critical to correctly specify the range when defining a Window. The
results highly depend on this range as we have seen with `last` function.

### Max

`max` returns the maximum value in the window partition range.

Using the comments from last section, add a new column to the dataframe containing
the maximum value of `nb` column in the window partition.

Add another column containing the difference between the maximum value over the
window and the current value of the row. See how the window function can effectively
be an expression.
