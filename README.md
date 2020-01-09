# Goals

We want to access data in Spark from a S3 bucket. S3 can be [provided by a cloud provider](https://aws.amazon.com/en/s3/) or we can create our own, private S3 cluster using [min.io](https://docs.min.io/docs/minio-quickstart-guide.html).

Using S3 is an interesting alternative to [HDFS](https://hadoop.apache.org/docs/r1.2.1/hdfs_design.html) as its a cloud native storage system.

In short, we'll be able to read from S3 in Spark

```
val parquetFileDF = spark.read.parquet("s3a://my-bucket/some.parquet")
```

We can also write data to an S3 bucket

```spark.range(42).toDF.write.parquet("s3a://my-bucket/range.parquet")```

And finally we will be able to save data to a Hive table and Hive will save the data as Parquet on S3. The data can of course also be read via Hive. **Note:** following example of writing and selecting data requires proper configuration of  ```spark.sql.warehouse.dir``` so that it's located on S3 (see example configuration below).

```
spark.range(42).toDF.write.saveAsTable("test")

sql("select * from test").show
```

# Setting up min.io

Min.io can be run for testing purposes with minimal setup effort. It's basically a one liner. So easy that using Docker might be considered overhead ;) There's an excellent [guide](https://docs.min.io/docs/minio-quickstart-guide.html) explaining how to get started.

Assuming min.io is installed natively, we just start the server

```minio server /tmp/minio```

## Creating a bucket in min.io

Browse to min.io Web UI (e.g. http://127.0.0.1:9000) and create a bucket. 

You can also inspect the files created by Spark using the Web UI.

# Preparing the Spark Installation

Following steps show how the Spark installation can be prepared so Spark (incl. Spark shell) can access S3 URIs. 

## Installing dependencies

Additional JARs need to be placed in ```$SPARK_HOME/jars```:

* ```hadoop-aws```
* ```aws-java-sdk```

We need two things/jars: Support for the S3 filesystem (```hadoop-aws```) and support for the S3 client (```aws-java-sdk```).

 You need to select the ```hadoop-aws``` version matching the Hadoop version of your Spark installation. The JAR can be found on [Maven Central](https://mvnrepository.com/artifact/org.apache.hadoop/hadoop-aws/).

```
wget https://repo1.maven.org/maven2/org/apache/hadoop/hadoop-aws/2.7.3/hadoop-aws-2.7.3.jar
wget https://repo1.maven.org/maven2/com/amazonaws/aws-java-sdk/1.7.4/aws-java-sdk-1.7.4.jar
```

## Configuring S3 credentials

S3 credentials need to be configured in ```$SPARK_HOME/conf/spark-defaults.conf```

Access keys and endpoint might need to be adjusted.

```
spark.hadoop.fs.s3a.access.key minioadmin
spark.hadoop.fs.s3a.secret.key minioadmin
spark.hadoop.fs.s3a.endpoint http://127.0.0.1:9000
spark.hadoop.fs.s3a.path.style.access True
spark.hadoop.fs.s3a.impl org.apache.hadoop.fs.s3a.S3AFileSystem
```

## Enabling Hive and configuring for S3

```spark.sql.warehouse.dir``` can be set in ```$SPARK_HOME/conf/spark-defaults.conf``` so it points to a directory in a S3 bucket. That way, Hive tables are read from S3 and written to S3.

```
spark.sql.warehouse.dir s3a://my-bucket/hive/warehouse
spark.sql.catalogImplementation hive
```

## Enabling in-memory Hive Metastore

Following properties in ```$SPARK_HOME/conf/spark-defaults.conf``` are optional, can be used for local testing when you don't have a persistent Metastore.

```
spark.hadoop.javax.jdo.option.ConnectionDriverName org.apache.derby.jdbc.EmbeddedDriver
spark.hadoop.javax.jdo.option.ConnectionURL jdbc:derby:memory:myInMemDB;create=true
spark.hadoop.javax.jdo.option.ConnectionUserName hiveuser
spark.hadoop.javax.jdo.option.ConnectionPassword hivepass
spark.hadoop.datanucleus.autoCreateSchema true
spark.hadoop.datanucleus.autoCreateTables true
spark.hadoop.datanucleus.fixedDatastore false
```

