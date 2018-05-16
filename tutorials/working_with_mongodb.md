# Working with MongoDB

## Installation

### Installing MongoDB community edition on windows

MongoDB Community Edition requires Windows Server 2008 R2, Windows 7, or later.

Download the latest release of MongoDB from https://www.mongodb.org/downloads.
The name of the extracted file should be mongodb-win32-i386-[version] or mongodb-win32-x86_64-[version]

Double-click the .msi downloaded file, a set of screens will appear to guide you
through the installation process. Choose “Custom” installation to specify an
installation directory e.g. d:\mongo

#### Run MongoDB

MongoDB requires a data directory to store all data. The default location for the
MongoDB data directory is c:\data\db. Create this folder by running the following
command in a Command Prompt:

```
md data\db
```
In order to install the MongoDB at a different location, you need to specify an
alternate path for \data\db by setting the path dbpath in mongod.exe as following
(Supposing that the mongodb installation folder is D:\mongo)

```
C:\>d:
D:\>cd "mongo\Mongodb\bin"
D:\mongo\Mongodb\bin>mongod.exe --dbpath "d:\mongo\data"
```
Now to run the MongoDB, you need to open another command prompt and run the following command:

```
D:\mongo\Mongodb\bin>mongo.exe
```

### Installing mongodb on Ubuntu

Run the following command to import the MongoDB public GPG key
```
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 2930ADAE8CAF5059EE73BB4B58712A2291FA4AD5
```

Create a /etc/apt/sources.list.d/mongodb.list file using the following command:
```
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 multiverse" \
  | sudo tee /etc/apt/sources.list.d/mongodb-org-3.6.list
```

Now issue the following command to update the repository:

```
sudo apt-get update
```

Next install the MongoDB by using the following command:

```
sudo apt-get install -y mongodb-org
```

In the above installation, 3.6.4 is currently released MongoDB version.
Make sure to install the latest version always.

#### Start MongoDB

To start the `mongod` service, run the following command:
```
sudo service mongod start
```

To use MongoDB client console run the following command:

```
mongo
```

This will connect you to running MongoDB instance.

#### Stop MongoDB

Use the following command in order to stop mongodb:

```
sudo service mongod stop
```

#### Restart MongoDB

Use the following command in order to restart mongodb:

```
sudo service mongod restart
```
## Create database

In this tutorial, we will create a database to store our application logs.
Use the following command to create a new database named `Logging`.

You need first to open MongoDB client console:

```
mongo
```

`use` command will create a new database if it doesn't exist, otherwise it will
return the existing database.

```
use Logging
```

To check the currently selected database, use the command `db`

```
db
```

To check the databases list, use the command `show dbs`

```
show dbs
```

See how the newly created database will not be listed. It is because it contains
nothing so far. Try to run this command again when you create a collection in the database.

## Drop Database

Use the following command to delete the selected database

```
## create a dummy database
use db_to_delete

## delete the database
db.dropDatabase()
```

## Create Collection

Now let’s create a collection named logs within the logging database using the command below:

```
## lets go back to use Logging database
use Logging

## create a collection in the database
db.createCollection('logs')
```

You can check the created collection by using the command `show collections`:

```
show collections
```

## Data manipulation

### Insert document

Use the following command to insert a document in the logs collection

```
db.logs.insert(
{
	Ip_address: "::ffff:54.221.205.80",
	Client_id:"client1",
	User_id:"user1",
	Date_time:new Date("2018-03-1T23:07:43"),
	Method:"POST",
	Endpoint:"/v1/data/publish/ISS/position",
	Protocol:"HTTP/1.1",
   Response_code: 200,
   Content_size:4
})
```

### Insert bulk documents

Use the following command in order to insert 2 documents to the logs collection:

```
db.logs.insert([
{
	Ip_address:"::ffff:54.243.49.1",
	Client_id:"client1",
	User_id:"user1",
	Date_time:new Date("2018-05-3T11:08:11"),
	Method:"POST",
	Endpoint: "/v1/data/write/klima/Temperature",
	Protocol:"HTTP/1.1",
	Response_code: 200,
	Content_size:4
},
{
	Ip_address:"::ffff:54.221.205.80",
	Client_id:"client2",
	User_id:"user2",
	Date_time:new Date("2018-03-2T11:00:21"),
	Method:"GET",
	Endpoint: "/v1/data/read/appliance/command?limit=1&source=raw",
	Protocol:"HTTP/1.1",
	Response_code: 404,
	Content_size:5
}]
)
```

### Query document

Use the following command to get all the documents in a given collection:

```
db.logs.find().pretty()
```

You can see in the query results the `_id` parameter corresponding to unique id
MongoDB created for every inserted document.

Now use the following command to get all documents with response code 200:

```
db.logs.find({"Response_code":200}).pretty()
```

Use the following command to get all the documents where response code = 200 and Client_id = client1:

```
db.logs.find(
   {
      $and: [
         {Response_code : 200}, {Client_id: "client1"}
      ]
   }
).pretty()
```

To query the document on the basis of some condition, you can use following operations:

```
Equality: 		        {<key>:<value>}
Less Than: 		        {<key>:{$lt:<value>}}
Less Than Equals: 	  {<key>:{$lte:<value>}}
Greater : 		        {<key>:{$gt:<value>}}
Greater Than Equals: 	{<key>:{$gte:<value>}}
Not Equals: 		      {<key>:{$ne:<value>}}
```

Use these conditions to create different filters on the data.

Now, use the following command to get the top document having the biggest content size:

```
db.logs.find().sort({"Content_size":-1}).limit(1).pretty()
```

Please note that, if you don't specify the sorting preference, then sort()
method will display the documents in `ascending` order.

The following command retrieves the logs where method equals either GET or POST

```
db.logs.find({Method: {$in: ["GET", "POST"]}}).pretty()
```

The following command retrieves the logs where method equals GET and Content_size is greater than or equal 4

```
db.logs.find({Method:"GET", Content_size: {$gte:4}}).pretty()
```

The following command retrieves the logs where Method equals "POST" and either Content_size = 4 or Client_id starts with "Client"

```
db.logs.find( {
     Method: "GET",
     $or: [ { Content_size: 4 }, { Client_id: /^client/ } ]
}).pretty()
```

## Indexing
Indexes support the efficient resolution of queries. Without indexes, MongoDB must
scan the entire document space of a collection to select those documents that
match the query statement. This scan is highly inefficient and requires MongoDB
to process a large volume of data.

Indexes are special data structures, that store a small portion of the data set
in an easy-to-traverse form. The index stores the value of a specific field or
set of fields, ordered by the value of the field as specified in the index.

When we create a collection in MongoDB then MongoDB creates a unique Index on `_id`
Field, this is called the Default index.

MongoDB supports multiple types of indexes like Single Filed Index, Compound Index,
Multikey Index, Text Index, Geospatial Index and Hashed Index.

In the following exercise, we will learn how to create a Single Field Index and a Compound Index.

### Create Single Field index

Let’s run the following query:

```
db.logs.find({"Response_code": 404}).explain("executionStats")
```
Note that `totalDocsExamined` value is equal to the number of all the documents
in the collection. This means that MongoDB scanned all the documents in order to
return the result. This is shown in `"stage" : "COLLSCAN"` which means collection
scan.

Let’s add an index on `Response_code`

```
db.logs.createIndex({"Response_code":1})
```

To create index in descending order you need to use -1:
```
db.logs.createIndex({"Response_code":-1})
```

Now let’s run the same query again

