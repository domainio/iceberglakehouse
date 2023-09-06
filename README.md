# Iceberg Lakehouse
Building a Data Lakehouse with Apache Iceberg, Spark, Dremio, Nessie &amp; Minio

## Launch environment
> :warning:  **If you are using mac M1/M2**
> Define these env vars:
> ```
> export DOCKER_BUILDKIT=0
> export COMPOSE_DOCKER_CLI_BUILD=0
> export DOCKER_DEFAULT_PLATFORM=linux/amd64
> ```
### Clone repo
`git clone git@github.com:domainio/iceberglakehouse.git`
### Minio Server
* Open a terminal
* `docker-compose up minioserver`
* Browse to `127.0.0.1:9001`
  * Username/password: `minioadmin`
* Create a bucket: `warehouse`
* Create access key - copy to `.env` file

### Nessie
* Open a terminal
* `docker-compose up nessie`

### Spark Notebook
* Open a terminal
* `docker-compose up spark_notebook`

### Dremio
* Open a terminal
* `docker-compose up dremio`

## Integrate Components
### Create Iceberg tables with Spark notebook
* Go to the terminal where you launched the Spark notebook and locate the URL with the token: `http://127.0.0.1:8888/?token=xxxxx`
* Browse to the URL with the token
* Create a new Pyhton3 notebook
* Run this code:
```
import pyspark
from pyspark.sql import SparkSession
import os
## DEFINE SENSITIVE VARIABLES
NESSIE_URI = os.environ.get("NESSIE_URI") ## Nessie Server URI
WAREHOUSE = os.environ.get("WAREHOUSE") ## BUCKET TO WRITE DATA TOO
AWS_ACCESS_KEY = os.environ.get("AWS_ACCESS_KEY") ## AWS CREDENTIALS
AWS_SECRET_KEY = os.environ.get("AWS_SECRET_KEY") ## AWS CREDENTIALS
AWS_S3_ENDPOINT= os.environ.get("AWS_S3_ENDPOINT") ## MINIO ENDPOINT
print(AWS_S3_ENDPOINT)
print(NESSIE_URI)
print(WAREHOUSE)
conf = (
    pyspark.SparkConf()
        .setAppName('app_name')
        .set('spark.jars.packages', 'org.apache.iceberg:iceberg-spark-runtime-3.3_2.12:1.3.1,org.projectnessie.nessie-integrations:nessie-spark-extensions-3.3_2.12:0.67.0,software.amazon.awssdk:bundle:2.17.178,software.amazon.awssdk:url-connection-client:2.17.178')
        .set('spark.sql.extensions', 'org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions,org.projectnessie.spark.extensions.NessieSparkSessionExtensions')
        .set('spark.sql.catalog.nessie', 'org.apache.iceberg.spark.SparkCatalog')
        .set('spark.sql.catalog.nessie.uri', NESSIE_URI)
        .set('spark.sql.catalog.nessie.ref', 'main')
        .set('spark.sql.catalog.nessie.authentication.type', 'NONE')
        .set('spark.sql.catalog.nessie.catalog-impl', 'org.apache.iceberg.nessie.NessieCatalog')
        .set('spark.sql.catalog.nessie.s3.endpoint', AWS_S3_ENDPOINT)
        .set('spark.sql.catalog.nessie.warehouse', WAREHOUSE)
        .set('spark.sql.catalog.nessie.io-impl', 'org.apache.iceberg.aws.s3.S3FileIO')
        .set('spark.hadoop.fs.s3a.access.key', AWS_ACCESS_KEY)
        .set('spark.hadoop.fs.s3a.secret.key', AWS_SECRET_KEY)
)
## Start Spark Session
spark = SparkSession.builder.config(conf=conf).getOrCreate()
print("Spark Running")
## Create a Table
spark.sql("CREATE TABLE nessie.names (name STRING) USING iceberg;").show()
## Insert Some Data
spark.sql("INSERT INTO nessie.names VALUES ('Backend group'), ('Tikal office'), ('Apache iceberg')").show()
## Query the Data
spark.sql("SELECT * FROM nessie.names;").show()
```

## Inspect warehouse data in minio
* Browse to `127.0.0.1:9001`
* Inspect minio Object browser for the data created by Spark: data and **metadata** files

