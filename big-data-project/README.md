# Hadoop Big Data Assessment II

# Setting up Docker compose service

> *This Dockerfile is optimized for ARM architecture (Apple M Chip)*
> 

### **Step 1: Prerequisites**

- Ensure Docker and Docker Compose are installed on your machine.
- Clone the repository containing your project.

### **Step 2: Project Structure**

Your project structure should look like this:

```bash
big-data-project/
  ├── docker-compose.yml
  ├── hadoop-namenode/
  │   ├── core-site.xml
  │   ├── hdfs-site.xml
  ├── hadoop-datanode/
  │   ├── core-site.xml
  │   ├── hdfs-site.xml
  ├── notebooks/
  │   ├── Dockerfile
  │   ├── spark-defaults.conf
  ├── hadoop-hive.env
  ├── README.md
  └── ...
```

### **Step 3: Docker Compose Configuration**

Here is the docker-compose.yml configuration for your services:

```bash
version: '3.8'

services:
  hdfs-namenode:
    image: hadoop-namenode:arm64
    container_name: hdfs-namenode
    hostname: namenode
    ports:
      - "9870:9870"  # NameNode web UI
      - "9000:9000"  # NameNode RPC
    environment:
      - CLUSTER_NAME=test
    volumes:
      - namenode:/hadoop/dfs/name
      - ./hadoop-namenode:/opt/hadoop/etc/hadoop

  hdfs-datanode:
    image: hadoop-datanode:arm64
    container_name: hdfs-datanode
    hostname: datanode
    ports:
      - "9864:9864"  # DataNode web UI
    environment:
      - CLUSTER_NAME=test
    depends_on:
      - hdfs-namenode
    volumes:
      - datanode:/hadoop/dfs/data
      - ./hadoop-datanode:/opt/hadoop/etc/hadoop

  spark-master:
    image: bitnami/spark:latest
    container_name: spark-master
    ports:
      - "8080:8080"
      - "7077:7077"
    environment:
      - SPARK_MODE=master

  spark-worker:
    image: bitnami/spark:latest
    container_name: spark-worker
    depends_on:
      - spark-master
    environment:
      - SPARK_MODE=worker
      - SPARK_MASTER=spark://spark-master:7077
    ports:
      - "8081:8081"

  spark-notebook:
    image: vengleab/docker-hive-arm:spark-notebook
    build:
      context: ./notebooks
    user: root
    environment:
      - JUPYTER_ENABLE_LAB="yes"
      - GRANT_SUDO="yes"
    volumes:
      - ./notebooks:/home/jovyan/work
      - ./notebooks/spark-defaults.conf:/usr/local/spark/conf/spark-defaults.conf
    ports:
      - "8888:8888"
      - "4040:4040"

volumes:
  namenode:
  datanode:
  postgres-data:

networks:
  default:
    driver: bridge
```

### **Step 4: HDFS Configuration**

Create the following configuration files for HDFS:

> Create both file in ***hadoop-namenode/*** and ***hadoop-datanode/***
> 

```bash
big-data-project/
  ├── hadoop-namenode/
  │   ├── core-site.xml
  │   ├── hdfs-site.xml
  ├── hadoop-datanode/
  │   ├── core-site.xml
  │   ├── hdfs-site.xml
```

***core-site.xml***

```bash
<configuration>
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://namenode:9000</value>
  </property>
</configuration>
```

***hdfs-site.xml***

```bash
<configuration>
  <property>
    <name>dfs.replication</name>
    <value>1</value>
  </property>
  <property>
    <name>dfs.namenode.name.dir</name>
    <value>file:///opt/hadoop/dfs/name</value>
  </property>
  <property>
    <name>dfs.datanode.data.dir</name>
    <value>file:///opt/hadoop/dfs/data</value>
  </property>
</configuration>
```

***cd into hadoop-namenode and run docker build image***

```bash
$ docker build -t hadoop-namenode:arm64 -f Dockerfile .
$ docker build -t hadoop-datanode:arm64 -f Dockerfile .
$ cd
$ docker compose -f docker-compose.yml up -d

OR
$ docker build -t hadoop-namenode:arm64 -f hadoop-namenode/Dockerfile hadoop-namenode
$ docker build -t hadoop-datanode:arm64 -f hadoop-datanode/Dockerfile hadoop-datanode

```

Put dataset into HDFS

```bash
# Run cmd from PC
# DatasetFile is directory and files
# Copy to container id (dd208076ddf6)

docker ps
docker cp ~/Downloads/DatasetFile container_id :/       

docker exec -it hdfs-namenode bash

hdfs dfs -ls  /

# Copy file into HDFS
hadoop fs -copyFromLocal /datasetsFile / # Root Directory
```

### **Step 5: Spark Configuration**

Create the following configuration files for Spark:

spark-defaults.conf

```bash
spark.jars                                          jars/*
spark.sql.extensions                                io.delta.sql.DeltaSparkSessionExtension
spark.sql.catalog.spark_catalog                     org.apache.spark.sql.delta.catalog.DeltaCatalog
# s3 configuration
spark.hadoop.fs.s3.endpoint                        http://minio:9000
spark.hadoop.fs.s3.access.key                      minio
spark.hadoop.fs.s3.secret.key                      minio123
spark.hadoop.fs.s3.path.style.access               true
spark.hadoop.fs.s3.connection.ssl.enabled          false
spark.hadoop.fs.s3.impl                            org.apache.hadoop.fs.s3a.S3AFileSystem
# s3a configuration
spark.hadoop.fs.s3a.endpoint                        http://minio:9000
spark.hadoop.fs.s3a.access.key                      minio
spark.hadoop.fs.s3a.secret.key                      minio123
spark.hadoop.fs.s3a.path.style.access               true
spark.hadoop.fs.s3a.connection.ssl.enabled          false
spark.hadoop.fs.s3a.impl                            org.apache.hadoop.fs.s3a.S3AFileSystem
```

### **Step 6: Build and Run the Project**

Navigate to the *big-data-project* directory and run the following commands:

For ARM Chip (Mac M1 and up):

```bash
make build-arm
make start-arm
```

### **Step 7: Access the Services**

- **HDFS NameNode Web UI**: `http://localhost:9870`
- **Spark Master Web UI**: `http://localhost:8080`
- **Spark Worker Web UI**: `http://localhost:8081`
- **Jupyter Notebook**: `http://localhost:8888`

### **Step 8: Example Use Case**

Here is an example of how to use Spark to read data from HDFS and perform some transformations:

```bash
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, sum

# Initialize Spark session
spark = SparkSession.builder \
    .appName("AmazonDataProcessing") \
    .config("spark.hadoop.fs.defaultFS", "hdfs://namenode:9000") \
    .getOrCreate()

# Read the Amazon data from HDFS
file_path = "hdfs://namenode:9000/datasets/Amazon Sale Report.csv"
df = spark.read.csv(file_path, header=True, inferSchema=True)

# Calculate the total sales amount per city
total_sales_per_city = df.groupBy("ship-city").agg(sum("Amount").alias("total_sales"))

# Show the result
total_sales_per_city.show()

# Stop the Spark session
spark.stop()
```

### **Summary**

This *docker-compose.yml* file sets up a big data environment with the following services:

- **HDFS NameNode and DataNode**: Distributed storage system for managing large datasets.
- **Spark Master and Worker**: Distributed processing framework for big data analytics.
- **Spark Notebook**: Jupyter notebook interface for interacting with Spark.