```
db.logs.find({"Response_code": 404}).explain("executionStats")
```
After Index, MongoDB will not do a complete collection scan, the `totalDocsExamined`
will show now the number of the logs where Response_code is 404 instead of the total
number of documents. The index allowed the query planner to use the `"stage" : "FETCH"`
which means simple fetching instead of collection scan. If we dig deeper in the
output of the query `explain` command, we will see the input stage using `IXSCAN`.
This means the database first made an index scan to identify corresponding documents,
then retrieved these documents with a simple fetch.

### Create Compound Index

Now suppose that we want to search based on both Client_id and Method.
In that case we will have to apply index on Client_id and Method, this is called
Compound Index:

```
db.logs.createIndex({"Client_id":1, "Method": 1})
```

You can use `stats()` function to get interesting statistics about databases and collections:

```
## database statistics
db.stats()

## collection statistics
db.logs.stats()
```

## Aggregation

Aggregations are used to process data records and return computed results.
Aggregation operations group values from multiple documents together, and can
perform a variety of operations on the grouped data to return a single result.

Use the following command to get the count of logs and total content size of each IP address:

```
db.logs.aggregate([{$group : {_id : "$Ip_address", count: {$sum : 1}, total_size: {$sum : "$Content_size"}}}])
```

Use the following command to get the count of logs for each endpoint:

```
db.logs.aggregate([{$group : {_id : "$Endpoint", count: {$sum : 1}}}])
```

Use the following command to get the count and maximum content size of each method:

```
db.logs.aggregate([{$group : {_id : "$Method", count: {$sum : 1}, max_size: {$max : "$Content_size"}}}])
```

Use the following command to get the count of each response code

```
db.logs.aggregate([{$group : {_id : "$Response_code", count: {$sum : 1}}}])
```

## Replication

In this exercise, we will use the same host machine to create a Replica Set with 2 nodes.
For this purpose, we will run 2 `mongod` instances. One instance acts as `PRIMARY` and the other instance acts as `SECONDARY`.

Shutdown already running MongoDB server instances.

Create two folders `data1` and `data2` in your home directory to host data for
the replication set.

Now start the MongoDB server by specifying the `--replSet` option as per the below:

```
mongod --port 27017 --dbpath <path_to_data1> --replSet my_replication_set
```

this will start a mongod instance on port 27017. `--replset my_replication_set`
means that we are setting up a set of nodes to replicate, and the name given to this Replica set is `my_replication_set`.

### Start another mongod instance

In order to run another `mongod` instance on the same machine, we should use a
different port and data path. In this exercise we will use port 27018 and path <path_to_data2> we already created.

Run the following command to run a new mongod instance

```
mongod --port 27018 --dbpath <path_to_data2 --replSet my_replication_set
```

Note that this instance is also started with same Replica Set

### Start Replication

After starting the 2 mongod instances, open MongoDB client console and run the
command `rs.initiate()` to initiate a new replica set.

```
rs.initiate()
```
You can check the status using `rs.status()` command.

### Add the second mongodb instance  to the Replica Set
To add the second MongoDB instance that we already started, run the following
command to add the second mongo instance to the replica set:

```
rs.add("localhost:27018")
```

If you get a response { “ok” : 1 }, this means that the addition of the mongod
instance to the replica set is successful.

You can check the status by running the following command:
```
rs.status()
```

### Check Replication

Now we will check if the replication is working correctly. We will insert a document to the primary instance:

```
use Logging

db.logs.insert(
{
	Ip_address: "::ffff:54.221.200.10",
	Client_id:"client5",
	User_id:"user5",
	Date_time:new Date("2018-03-03T11:00:00"),
	Method:"GET",
	Endpoint: "/v1/data/read/temp",
	Protocol:"HTTP/1.1",
Response_code: 200,
Content_size:4
}
)
````

Now let’s check in the SECONDARY instance, if this document is replicated.
In a new terminal window, start a new mongo client console and connect it to the
secondary database using the port number parameter:

```
mongo --port 27018

## Allow read operations on secondary
rs.slaveOk()

## Connect to Logging database
use Logging

## read documents from logs collections
db.logs.find()
```

As you can see, replication is up and running!