### Add Nessie data source to Dremio
* Browse to `http://localhost:9047/`
* Add source: nessie
  * General
    * Name: `nessie`
    * Nessie Endpoint URL: `HTTP://nessie:19120/api/v2`
      > ℹ️ Here we use the docker-compose network ability.
      > Also, using the v2 Nessie/Dermio connector
    * Nessue Authentication Type: `None`
* Storage
  * AWS Access Key: `xxxxxxx` <from .env>
  * AWS Access Secret: `xxxxxxx` <from .env>
  * AWS Root Path: `/warehouse`
  * Connection Properties (allow access to storage)
    * Name: `fs.s3a.path.style.access` Value: `true` (access to s3 API) 
    * Name: `fs.s3a.endpoint` Value: `minio:9000` (the container name)
    * Name: `dremio.s3.compat` Value: `true`  (allow to use s3 compatible storage layer)
    * Encrypt connection: unchecked [ ]
    

## Hand-on
### Query Iceberg Metadata
* Querying a Table's **Data File** Metadata `SELECT * FROM TABLE( table_files('<table_name>') )`
* Querying a Table's **History** Metadata `SELECT * FROM TABLE( table_history('<table_name>') )`
* Querying a Table's **Manifest** File Metadata `SELECT * FROM TABLE( table_manifests('<table_name>') )`
* Querying a Table's **Partition** Metadata `SELECT * FROM TABLE( table_partitions('<table_name>') )`
* Querying a Table's **Snapshot** Metadata `SELECT * FROM TABLE( table_snapshot('<table_name>') )`
* Time Travel Queries
  * Time Travel by Timestamps `SELECT * FROM <table_name> AT <timestamp>`
  * Time Travel by Snapshot ID `SELECT * FROM <table_name> AT SNAPSHOT '<snapshot-id>'`

https://docs.dremio.com/current/reference/sql/commands/apache-iceberg-tables/apache-iceberg-select/


### Iceberg Tags
* Run commands on Spark notebook
* `spark.sql("CREATE TABLE IF NOT EXISTS nessie.employees (id BIGINT, name STRING, role STRING, salary double) USING iceberg")`
* `spark.sql("INSERT INTO nessie.employees values (1, 'Harry', 'Software Engineer', 25000), (2, 'John', 'Marketing Ops', 17000)")`
* `spark.sql("SELECT * FROM nessie.employees").show()`
* `spark.sql("ALTER TABLE nessie.employees CREATE TAG etl_1 RETAIN 10 DAYS")`
* `spark.sql("INSERT INTO nessie.employees values (3, 'Jake', 'Architect', 32000), (4, 'Noam', 'Manager', 33000)")`
* `spark.sql("SELECT * FROM nessie.employees").show()`
* `spark.sql("SELECT * FROM nessie.employees VERSION AS OF 'etl_1'").show()`

### Iceberg Branches
* Run commands on Spark notebook
* `spark.sql("ALTER TABLE nessie.employees CREATE BRANCH ML_exp")`
* ```
  schema = spark.table("nessie.employees").schema
  data = [
    (6, "Troy", "CMO", 30000.0),
    (7, "Raine", "UX", 21000.0),
    (8, "Harry", "QA", 22000.0)
  ]
  df = spark.createDataFrame(data, schema)
  df.write.format("iceberg").mode("append").save("nessie.employees.branch_ML_exp")
  ```
* `spark.sql("SELECT * FROM nessie.employees VERSION AS OF 'ML_exp'").show()`
* `spark.sql("SELECT * FROM nessie.employees").show()`
* `spark.sql("SELECT * FROM nessie.employees.refs").show()`
* `spark.sql("ALTER TABLE nessie.employees DROP BRANCH ML_exp")`
* `spark.sql("SELECT * FROM nessie.employees.refs").show()`

### Nessie Branches (!)
* Run commands on Dremio sql runner
* `CREATE BRANCH etl_1 in nessie;`
* Set branch reference to "etl_1"
* `CREATE TABLE nessie.names2 (name VARCHAR);`
* `INSERT INTO nessie.names2 VALUES ('your name');`
* `MERGE BRANCH "etl_1" INTO "main";`
* `SELECT * FROM nessie.names2 AT BRANCH "main";`

### Dremio
* Save as view
* Inspect the data products
