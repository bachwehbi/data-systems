# Working with Apache Kafka

Still Work in Progress

## Install Kafka

Create a new folder `dev` in your home directory, then douwnload latest version (v-1.0.1) of Apache Kafka.

```
mkdir dev
cd dev
wget http://apache.crihan.fr/dist/kafka/1.0.1/kafka_2.11-1.0.1.tgz
tar -xzvf kafka_2.11-1.0.1.tgz
mv kafka_2.11-1.0.1 kafka
```

## Run Zookeeper

Kafka uses Zookeeper for various tasks including:

* Electing a controller and ensuring there is only one and elect a new one if it crashes.
* Maintaining the Kafka cluster membership to track brokers joins and leaves: which brokers are part of the cluster.
* Topic configuration management to track existing topics, topics partitions, topics replicas, etc.
* Quota configuration: defining the maximum bandwidth and CPU utilization a Kafka client can use.
* ACL configuration: defining the access rights of Kafka clients to read/write from/to topics.

For the purpuse of this tutorial, we will use a single node (non production) Zookeeper cluster. Kafka provides a convenient Zookeeper startup script `./bin/zookeeper-server-start.sh` with its minimalistic configuration file `./config/zookeeper.properties`:


```
> ./bin/zookeeper-server-start.sh config/zookeeper.properties
```

You should get the following output log. Zookeeper is now up and running listening on port number `2181`.

```
...
[2018-03-30 12:38:33,664] INFO Server environment:java.library.path=/usr/java/packages/lib/amd64:/usr/lib/x86_64-linux-gnu/jni:/lib/x86_64-linux-gnu:/usr/lib/x86_64-linux-gnu:/usr/lib/jni:/lib:/usr/lib (org.apache.zookeeper.server.ZooKeeperServer)
[2018-03-30 12:38:33,666] INFO Server environment:java.io.tmpdir=/tmp (org.apache.zookeeper.server.ZooKeeperServer)
[2018-03-30 12:38:33,666] INFO Server environment:java.compiler=<NA> (org.apache.zookeeper.server.ZooKeeperServer)
[2018-03-30 12:38:33,666] INFO Server environment:os.name=Linux (org.apache.zookeeper.server.ZooKeeperServer)
[2018-03-30 12:38:33,667] INFO Server environment:os.arch=amd64 (org.apache.zookeeper.server.ZooKeeperServer)
[2018-03-30 12:38:33,667] INFO Server environment:os.version=4.13.0-36-generic (org.apache.zookeeper.server.ZooKeeperServer)
[2018-03-30 12:38:33,667] INFO Server environment:user.name=bachwehbi (org.apache.zookeeper.server.ZooKeeperServer)
[2018-03-30 12:38:33,667] INFO Server environment:user.home=/home/bachwehbi (org.apache.zookeeper.server.ZooKeeperServer)
[2018-03-30 12:38:33,667] INFO Server environment:user.dir=/home/bachwehbi/dev/kafka (org.apache.zookeeper.server.ZooKeeperServer)
[2018-03-30 12:38:33,681] INFO tickTime set to 3000 (org.apache.zookeeper.server.ZooKeeperServer)
[2018-03-30 12:38:33,690] INFO minSessionTimeout set to -1 (org.apache.zookeeper.server.ZooKeeperServer)
[2018-03-30 12:38:33,690] INFO maxSessionTimeout set to -1 (org.apache.zookeeper.server.ZooKeeperServer)
[2018-03-30 12:38:33,717] INFO binding to port 0.0.0.0/0.0.0.0:2181 (org.apache.zookeeper.server.NIOServerCnxnFactory)

```

## Running Kafka

The Kafka package comes with a ready to use configuration file. Let's review the main points in this file:

### Kafka Configuration Options

#### Broker id

Every broker in a Kafka cluster must have a unique identifier.

```
############################# Server Basics #############################

# The id of the broker. This must be set to a unique integer for each broker.
broker.id=0

```

#### Broker Socket settings

The broker socket settings include the hostname and port number the broker listens to and the hostname and post it will advertise to producers and consumers. By default, Kafka listens on local hostname and port number 9092.

