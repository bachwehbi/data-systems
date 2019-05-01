
# Working with Spark and Kafka

## Spark Streaming with Kafka

So far we have been working with batch processing with Spark. Spark is a powerful
data processing engine with streaming capabilities. Spark streaming is `mico-batching`.
This means, Spark will process data in short time windows instead of processing
on a per event basis as Kafka.

Spark streaming period is defined based on the application or use case needs.
It can be a few seconds for latency sensitive applications.

In this section, we will work with Spark streaming with Kafka.



### Install Kafka Connector

Spark does not come shipped with a Kafka connector. This dependency should be manually added.

Spark streaming and Kafka integration guide can be found [here](https://spark.apache.org/docs/2.4.1/streaming-kafka-integration.html).
As you can see, there are two libraries with different maturity. We will be using
[spark-streaming-kafka-0-8](https://spark.apache.org/docs/2.4.1/streaming-kafka-0-8-integration.html)
in this tutorial as it is the one supporting Python.

The Kafka connector `spark-streaming-kafka-0-8_2.11` and its dependencies can be directly added to `pyspark` (or `spark-submit`) using `--packages`.

Run the following:

```
pyspark --packages org.apache.spark:spark-streaming-kafka-0-8_2.11:2.4.1
```

Now you have Kafka connector ready in your pyspark interactive session in Jupyter.

### Working with Kafka

Before going further, you need to start the Kafka broker, create a new topic,
let's name it `mytopic`, and start a producer console. You can refer to
`working_with_kafka` tutorial for more details.

**Start the streaming context**

You can start the Spark streaming context as follows:




```python
from pyspark import SparkContext
from pyspark.streaming import StreamingContext

ssc = StreamingContext(sc, 5)
```

This starts a streaming context with a window period of 5 seconds. That means,
all messages arriving in a 5 seconds window, will be processed together.

**Connect to Kafka topic**


```python
from pyspark.streaming.kafka import KafkaUtils

zk = 'localhost:2181' ## Zookeeper
mytopic = 'mytopic' ## topic name

## create a kafka stream documentation at: https://spark.apache.org/docs/2.4.1/api/python/pyspark.streaming.html#pyspark.streaming.kafka.KafkaUtils
kafka_dstream = KafkaUtils.createStream(ssc, zk, 'sparkit', {mytopic: 1})

```

**Register an output operation**

When working with streams, you need to register an output operation to be executed
window period. This is basically the computation flow we want to make on arriving
data. For the moment, let's simply register an operation that will simply write
arriving messages to a text file:



```python
kafka_dstream.saveAsTextFiles(prefix='PATH_TO_FOLDER/some_prefix', suffix='log')

```

**Start the streaming context**

Now that the output operation is registered, you can start the streaming context:



```python
ssc.start()

```

Now go to your Kafka producer console and send few messages for about 15 seconds.


**Stop the streaming context**

Now let's stop the streaming context and check what has been done:



```python
## stop the streaming context by keep the Spark context
ssc.stop(stopSparkContext=False, stopGraceFully=True)

```

Now check the destination folder where data has been written to and check the content.
You will see some files (one per window period) containing text files with
content similar to this:

```
(u'key1', u'message1')
(u'key2', u'message2')
```

Now read the data into Spark dataframe (you will need to parse the content first):


```python
from pyspark.sql import Row
from ast import literal_eval

def make_tuple(line):
    tp = literal_eval(line)
    return Row(
        key = str(tp[0]),
        value = tp[1]
    )

df = sc.textFile('PATH_TO_FOLDER/some_prefix*').map(make_tuple).toDF()

df.show()
```

**Word Count**

Now let's add some processing to received messages. We will implement a `word count`.
Change your code as follows:




```python
## get only the value part of message. Remember messages are key:value pairs
values = kafka_dstream.map(lambda x: x[1])

## count words in received messages
counts = values.flatMap(lambda value: value.split(' ')).map(lambda word: (word, 1)).reduceByKey(lambda a, b: a + b)

counts.saveAsTextFiles(prefix='PATH_TO_FOLDER/some_prefix', suffix='log')
```

Go over the whole process and see how you will be getting word counts now:

```
(u'message1', 1)
(u'message2', 2)
(u'abc', 1)
```

Now adapt the Spark text file reader to show the word count results.

### Submitting the Spark Streaming job with spark-submit

So far, we have worked with Jupyter interactive notebooks. Next, we will launch the Spark Streaming job with `spark-submit`.

You need to create a Python file containing the pyspark code to connect to Kafka topic and store word count.

The script will be something similar to:

```python
#!/usr/bin/env python
# coding: utf-8

from pyspark import SparkContext
from pyspark.streaming import StreamingContext
from pyspark.streaming.kafka import KafkaUtils

zk = 'localhost:2181' ## Zookeeper
mytopic = 'mytopic' ## topic name

# create Spark Context
sc = SparkContext(appName="kafkastream")

# create Spark Streaming Context
ssc = StreamingContext(sc, 5)

kafka_dstream = KafkaUtils.createStream(ssc, zk, 'sparkit', {mytopic: 1})

values = kafka_dstream.map(lambda x: x[1])

counts = values.flatMap(lambda value: value.split(' ')).map(lambda word: (word, 1)).reduceByKey(lambda a, b: a + b)

counts.saveAsTextFiles(prefix='PATH_TO_FOLDER/count', suffix='log')

ssc.start()

#wait for the execution to stop. For example, on user termination.
ssc.awaitTermination()

ssc.stop(stopGraceFully=True)

```

There are two differences with what we used to do in the notebook:

* Creating the Spark Context: this was automatically done in the notebook when running pyspark.
  We need to explicitly start the Spark Context otherwise.

  ```python
  # create Spark Context
  sc = SparkContext(appName="kafkastream")
  ```

* Instruct the Spark Streaming Context to wait the execution to stop. Otherwise, Spark will exit directly.

  ```python
  ssc.awaitTermination()
  ```

To submit the job:

```
# instruct pyspark to use python
export PYSPARK_DRIVER_PYTHON="python3"
export PYSPARK_DRIVER_PYTHON_OPTS=""

spark-submit --master local[2] --packages org.apache.spark:spark-streaming-kafka-0-8_2.11:2.4.1 PATH_TO_SCRIPT.py
```

This tells Spark to run the python script in standalone (`local`) mode and using two threads for the execution. Spark Streaming requires a minimum of two threads to operate.
