# Working with File Formats

In this tutorial, we will work with different file formats.

## Parquet Tools

### Install parquet-tools

`parquet-tools` includes a set of utilities to work with and to debug Parquet files.

It is part of the parquet project repository on Github. You can install it and build it as follows:

```
  git clone https://github.com/apache/parquet-mr.git
  cd parquet-mr

  # checkout latest version 1.10.1
  git checkout tags/apache-parquet-1.10.1

  cd parquet-tools
  ## build the project
  mvn clean package -Plocal
```

The build process will create `parquet-tools` jar files under `target` folder. We will be using `parquet-tools-1.10.1.jar` in the following.

Building Parquet tools requires `maven`. If you don't have maven on your machine, install it with:

```
  sudo apt-get -y install maven
```

Alternatively, you can get `parquet-tools` jar file from [maven central](https://mvnrepository.com/artifact/org.apache.parquet/parquet-tools/1.10.1). 
The jar files for the latest version can be found [here](http://central.maven.org/maven2/org/apache/parquet/parquet-tools/1.10.1/).

### Working with parquet-tools

**Print help**

You can get parquet-tools help with:

```
java -jar parquet-tools-1.10.1.jar -h
```

**Print file schema**

You can print the Schema of a Parquet file using:

```
java -jar parquet-tools-1.10.1.jar schema <path_to_data>/weblog.parquet 
message spark_schema {
  optional binary client_id (UTF8);
  optional int64 content_size;
  optional binary date_time (UTF8);
  optional binary endpoint (UTF8);
  optional binary ip_address (UTF8);
  optional binary method (UTF8);
  optional binary protocol (UTF8);
  optional int64 response_code;
  optional binary user_id (UTF8);
}

```

What `optional` in front of column type definition means?

As you most probably thought, it is to indicate that the column in nullable. That means, the column accepts Null values.

**Check file content**

You can print the first *n* records in the file using:

```
$ java -jar parquet-tools-1.10.1.jar head -n 20 <path_to_data>/weblog.parquet 

client_id = -
content_size = 4
date_time = 01/Mar/2018:23:07:31 +0000
endpoint = /v1/data/write/Ras_Beirut/statusRas_Beirut4
ip_address = ::ffff:54.243.49.1
method = POST
protocol = HTTP/1.1
response_code = 200
user_id = -

....
```

You can print the full data (no metadata) content of the file using:

```
$ java -jar parquet-tools-1.10.1.jar cat <path_to_data>/weblog.parquet 

client_id = -
content_size = 4
date_time = 01/Mar/2018:23:07:31 +0000
endpoint = /v1/data/write/Ras_Beirut/statusRas_Beirut4
ip_address = ::ffff:54.243.49.1
method = POST
protocol = HTTP/1.1
response_code = 200
user_id = -

....
```

If you want to print the content in JSON format use:

```
$ java -jar parquet-tools-1.10.1.jar cat -j <path_to_data>/weblog.parquet 
....
```

**Print file metadata**

Parquet format adds per column statistics per row group. The check these statistics along with other metadata:

```
java -jar parquet-tools-1.10.1.jar meta <path_to_data>/weblog.parquet 
```

More detailed metadata information can be obtained using:

```
java -jar parquet-tools-1.10.1.jar dump -d <path_to_data>/weblog.parquet 
```

Take your time to inspect the output of the last command!

You can print row count and total size of the file using:

```
java -jar parquet-tools-1.10.1.jar rowcount <path_to_data>/weblog.parquet 

java -jar parquet-tools-1.10.1.jar size <path_to_data>/weblog.parquet
```

## ORC Tools

### Install orc-tools

`orc-tools` includes a set of utilities to work with and to debug ORC files.

It is part of the ORC project repository on Github. You can install it and build it from sources or alternatively,
you can get `orc-tools` jar file from [maven central](https://mvnrepository.com/artifact/org.apache.orc/orc-tools/1.5.5).
The jar files for the latest version can be found [here](http://repo1.maven.org/maven2/org/apache/orc/orc-tools/1.5.5/).

### Working with orc-tools

**Print help**

You can get orc-tools help with:

```
java -jar orc-tools-1.5.5-uber.jar -h
```

**Print file Schema**

See *print file metadata* below.

**Check file content**

You can print the ORC file content in JSON format to the standard output using:

```
java -jar orc-tools-1.5.5-uber.jar data <path_to_data>/weblog.orc
```

**Scan file**

You can scan an ORC file to check the ORC batches number and row count using:

```
java -jar orc-tools-1.5.5-uber.jar scan <path_to_data>/weblog.orc
```

**Print file metadata**

ORC file metadata including the Schema, column stats, and stripe information can be printed using:

```
java -jar orc-tools-1.5.5-uber.jar meta <path_to_data>/weblog.orc
```

Take your time to inspect the output of the last command and compare it to the metadata of parquet file.
Where in your opinion ORC is behaving better than parquet?

## Avro Tools

### Install avro-tools

`avro-tools` includes a set of utilities to work with and to debug Avro files.

It is part of the Avro project repository on Github. You can install it and build it from sources or alternatively,
you can get `avro-tools` jar file from [maven central](https://mvnrepository.com/artifact/org.apache.avro/avro-tools/1.8.2). 
The jar files for the latest version can be found [here](http://central.maven.org/maven2/org/apache/avro/avro-tools/1.8.2/).

### Working with avro-tools

**Print help**

You can get avro-tools help with:

```
java -jar avro-tools-1.8.2.jar -h
```

**Print file Schema**

To print the file schema of an Avro file run:

```
java -jar avro-tools-1.8.2.jar getschema <path_to_data>/weblog.avro
```

**Check file content**

You can print an extract of the Avro file content to the standard output using:

```
java -jar avro-tools-1.8.2.jar cat --limit 10 <path_to_data>/weblog.avro -
```

Avro tools provides a utility to convert Avro files to JSON as follows:

```
java -jar avro-tools-1.8.2.jar tojson <path_to_data>/weblog.avro

```
**Print file metadata**

Avro file metadata can be printed using:

```
java -jar avro-tools-1.8.2.jar getmeta <path_to_data>/weblog.avro
```

Avro is a row based file format. The metadata is relatively very limited compared to more optimized columnar based formats.

## File Size

Compare the file size of the different formats. See how the sizes are vary significantly.

What do you think about the file size of CSV and Avro formats?

What do you think about the file size of JSON and CSV formats?

What do you think about the file size of Parquet and ORC formats?
