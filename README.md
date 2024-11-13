# 📚 GoodReads Data Pipeline

![GoodReads Pipeline](https://github.com/san089/goodreads_etl_pipeline/blob/master/docs/images/goodreads.png)

Welcome to the **GoodReads Data Pipeline**! This project collects, processes, and analyzes data from GoodReads in real-time using a fully scalable ETL pipeline. Here’s a breakdown of what it does, how it works, and how to get started! 🚀

---

## ⚙️ Architecture

![Pipeline Architecture](https://github.com/san089/goodreads_etl_pipeline/blob/master/docs/images/architecture.png)

This pipeline is powered by several key modules:

- **[GoodReads Python Wrapper](https://github.com/san089/goodreads)** for API data fetching
- **ETL Jobs** for data processing
- **Redshift Data Warehouse Module** for storage and querying
- **Analytics Module** for insights and reporting

---

## 📜 Overview

Data is collected in real-time from the GoodReads API using a custom Python wrapper and is then sent to AWS S3 storage. The pipeline uses **Spark** and **Airflow** for ETL jobs to automate data movement and transformation every 10 minutes.

---

### 🔄 ETL Flow

1. **Data Collection**: Data is fetched from GoodReads API and saved to AWS S3.
2. **Data Transformation**: Spark jobs transform and partition data in a **Processed Zone** on S3.
3. **Data Warehousing**: The transformed data is loaded to Redshift, updating the data warehouse with the latest data.
4. **Data Quality Checks**: Airflow DAGs run quality checks on Redshift tables.
5. **Analytics & Reporting**: Custom queries provide real-time insights, validated with quality checks.

---

## 🛠️ Environment Setup

### ⚙️ Hardware

- **EMR Cluster**: 3 nodes of `m5.xlarge` instances (4 vCore, 16 GiB memory, 64 GiB EBS).
- **Redshift**: 2-node cluster of `dc2.large` instances.

### Setting Up Airflow 🛠️

I've provided a detailed guide to setting up **Airflow** using an **AWS CloudFormation** script. Follow the instructions here: [Airflow using AWS CloudFormation](https://github.com/san089/Data_Engineering_Projects/blob/master/Airflow_Livy_Setup_CloudFormation.md).

> **Note**: This setup uses an EC2 instance and a Postgres RDS instance. Be mindful of AWS charges before running the CloudFormation stack. 

- Install `sshtunnel` for submitting Spark jobs over SSH:
    ```bash
    pip install apache-airflow[sshtunnel]
    ```
- Finally, copy the DAG and plugin folders to EC2’s Airflow home directory. You can also check out the [Airflow Connection Guide](https://github.com/san089/goodreads_etl_pipeline/blob/master/docs/Airflow_Connections.md) for connecting to EMR and Redshift.

### Setting up EMR
Spinning up EMR cluster is pretty straight forward. You can use AWS Guide available [here](https://docs.aws.amazon.com/emr/latest/ManagementGuide/emr-gs.html).

ETL jobs in the project uses [psycopg2](https://pypi.org/project/psycopg2/) to connect to Redshift cluster to run staging and warehouse queries. 
To install psycopg2 on EMR:

    sudo pip-3.6 install psycopg2

psycopg2 uses `postgresql-devel` and `postgresql-libs`, and sometimes pscopg2 installation may fail if these dependencies are not available. To install run commands:

    sudo yum install postgresql-libs
    sudo yum install postgresql-devel

ETL jobs also use [boto3](https://boto3.amazonaws.com/v1/documentation/api/latest/index.html) move files between s3 buckets. To install boto3 run:

    pip-3.6 install boto3 --user

Finally,  pyspark uses python2 as default setup on EMR. To change to python3, setup environment variables:

    export PYSPARK_DRIVER_PYTHON=python3
    export PYSPARK_PYTHON=python3

Copy the ETL scripts to EMR and we have our EMR ready to run jobs. 

### Setting up Redshift
You can follow the AWS [ Guide](https://docs.aws.amazon.com/redshift/latest/gsg/rs-gsg-launch-sample-cluster.html) to run a Redshift cluster or alternatively you can use [Redshift_Cluster_IaC.py](https://github.com/san089/Data_Engineering_Projects/blob/master/Redshift_Cluster_IaC.py) Script to create cluster automatically. 


## How to run 
Make sure Airflow webserver and scheduler is running. 
Open the Airflow UI `http://< ec2-instance-ip >:< configured-port >` 

GoodReads Pipeline DAG
![Pipeline DAG](https://github.com/san089/goodreads_etl_pipeline/blob/master/docs/images/goodreads_dag.PNG)

DAG View:
![DAG View](https://github.com/san089/goodreads_etl_pipeline/blob/master/docs/images/DAG.PNG)

DAG Tree View:
![DAG Tree](https://github.com/san089/goodreads_etl_pipeline/blob/master/docs/images/DAG_tree_view.PNG)

DAG Gantt View: 
![DAG Gantt View](https://github.com/san089/goodreads_etl_pipeline/blob/master/docs/images/DAG_Gantt.PNG)


## Testing the Limits
The `goodreadsfaker` module in this project generates Fake data which is used to test the ETL pipeline on heavy load.  

 To test the pipeline I used `goodreadsfaker` to generate 11.4 GB of data which is to be processed every 10 minutes (including ETL jobs + populating data into warehouse + running analytical queries) by the pipeline which equates to around 68 GB/hour and about 1.6 TB/day.

Source DataSet Count:
![Source Dataset Count](https://github.com/san089/goodreads_etl_pipeline/blob/master/docs/images/DatasetCount.PNG)


DAG Run Results:
![GoodReads DAG Run](https://github.com/san089/goodreads_etl_pipeline/blob/master/docs/images/DAG_tree_view.PNG)

Data Loaded to Warehouse:
![GoodReads Warehouse Count](https://github.com/san089/goodreads_etl_pipeline/blob/master/docs/images/WarehouseCount.PNG)



## Scenarios

-   Data increase by 100x. read > write. write > read
    
    -   Redshift: Analytical database, optimized for aggregation, also good performance for read-heavy workloads
    -   Increase EMR cluster size to handle bigger volume of data

-   Pipelines would be run on 7am daily. how to update dashboard? would it still work?
    
    -   DAG is scheduled to run every 10 minutes and can be configured to run every morning at 7 AM if required. 
    -   Data quality operators are used at appropriate position. In case of DAG failures email triggers can be configured to let the team know about pipeline failures.
    
-   Make it available to 100+ people
    -   We can set the concurrency limit for your Amazon Redshift cluster. While the concurrency limit is 50 parallel queries for a single period of time, this is on a per cluster basis, meaning you can launch as many clusters as fit for you business.