```
############################# Socket Server Settings #############################

# The address the socket server listens on. It will get the value returned from
# java.net.InetAddress.getCanonicalHostName() if not configured.
#   FORMAT:
#     listeners = listener_name://host_name:port
#   EXAMPLE:
#     listeners = PLAINTEXT://your.host.name:9092
#listeners=PLAINTEXT://:9092

# Hostname and port the broker will advertise to producers and consumers. If not set,
# it uses the value for "listeners" if configured.  Otherwise, it will use the value
# returned from java.net.InetAddress.getCanonicalHostName().
#advertised.listeners=PLAINTEXT://your.host.name:9092

```

#### Zookeeper settings

Kafka broker needs to connect to Zookeeper. The Zookeeper settings provide the connection string (one or multiple hostname:port) of Zookeeper cluster. By default, Kafka assumes a local Zooker is running on port number 2181.

```
############################# Zookeeper #############################

# Zookeeper connection string (see zookeeper docs for details).
# This is a comma separated host:port pairs, each corresponding to a zk
# server. e.g. "127.0.0.1:3000,127.0.0.1:3001,127.0.0.1:3002".
# You can also append an optional chroot string to the urls to specify the
# root directory for all kafka znodes.
zookeeper.connect=localhost:2181

# Timeout in ms for connecting to zookeeper
zookeeper.connection.timeout.ms=6000
```

### Run Kafka

You can now start the Kafka cluster with:

```
> bin/kafka-server-start.sh config/server.properties
```

You should get the following output log. A Kafka instance is up and running!

```
...
[2018-03-30 13:07:19,734] INFO Registered broker 0 at path /brokers/ids/0 with addresses: EndPoint(bachwehbi-VirtualBox,9092,ListenerName(PLAINTEXT),PLAINTEXT) (kafka.utils.ZkUtils)
[2018-03-30 13:07:19,736] WARN No meta.properties file under dir /tmp/kafka-logs/meta.properties (kafka.server.BrokerMetadataCheckpoint)
[2018-03-30 13:07:19,808] INFO Kafka version : 1.1.0 (org.apache.kafka.common.utils.AppInfoParser)
[2018-03-30 13:07:19,822] INFO Kafka commitId : c0518aa65f25317e (org.apache.kafka.common.utils.AppInfoParser)
[2018-03-30 13:07:19,830] INFO [KafkaServer id=0] started (kafka.server.KafkaServer)
```

## Working with topics

### Create a topic

Now that the Kafka broker is running, it's time to work with topics. Kafka provides a utilities script to work with topics `kafka-topics.sh`. To create a new topic named `test1` run the following:

```
> bin/kafka-topics.sh --create --zookeeper localhost:2181 \
  --replication-factor 1 --partitions 1 --topic test1
```

This creates a new topic named `test1` with 1 partition and without replication. With our single node Kafka cluster, it is possible to partition topics but it won't be possible to replicate data. The latter requires a multi-node cluster.

For a partitioned topic run the following:
```
> bin/kafka-topics.sh --create --zookeeper localhost:2181 \
  --replication-factor 1 --partitions 3 --topic my-partitioned-topic
```

Try now to set `replication-factor` to 3. We will receive an error indicating that the replication factor is larger than available brokers.

```
> bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 3 --partitions 13 --topic test3
Error while executing topic command : Replication factor: 3 larger than available brokers: 1.
[2018-03-30 15:37:24,227] ERROR org.apache.kafka.common.errors.InvalidReplicationFactorException: Replication factor: 3 larger than available brokers: 1.
 (kafka.admin.TopicCommand$)

```

### Listing existing topics

Existing topics can be listed by running the following command:
```
$ ./bin/kafka-topics.sh --list --zookeeper localhost:2181
test1
test2
```

### Describing a topic

We can alo describe a particular topic using:
```
$ bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic test1
    Topic:test1	PartitionCount:1	ReplicationFactor:1	Configs:
    Topic: test1	Partition: 0	Leader: 0	Replicas: 0	Isr: 0
```

### Auto-creating topics

It is possible to configure Kafka to auto-create topics when a non existent topic is published to. In production deployments, it is however not recommended to enable this feature as it might lead to the undesired creation of a large amount ot topics.

## Writing to Kafka

Kafka provides a command line producer utility to send messages out to a Kafka topic. The utility called `kafka-console-producer.sh` takes input from a file or from the standard input. You can use it as follows:

```
> ./bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test1
>message 1
>message 2
>message 3
```

Write a text message and press enter to publish it to the topic. To send multiple messages, you can repeat this process as many times as you wish.

## Consuming messages

Kafka also has a command line consumer utility that will subscribe to a topic dump out received messages to the standard output. Start the consumer as follows:

```
> bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test1 --from-beginning
message 1
message 2
message 3
```

When running the consumer, you will receive all the messages published to that topic. Re-run the same command and see how you will get the same messages over and over again. This is because of the `--from-beginning` option that instructs the consumer to request all data on the commit log of the topic.

Now go back to the producer console and publish additional messages and see how the consumer receives them in real time.

You can also specify the offset id from which to consume from. When specifying an offset, it is required to specify the partition as well as Kafka maintains an ordered offset per partition.

```
> bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 \
    --topic test1 --partition 0 --offset 2
message 3
message 4
message 5
```

You see that messages are delivered in the insertion order. This is because the topic `test1` is single partitioned. Let's now try to work with partitioned topic `my-partitioned-topic` we already created.

Now start a couple of additional consumers in separate terminals and go back to your producer to send few messages. See how messages are received by all consumers? This is because each consumer is in a different `consumer group`. We will work with consumer groups later in this tutorial.

## Working with partitioned topics

Publishing to a partitioned topic is exactly as publishing to a single partition topic.

```
> ./bin/kafka-console-producer.sh --broker-list localhost:9092 --topic my-partitioned-topic
>message 1
>message 2
>message 3
>message 4
>message 5
>message 6
```

Describing the topic shows its 3 non replicated partitions:

```
bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic my-partitioned-topic
Topic:my-partitioned-topic	PartitionCount:3	ReplicationFactor:1	Configs:
	Topic: my-partitioned-topic	Partition: 0	Leader: 0	Replicas: 0	Isr: 0
	Topic: my-partitioned-topic	Partition: 1	Leader: 0	Replicas: 0	Isr: 0
	Topic: my-partitioned-topic	Partition: 2	Leader: 0	Replicas: 0	Isr: 0
```

Now start a consumer console utility:
```
> bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic my-partitioned-topic --from-beginning
message 3
message 6
message 1
message 4
message 2
message 5
```

Messages are not delivered in the publishing order here. This is the normal behavior of Kafka as it guarantees order within every partition. To see this, let's read messages from the topic's 3 partitions:

```
> bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 \
     --topic my-partitioned-topic --partition 0 --offset 0
message 2
message 5

> bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 \
    --topic my-partitioned-topic --partition 1 --offset 0
message 1
message 4

> bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 \
    --topic my-partitioned-topic --partition 2 --offset 0
message 3
message 6
```

See how messages were delivered in order from every partition. The key point here is to remember that Kafka orders messages per partition and not per topic.

## Working with consumer groups

### Consumer group utility

In what we have seen so far, consumers were started without explicitly indicating there group identifier. In this case, the consumer self assigns a (randomly) chosen group id where it is the only consumer.

Kafka provides a utility to work with consumer groups `kafka-consumer-groups.sh`. Let's list the groups known by our cluster:

```
> ./bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list
Note: This will not show information about old Zookeeper-based consumers.

console-consumer-16911
console-consumer-83013
console-consumer-43569
console-consumer-77790
console-consumer-20314
...
```

We can see the different consumer groups that were automatically created by our previous consumers. We can also check the details of a particular group to see consumer positions per partition:

```
> bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group console-consumer-20314
Note: This will not show information about old Zookeeper-based consumers.


TOPIC                          PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG        CONSUMER-ID                                       HOST                           CLIENT-ID
my-partitioned-topic           0          2               2               0          consumer-1-2fb58ed7-81b6-4c8c-9a28-6638f44c7070   /127.0.0.1                     consumer-1
my-partitioned-topic           1          2               2               0          consumer-1-2fb58ed7-81b6-4c8c-9a28-6638f44c7070   /127.0.0.1                     consumer-1
my-partitioned-topic           2          2               2               0          consumer-1-2fb58ed7-81b6-4c8c-9a28-6638f44c7070   /127.0.0.1                     consumer-1
```

We can also use the consumer group utility to check members in a particular consumer group:

```
> bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group console-consumer-20314 --members
 
CONSUMER-ID                                    HOST            CLIENT-ID       #PARTITIONS
consumer-1-2fb58ed7-81b6-4c8c-9a28-6638f44c7070 /127.0.0.1     consumer-1      3
```

### Consuming messages in a consumer group

To start a consumer in a consumer group, we need just to add an option `--consumer-property group.id=GROUP_NAME` before running the cosumer.

