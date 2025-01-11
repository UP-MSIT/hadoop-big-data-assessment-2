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
  zookeeper:
    image: bitnami/zookeeper:latest
    container_name: zookeeper
    ports:
      - "2181:2181"
    environment:
      - ALLOW_ANONYMOUS_LOGIN=yes

  kafka:
    image: bitnami/kafka:latest
    container_name: kafka
    ports:
      - "9092:9092"
    environment:
      - KAFKA_BROKER_ID=1
      - KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181
      - ALLOW_PLAINTEXT_LISTENER=yes

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

  # hive-server:
  #   image: hive-server:arm64
  #   container_name: hive-server
  #   hostname: hive-server
  #   depends_on:
  #     - hdfs-namenode
  #     - hive-metastore
  #   env_file:
  #     - ./hadoop-hive.env
  #   environment:
  #     HIVE_CORE_CONF_javax_jdo_option_ConnectionURL: "jdbc:postgresql://hive-metastore/metastore"
  #     SERVICE_PRECONDITION: "hive-metastore:9083"
  #   ports:
  #     - "10000:10000"
  #   volumes:
  #     - ./hadoop-namenode:/opt/hadoop/etc/hadoop

  # hive-metastore:
  #   image: hive-metastore:arm64
  #   container_name: hive-metastore
  #   hostname: hive-metastore
  #   depends_on:
  #     - hdfs-namenode
  #     - hive-metastore-postgresql
  #   env_file:
  #     - ./hadoop-hive.env
  #   command: /opt/hive/bin/hive --service metastore
  #   environment:
  #     SERVICE_PRECONDITION: "hdfs-namenode:50070 hdfs-datanode:50075 hive-metastore-postgresql:5432"
  #   ports:
  #     - "9083:9083"
  #   volumes:
  #     - ./hadoop-namenode:/opt/hadoop/etc/hadoop

  # hive-metastore-postgresql:
  #   image: postgres:latest
  #   container_name: hive-metastore-postgresql
  #   hostname: hive-metastore-postgresql
  #   environment:
  #     POSTGRES_DB: metastore
  #     POSTGRES_USER: hive
  #     POSTGRES_PASSWORD: hive
  #   ports:
  #     - "5432:5432"
  #   volumes:
  #     - postgres-data:/var/lib/postgresql/data

  trino:
    image: trinodb/trino
    container_name: trino
    hostname: trino
    ports:
      - "8082:8080"
    depends_on:
      - hdfs-namenode
  
  cassandra:
    image: cassandra:latest
    container_name: cassandra
    ports:
      - "9042:9042"

  grafana:
    image: grafana/grafana
    container_name: grafana
    ports:
      - "3000:3000"

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

- Spark Service
    - Inside Spark
    - How Spark work
- Spark Notebook
    - For writing jupyter notebook pyspark code
- Example code using Spark SQL
    - Load/Read dataset from HDFS
    - Execute SQL in dataset from HDFS
    - Function calculation using Spark