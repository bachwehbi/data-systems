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
