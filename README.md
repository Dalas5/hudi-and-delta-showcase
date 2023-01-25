
# Apache Hudi vs Delta Lake Showcase

In recent years, ACID compliant data lakes have gained popularity. This transactionality capability is managed by storage layers called Metadata and Governance Layers, two of the most popular competitors are Databricks Delta Lake and Uber’s Hudi.

Both Hudi and Delta work with data abstractions based on *parquet* file format, but each handle transactionality in different ways and provide different features for data lakes. Both tools will be showcased in a basic example to understand their functionality under the hood.

## Environmnet Setup

The infrastructure in which the demo will take place is depicted below.

![Environment setup](/images/poc-structure.png)

**Source Database**: Cloud SQL MySQL  
**Change Data Capture Tool**: GCP Datastream  
**Hudi Setup**: Dataproc Cluster Image 2.0.53-debian10  
**Delta Setup**: Databricks Runtime 9.1  
**Object/File Storage**: GCP Cloud Storage


## Data Preparation  
### 1. Bucket Creation

The first bucket will store the information streamed by the CDC Tool

```bash
gcloud storage buckets create gs://dalas-data-bucket
```

The second bucket will store the hudi tables
```bash
gcloud storage buckets create gs://data-lake-cdc
```

### 2. Cloud SQL Instance - MySQL

The command shown below was used to create the MySQL instance, taking into account the configurations needed to make it work alongside Datastream.


```bash
MYSQL_INSTANCE=mysql-db
DATASTREAM_IPS=34.72.28.29,34.67.234.134,34.67.6.157,34.72.239.218,34.71.242.81
gcloud sql instances create ${MYSQL_INSTANCE} \
--cpu=2 --memory=10GB \
--authorized-networks=${DATASTREAM_IPS} \
--enable-bin-log \
--region=us-central1 \
--database-version=MYSQL_8_0 \
--root-password password123
```

Some key points to keep in mind to being able to connect MySQL with Datastream are:
* Make sure that binary logging is enabled on your cloudSQL, if not, the instance can be patched with the following command:

```bash
gcloud sql instances patch <instance name> --enable-bin-log
```
* Set a list of authorized networks to allow connectivity for Datastream 


### MySQL Instance Data Load

After connecting to the instance through the cloud shell, a new database called *demo* is created to store the tables with data. Also a simple table called hudi_delta_test is created with the following schema, using commands below.

```sql
CREATE DATABASE demo;
USE demo;

CREATE TABLE hudi_delta_test
(
pk_id integer,
name varchar(255),
value integer,
updated_at timestamp default now() on update now(),
created_at timestamp default now(),
constraint pk primary key(pk_id)
);
```

Initial demo data is loaded with the following commands.

```sql
insert into hudi_delta_test(pk_id,name,value) values(1,’apple’,10);
insert into hudi_delta_test(pk_id,name,value) values(2,’samsung’,20);
insert into hudi_delta_test(pk_id,name,value) values(3,’dell’,30);
insert into hudi_delta_test(pk_id,name,value) values(4,’motorola’,40);
```

For Datastream to connect properly, is neccesary to add a datastream user to MySQL instance, this is done with the following SQL commands.
```sql
CREATE USER 'datastream'@'%' IDENTIFIED BY 'data';
GRANT REPLICATION SLAVE, SELECT, RELOAD, REPLICATION CLIENT, LOCK TABLES, EXECUTE ON *.* TO 'datastream'@'%';
FLUSH PRIVILEGES;
```


### 3. Datastream  

First, it is needed to provide permissions to Datastream service account to read and create objects on the destination bucket. The email address of the service account will be **service-[project_number]@gcp-sa-datastream.iam.gserviceaccount.com**

Select the destination bucket and assign the following permissions to it:
```
roles/storage.objectViewer 
roles/storage.objectCreator
roles/storage.legacyBucketReader
```

Now the stream creation process can begin. First, the Stream name and ID are set, as well as the region, Source type (MySQL), and Destination type (Cloud Storage).


![create view image](./images/datastream-create.png)

