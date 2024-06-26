# Spark data sources and sinks
## Spark Data Sources and Sinks Intro
External data sources. External of your data lake, such as Oracle, SQL Server, application logs, MongoDB, Snowflake, Kafka, etc. 

You cannot process the data from external data sources directly using Spark. You could do either of these:
1. Bring the data to the data lake, and store them in your lake's distributed storage, using data integration tool. 
2. Directly connect to these external systems, using the Spark Data Source API. 

The instructor of the course prefers to use the 1st method for all their batch processing requirements, and the 2nd method for all their stream processing requirements. 

Bring data correctly and efficiently to the data lake it a complex goal by itself. Decoupling the ingestion from the processing could improve the manageability. Also, the capacity of your source system could have been planned accordingly, because it might have been designed for some specific purpose, other than this. 

Spark is an excellent tool for data processing, but it wasn't designed to handle the complexities of data ingestion. So, most of the well-designed real life projects do not directly connect to the external systems, even if we can do it. 

Internal data sources. It is your distributed storage, like HDFS, or cloud-based storage. The mechanics of reading data from these two are the same. The difference lies in the data file formats (csv, json, parquet, avro, plain text). Two other options are Spark SQL tables and Delta Lake, both are also backed by data files, but they have additional metadata stored outside of the data file. 

Data sink. The final destination of the processed data that you write to. Could be internal (a data file in your data lake storage, etc) or external (JDBC db, No sql db, etc). Spark allows you to write data in a variety of file formats, sql tables, and delta lake. 

Spark also allow you to directly write the data to many external sources, such as JDBC databases, Cassandra, MongoDB. Thought it is not recommended to do so, same reason as we don't read from these systems. 

## Spark DataFrameReader API
The "mode" option specifies behaviors when encountering a malformed record. The modes are:
- PERMISSIVE. Default. Set all the fields to null for the corrupt record, and put it in a string col named "_corrupt_record"
- DROPMALFORMED. Drop the corrupt record. 
- FAILFAST. Raises an exception, and terminates immediately, up on a malformed record. 

The "schema" is optional in many cases. Sometimes you can infer the schema. Some data formats came with a well-defined schema. 

Recommend to avoid using shortcuts such as "csv()" methods. Using the standard style add to the code maintainability. 

## Reading CSV, JSON and Parquet files
"SparkSchemaDemo.py":
```py
from pyspark.sql import SparkSession
from pyspark.sql.types import StructType, StructField, DateType, StringType, IntegerType
from lib.logger import Log4j

if __name__ == "__main__":
    spark = SparkSession \
        .builder \
        .master("local[3]") \
        .appName("SparkSchemaDemo") \
        .getOrCreate()

    logger = Log4j(spark)

    # define schema programmatically. complex. Must use spark data types
    # syntax: StructField(col_name, data_type)
    flightSchemaStruct = StructType([
        StructField("FL_DATE", DateType()),
        StructField("OP_CARRIER", StringType()),
        StructField("OP_CARRIER_FL_NUM", IntegerType()),
        StructField("ORIGIN", StringType()),
        StructField("ORIGIN_CITY_NAME", StringType()),
        StructField("DEST", StringType()),
        StructField("DEST_CITY_NAME", StringType()),
        StructField("CRS_DEP_TIME", IntegerType()),
        StructField("DEP_TIME", IntegerType()),
        StructField("WHEELS_ON", IntegerType()),
        StructField("TAXI_IN", IntegerType()),
        StructField("CRS_ARR_TIME", IntegerType()),
        StructField("ARR_TIME", IntegerType()),
        StructField("CANCELLED", IntegerType()),
        StructField("DISTANCE", IntegerType())
    ])

    # if use no schema option, all col will be str data type. 
    # if use the inferSchema option, they dates got inferred as string, 
    # but the numbers a inferred as int as expected. 
    # 
    flightTimeCsvDF = spark.read \
        .format("csv") \
        .option("header", "true") \
        .schema(flightSchemaStruct) \
        .option("mode", "FAILFAST") \
        .option("dateFormat", "M/d/y") \
        .load("data/flight*.csv") # you can use wild card here

    flightTimeCsvDF.show(5)
    logger.info("CSV Schema:" + flightTimeCsvDF.schema.simpleString())

    # the simpler/easier way to define schema
    # col_name and data_type, separated by comma. 
    flightSchemaDDL = """FL_DATE DATE, OP_CARRIER STRING, OP_CARRIER_FL_NUM INT, ORIGIN STRING, 
          ORIGIN_CITY_NAME STRING, DEST STRING, DEST_CITY_NAME STRING, CRS_DEP_TIME INT, DEP_TIME INT, 
          WHEELS_ON INT, TAXI_IN INT, CRS_ARR_TIME INT, ARR_TIME INT, CANCELLED INT, DISTANCE INT"""

    # json format doesn't have infer schema, as it already try to do it. 
    # but without doing anything specific, it read dates as strings. 
    flightTimeJsonDF = spark.read \
        .format("json") \
        .schema(flightSchemaDDL) \
        .option("dateFormat", "M/d/y") \
        .load("data/flight*.json")

    flightTimeJsonDF.show(5)
    logger.info("JSON Schema:" + flightTimeJsonDF.schema.simpleString())

    flightTimeParquetDF = spark.read \
        .format("parquet") \
        .load("data/flight*.parquet")

    flightTimeParquetDF.show(5)
    logger.info("Parquet Schema:" + flightTimeParquetDF.schema.simpleString())
```