Open 3 terminal windows and run the following command:

```
> bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic my-partitioned-topic --from-beginning --consumer-property group.id=group1
```

Go back to the producer terminal and publish few messages. See how these messages are delivered evenly between the different consumers.
Kafka requires every topic partition messages to be delivered to a single consumer in a consumer group.

Describe the consumer group to see how this is reflected:

```
> bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group group1
Note: This will not show information about old Zookeeper-based consumers.


TOPIC                          PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG        CONSUMER-ID                                       HOST                           CLIENT-ID
my-partitioned-topic           0          4               4               0          consumer-1-0af5a5ca-72e5-4328-b68b-e0949d07c2b9   /127.0.0.1                     consumer-1
my-partitioned-topic           2          4               4               0          consumer-1-d60f1be2-b03b-4e1d-91ef-418588e530f2   /127.0.0.1                     consumer-1
my-partitioned-topic           1          4               4               0          consumer-1-2ff1f615-28fd-49cf-9ade-e8393c687573   /127.0.0.1                     consumer-1

```

As our topic has 3 partitions, what would happen if we start a 4th consumer in the group?

Nothing, see :)

```
> bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group group1
Note: This will not show information about old Zookeeper-based consumers.


TOPIC                          PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG        CONSUMER-ID                                       HOST                           CLIENT-ID
my-partitioned-topic           0          4               4               0          consumer-1-0af5a5ca-72e5-4328-b68b-e0949d07c2b9   /127.0.0.1                     consumer-1
my-partitioned-topic           2          4               4               0          consumer-1-d60f1be2-b03b-4e1d-91ef-418588e530f2   /127.0.0.1                     consumer-1
my-partitioned-topic           1          4               4               0          consumer-1-2ff1f615-28fd-49cf-9ade-e8393c687573   /127.0.0.1                     consumer-1
-                              -          -               -               -          consumer-1-d60f1be2-b03b-4e1d-91ef-418588e530f2   /127.0.0.1                     consumer-1

```

the 4th consumer will not receive any messages as there are more consumers in the group than partitions in the topic. However, this consumer will be ready to take over in case of consumer failure.

In the next section, we will try to understand the failover strategies of Kafka.

## Kafka Fault Tolerence

### Running a multi-broker cluster

So far we have worked with a single broker Kafka cluster. Although this is fine for dev and debug purposes, it is important to have a proper multi node cluster for production.
A Kafka cluster is composed of multiple brokers running on different phisycal or virtual machines and potentially across multiple data centers.
For the purpose our this tutorial, we will run a kafka cluster composed of 3 brokers running on the same machine. Every broker will be listenning to a different port number.

Every broker needs its own configuration file, so let's start by duplicating the configuration file of our existing broker:

```
cp config/server.properties config/server1.properties
cp config/server.properties config/server2.properties
```

Open the newly created configuration files in your favorite text editor, locate the lines corresponding to: `broker.id`, `port`, and, `log.dirs` properties and change them as follows:

```
config/server1.properties:
    ...
    broker.id=1
    listeners=PLAINTEXT://:9093
    log.dir=/tmp/kafka-logs1
    ...
 
config/server-2.properties:
    ...
    broker.id=2
    listeners=PLAINTEXT://:9094
    log.dir=/tmp/kafka-logs2
```

Now open two terminal windows and start two additional Kafka brokers with these configuration files:

*Start second broker in a new terminal window*
```
> bin/kafka-server-start.sh config/server1.properties
```

*Start the third broker in a another terminal window*
```
> bin/kafka-server-start.sh config/server2.properties
```

Great! Now you have a Kafka cluster with 3 brokers connected to a single node Zookeeper cluster.

In the next two sections, we will see Kafka failover in action.

### Create a replicated topic

In the previous sections of this tutorial, we created a single partitionaed and a multi partitions topics. There were no replication as we had only one broker.
Let's create now a replicated topic. With replication, Kafka achieves data durability and tolerence to broker failures.

```
> bin/kafka-topics.sh --create --zookeeper localhost:2181 \
  --replication-factor 3 --partitions 3 --topic my-replicated-topic
```

This creates a topic named `my-replicated-topic` with three partitions replicated over three brokers each. As our cluster has 3 brokers, this would work just fine.