Before continuing, prerequisites are checked as provided in the documentation attached in the same window. This prerequisites were set in place already by previous steps, but always check them in case something has changed overtime.

![prerequisites image](./images/datastream-prerequisites.png)

Next, the MySQL connection profile is defined. The following details are needed:
* Connection name and Connection ID
* Hostname or IP of the cloudSQL or MySQL
* Username and Password for the Datastream user created



![connection profile](/images/datastream-conn-prof-1.png)


![connection profile 2](/images/datastream-conn-prof-2.png)


Next for the connectivity, you can use various connectivity method such as

    Private connectivity (VPC Peering)
    IP Whitelisting
    Proxy tunneling

For this demo, IP Whitelisting was chosen.

Once the connectivity is established, the connection connection can be tested to ensure the connection doesn't have any blockers.

![connection test](/images/datastream-test-passed.png)


To configure the Source, we get to choose which tables and schemas are to be included for the datastream to consider for the CDC. For this demo, the whole database created in MySQL (*demo*) and it's future tables are chosen.

![configure source image](/images/datastream-source-config.png)

Define Destination step. In this step, the destination bucket name is selected and any spefic Path prefix to store the output files can be specified too.

![destination image](/images/datastream-define-destination-1.png)

Datastream allows to store the change data in two formats: avro or json. For this demo, avro is the format chosen.

![destination 2 image](/images/datastream-define-destination-2.png)



In the final step, we can run validations, to validate the configuration and then click create and start to have the stream get started straight away.


![details image](/images/datastream-review-details-1.png)

![details image 2](/images/datastream-review-details-2.png)

Now the Datastream created will ingest changes from the MySQL hosted on CloudSQL into the Datastream.

### 4. Data changes

Changes are introduced into MySQL database in order to have change data streamed into GCS and later work with it with Hudi and Delta.

The changes are issued by connecting again to MySQL cloud shell, and running the SQL command below.

```sql
insert into hudi_delta_test(pk_id,name,value) values(4,’motorola’,40);
update hudi_delta_test set value = 201 where pk_id=2;
delete from hudi_delta_test where pk_id=3;
```


## Cluster Setup

Before creating the cluster, it is needed to create a auxiliary script file that will tell GCP what packages to load in the machines in the creation step. To work with hudi alongside spark, the right versions of each package need to be chosen. As of January 2023, Dataproc Image 2.0.53-debian10 comes with Spark 3.1.3,  and the following dependencies specified in the bash script work well with it.


Packages install file

```bash
get  -P /usr/lib/delta/jars/ "https://repo1.maven.org/maven2/org/apache/spark/spark-avro_2.12/3.1.3/spark-avro_2.12-3.1.3.jar"

wget  -P /usr/lib/delta/jars/ "https://repo1.maven.org/maven2/org/apache/hudi/hudi-spark3-bundle_2.12/0.12.2/hudi-spark3-bundle_2.12-0.12.2.jar"
```

Create this script file and locate it in a bucket to use it later with the Dataproc configuration step.

For any other spark versions, refer to this compatibility documentation.  
* https://hudi.apache.org/docs/quick-start-guide/
* https://docs.delta.io/latest/releases.html
* https://docs.databricks.com/delta/index.html



## Cluster creation

```bash
gcloud dataproc clusters create cluster-c9cc \
--enable-component-gateway \
--region us-central1 \
--zone us-central1-c \
--master-machine-type n1-standard-2 \
--master-boot-disk-size 30 \
--num-workers 2 \
--worker-machine-type n1-standard-2 \
--worker-boot-disk-size 30 \
--image-version 2.0-debian10 \
--optional-components HIVE_WEBHCAT,JUPYTER \
--initialization-actions 'gs://data-lake-cdc/packages-install.sh' \
--project dalas-gcp-data-eng
```



## References
* https://blog.searce.com/giving-a-spin-to-cloud-datastream-the-new-serverless-cdc-offering-on-google-cloud-114f5132d3cf
* https://www.cloudskillsboost.google/focuses/22950?parent=catalog
* https://medium.com/swlh/apache-hudi-vs-delta-lake-295c019fe3c5