You should prefer using the Parquet file format as long as it is possible. It is als the recommended and default file format for Apache Spark. 

Common Spark data types in Python: int, long, float, string, datetime.date, datetime.datetime, list/tuple/array, dict. 

And these data types in Spark data types: IntegerType, LongType, FloatType, DoubleType, StringType, DateType, TimestampType, ArrayType, MapType. 

## Spark DataFrameWriter API
The DataFrameWriter API allows for writing data. Its general structure:
```
DataFrameWriter
    .format(...)
    .option(...)
    .partitionBy(...)
    .bucketBy(...)
    .sortBy(...)
    .save()
```

"save mode" has 4 options:
- append. 
- overwrite. Remove existing data files, and create new files. 
- errorIfExists. 
- ignore. If data files already exist, do nothing. Otherwise write the data files. 

By default, you get one output file per partition, because each partition is written by an executor core in parallel. You can customize by re-partitioning the data. 

"partitionBy()" method re-partition your data based on a key column or a composite column key. Key-based partitioning breaks your data logically, and helps to improve your Spark SQL performance, using partition pruning. 

Bucketing with the "bucketBy()" method. Partition your data into a fixed number of predefined buckets. It is only available on Spark managed tables. 

"sortBy()" commonly used with the "bucketBy()", to create sorted buckets. 

MaxRecordsPerFile is an option. Can be used with or without "partitionBy()". Helps to control the file size based on the number of records, and protect you from creating huge/inefficient files. 

## Writing Your Data and Managing Layout

"DataSinkDemo.py":
```py
from pyspark.sql import *
from pyspark.sql.functions import spark_partition_id
from lib.logger import Log4j

if __name__ == "__main__":
    spark = SparkSession \
        .builder \
        .master("local[3]") \
        .appName("SparkSchemaDemo") \
        .getOrCreate()

    logger = Log4j(spark)

    flightTimeParquetDF = spark.read \
        .format("parquet") \
        .load("dataSource/flight*.parquet")

    # see 2 partitions, but only 1 partition has data, so only saved to 1 file
    logger.info("Num Partitions before: " + str(flightTimeParquetDF.rdd.getNumPartitions()))
    flightTimeParquetDF.groupBy(spark_partition_id()).count().show()

    # now have 5 equal partitions
    partitionedDF = flightTimeParquetDF.repartition(5)
    logger.info("Num Partitions after: " + str(partitionedDF.rdd.getNumPartitions()))
    partitionedDF.groupBy(spark_partition_id()).count().show()

    # number of files depends on number of partitions
    partitionedDF.write \
        .format("avro") \
        .mode("overwrite") \
        .option("path", "dataSink/avro/") \
        .save()

    # because of partitionBy(...), 
    # will have many folders named by "OP_CARRIER=..."
    # then in each folder, many subfolders name by "ORIGIN=..."
    # this can improve the performance of some read operations
    # also, these 2 cols are not included in the data file, 
    # because they are redundant
    flightTimeParquetDF.write \
        .format("json") \
        .mode("overwrite") \
        .option("path", "dataSink/json/") \
        .partitionBy("OP_CARRIER", "ORIGIN") \
        .option("maxRecordsPerFile", 10000) \
        .save()
```