Let's `describe` the topic and see what Kafka tells us about it:
```
$ bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic my-replicated-topic
Topic:my-replicated-topic	PartitionCount:3	ReplicationFactor:3	Configs:
	Topic: my-replicated-topic	Partition: 0	Leader: 1	Replicas: 1,0,2	Isr: 1,0,2
	Topic: my-replicated-topic	Partition: 1	Leader: 2	Replicas: 2,1,0	Isr: 2,1,0
	Topic: my-replicated-topic	Partition: 2	Leader: 0	Replicas: 0,2,1	Isr: 0,2,1
```

Kafka has distributed the responsibilities evenly between the different brokers. Every topic partition has its own leader (broker) and a list of replicas and an indication if they are in-sync with the leader (the `Isr` list).

### Consumer failure

We will now start 3 consumers in the same consumer group. Open 3 terminal windows and run the following command:

```
> bin/kafka-console-consumer.sh --bootstrap-server localhost:9092,localhost:9093 --topic my-replicated-topic --from-beginning --consumer-property group.id=rep-group
```

Once the three consumers are running, we will use the console producer utility to send some messages to the topic:

```
> ./bin/kafka-console-producer.sh --broker-list localhost:9092,localhost:9093 --topic my-replicated-topic
>message 1
>message 2
>message 3
>message 4
>message 5
>message 6
```
See how the consumers get a share of the messages. Every message is received by one of the consumers in the group.

```
> bin/kafka-console-consumer.sh --bootstrap-server localhost:9092,localhost:9093 --topic my-replicated-topic --from-beginning --consumer-property group.id=rep-group
message 1
message 4

> bin/kafka-console-consumer.sh --bootstrap-server localhost:9092,localhost:9093 --topic my-replicated-topic --from-beginning --consumer-property group.id=rep-group
message 2
message 5

> bin/kafka-console-consumer.sh --bootstrap-server localhost:9092,localhost:9093 --topic my-replicated-topic --from-beginning --consumer-property group.id=rep-group
message 3
message 6
```

The good thing with consumer groups is that not only load is distributed among the consumers, it also gets reconfigured when a consumer fails or leaves the group.

To show this, stop one of the consumers and send 6 additional messages (*message 7* - *message 12*) from the producer.

```
> bin/kafka-console-consumer.sh --bootstrap-server localhost:9092,localhost:9093 --topic my-replicated-topic --from-beginning --consumer-property group.id=rep-group
message 1
message 4
message 7
message 9
message 10
message 12

> bin/kafka-console-consumer.sh --bootstrap-server localhost:9092,localhost:9093 --topic my-replicated-topic --from-beginning --consumer-property group.id=rep-group
message 2
message 5
message 8
message 11

> bin/kafka-console-consumer.sh --bootstrap-server localhost:9092,localhost:9093 --topic my-replicated-topic --from-beginning --consumer-property group.id=rep-group
message 3
message 6
^CProcessed a total of 2 messages
```

When one of the consumers was stopped, Kafka automatically redistributed the topic partitions among the remaining consumers. As the topic has 3 partitions and only 2 consumers were left in the group, one of the consumers was attributed 2 partitions while the other was attributed the third one.

This is how Kafka consumer group fault tolerence works. The Kafka cluster will reconfigure the partition distribution among the consumers in the group with every consumer join or leave.

### Broker Failure

Let's now simulate a broker failure by stopping on of the brokers in the cluster.

Open one of the broker terminals and hit `CTRL-C`, this will stop the broker.

Once the broker has stopped, describe the topic:

```
$ bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic my-replicated-topic
Topic:my-replicated-topic	PartitionCount:3	ReplicationFactor:3	Configs:
	Topic: my-replicated-topic	Partition: 0	Leader: 0	Replicas: 1,0,2	Isr: 0,2
	Topic: my-replicated-topic	Partition: 1	Leader: 2	Replicas: 2,1,0	Isr: 0,2
	Topic: my-replicated-topic	Partition: 2	Leader: 0	Replicas: 0,2,1	Isr: 0,2
```

See how the leaders for the topic partitions have been reconfigured to account for the failed broker. Similarly for the in-sync list.

Now to show that the cluster continues to work normally in the presence of broker failure, go back to the producer terminal and send a bucnh of messages. 

See how all published messages have been delivered to the active consumers? Voilà :)

## Working with Kafka Connect

# References
https://www.quora.com/What-is-the-actual-role-of-Zookeeper-in-Kafka-What-benefits-will-I-miss-out-on-if-I-don%E2%80%99t-use-Zookeeper-and-Kafka-together