Partitioning your data to equal chunks may not make a good sense in most of the cases. We do want to do the partitioning, because we will be working with massive volumes. Also, partitioning the data have benefits, like parallel processing, and partition elimination for certain read operations. 

In real life scenario, the instructor prefer the file size to be between 0.5GB to a few GBs. Control file size by "maxRecordsPerFile" option. 

## Spark Databases and Tables
Apache Spark is not only a set of APIs and a processing engine. It is a database itself. So you can create databases, and tables/views in side them. 

A table has two parts, the table data, and table metadata. 
- The table data live as data files in your distributed storage, you can control the file type, by default, it is a parquet file. 
- The metadata is stored in a meta-store called catalog. It hold info on the table and its data, such as schema, table name, database name, col names, partitions, the physical location of the data files. By default, Spark comes with an in-memory catalog, which is maintained per spark session, and it goes away when the session ends. 

Spark table:
1. Managed tables. Spark manages both metadata and the data. "spark.sql.warehouse.dir" directory is the base location where your managed table data are stored, it is set up by your cluster admin, and you should not change it at runtime. If you drop a managed table, spark will drop the data files and the metadata. 
2. Unmanaged tables (external tables). Only metadata is stored. If you drop an unmanaged table, spark will only drop the metadata. 

We prefer using managed tables, because they offer additional features such as bucketing and sorting. All the future improvements in Spark SQL will also target managed tables. 

## Working with Spark SQL Tables
Create a managed table, and access the catalog. 

Spark depends on Hive metastore, so need to "enableHiveSupport()". 

If you create a managed table in a Spark databse, then your data is available to other sql compliant tools, instead of having to be read into a df. Spark database tables can be accessed using SQL expressions, over JDBC/ODBC connectors. So you can use 3rd-party tools such as Tableau, PowerBI, Talend, etc. Note that plain data files are not accessible through JDBC/ODBC interface. 

"SparkSQLTableDemo.py":
```py
from pyspark.sql import *
from lib.logger import Log4j

if __name__ == "__main__":
    spark = SparkSession \
        .builder \
        .master("local[3]") \
        .appName("SparkSQLTableDemo") \
        .enableHiveSupport() \
        .getOrCreate()

    logger = Log4j(spark)

    flightTimeParquetDF = spark.read \
        .format("parquet") \
        .load("dataSource/")

    # create a db
    spark.sql("CREATE DATABASE IF NOT EXISTS AIRLINE_DB")
    # set db context
    spark.catalog.setCurrentDatabase("AIRLINE_DB")

    # create a table
    flightTimeParquetDF.write \
        .mode("overwrite") \
        # .partitionBy("ORIGIN", "OP_CARRIER") \
        .saveAsTable("flight_data_tbl")
        # or .saveAsTable("airline_db.flight_data_tbl"), with db name

    # access the catalog
    logger.info(spark.catalog.listTables("AIRLINE_DB"))

```

Run the code, in the project folder, see the "spark-warehouse" folder, which is the spark sql warehouse dir. Inside it, there is a folder "airline_db.db" folder, and inside it "flight_data_tbl" folder, and inside it the daa files for the folder. 

There is also a "metastore_db" folder in the project folder, which is the persistent metadata store. 

When you run this code on a local machine, these folders are created in your current directory. However, in a cluster environment, both of these are configured by your cluster admin, and it will be a common location across all the Spark applications. 

Note that you should not partition your data for a col that has too many unique values. Instead, you can use the bucketBy():
```python
    flightTimeParquetDF.write \
        .format("csv") \
        .mode("overwrite") \
        .bucketBy(5, "ORIGIN", "OP_CARRIER") \
        # 5 buckets, so will have 5 files in the table folder
        .sortBy("ORIGIN", "OP_CARRIER") \
        .saveAsTable("flight_data_tbl")
```

Bucket mechanism: it will take the hash of both origin + op_carrier, and take a mod(5), so that each unique key combination will land into the same bucket (file). Sometimes, these buckets can improve your join operations significantly. 

If the records in the bucket are sorted, they might be more ready to use for certain operations. 

