---
title: 'Zoom Data Engineering Part 2'

header:
  image: "/images/chicagotwo.jpeg"

date: 2022-07-21

toc: true
toc_label: "Table of Contents" 


tags:
  - Data Engineering
  - Tutorials
  - SQL
  - Database
---


In this section I will extract data from the source and upload it to our Data Warehouse in Google. Which covers topics such as **Data Lake** and *Pipeline Orchestartion* with **Airflow**.


## Data Lake

What is a Data Lake?

![](https://learn.g2crowd.com/hubfs/What-is-a-data-lake.png)

A *data lake* is a repository for structured, unstructured, and semi-structured data. Data lakes are much different from data warehouses since they allow data to be in its rawest form without needing to be converted and analyzed first.

The main goal behind a Data Lake is being able to ingest data as quickly as possible and making is easily available to all memebers of a data team!


## Data Lake (DL) vs. Data Warehouse (DW)

![](https://www.grazitti.com/wp-content/uploads/2018/10/BlogimageDataWarehouse-1.png)


Basic Differences:

* Data Processing: 
  * DL: Data is **raw** and hasn't been cleaned (if so then little), data is generally unstructured.
  * DW: Data is **refined**; it has been cleaned, pre-processed and structured for use

* Size:
  * DL: Data Lakes are **LARGE** and contains a lot of data
  * DW: Data Warehouses in comparison are **SMALLER** as datan has been cleaned and structured

* Users:
  * DL: Data Scientists, Data Analysts, Data Engineers generally have access
  * DW: Business Analyst (Sorry we don't trust you with the data lake)

* Use Cases:
  * DL: Stream Processing, Machine Learning, Real-Time Analytics
  * DW: Batch Processing, Reporting, Business Intelligence


Why did Data Lakes start coming into existence?

*  companies started to realize the importance of data, they soon found out that they couldn't ingest data right away into their DWs and didn't want to wasted any of the data.


## ETL vs. ELT

When ingesting data, DWs use the Export, Transform and Load (ETL) model whereas DLs use Export, Load and Transform (ELT).


ELT (Schema on read) the data is directly stored without any transformations and any schemas are derived when reading the data from the DL.


## Data Swamp 

Data Lakes that have gone wrong. 

![](https://cdn.britannica.com/61/66861-050-D9F41AC6/Lily-pads-Okefenokee-Swamp-Georgia.jpg)


  
## Data Lake Cloud Providers

* Google Cloud Platform: Cloud Storage
* Amazon Web Services: Amazon S3
* Microsoft Azure: Azure Blob Storage

For this Project we will use Google Cloud Storage.



## Airflow Orchestation

**What is Airflow?**

[Airflow](https://airflow.apache.org/) is a platform created by the community to programmatically author, `schedule and monitor workflows`. So Airflow provides us with a platform where we can `create` and `orchestrat`e our workflow or pipelines. In Airflow, these workflows are represented as **DAGs**.

Here is Airflow Architecture:

![](https://airflow.apache.org/docs/apache-airflow/2.0.1/_images/arch-diag-basic.png)

Heres a few [notes](https://airflow.apache.org/docs/apache-airflow/2.0.1/concepts.html) on the components above:

* **Metadata Database**: Airflow uses a SQL database to store metadata about the data pipelines being run. In the diagram above, this is represented as Postgres which is extremely popular with Airflow. Alternate databases supported with Airflow include MySQL.

**Web Server and Scheduler:** The Airflow web server and Scheduler are separate processes run (in this case) on the local machine and interact with the database mentioned above.

The **Executor** is shown separately above, since it is commonly discussed within Airflow and in the documentation, but in reality it is **NOT a separate process, but run within the Scheduler**

The **Worker(s)** are separate processes which also interact with the other components of the Airflow architecture and the metadata repository.

`airflow.cfg` is the Airflow configuration file which is accessed by the Web Server, Scheduler, and Workers.

**DAGs** (Directed Acyclic Graph) refers to the DAG files containing Python code, representing the data pipelines to be run by Airflow. The location of these files is specified in the Airflow configuration file, but they need to be accessible by the Web Server, Scheduler, and Workers. They are usally in a seperate `dag` folder within the project.

**Core Ideas and Additional Definitions:**

**DAG** is a `collection of all the tasks` you want to run, organized in a way that reflects their relationships and dependencies.  DAG describes how you want to carry out your workflow.
  * Task Dependencies (control flow: `>>` or `<<`)

I have tons of notes on DAGs here: [Notes on Graphs](https://devintheengineer.com/algorithms/intro_graph)

**Task**: a defined unit of work. Tasks describe what to do (e.g fetch data, run analysis, trigger something, etc). Most common types of tasks are:
  * **Operators**:
  * **Sensors**
  * **TaskFlow Decorator**


In this lesson we will create a more *complex* pipeline using Airflow:

```bash
(web)
  ↓
DOWNLOAD
  ↓
(csv)
  ↓
PARQUETIZE
  ↓
(parquet)
  ↓
UPLOAD TO GCS
  ↓
(parquet in GCS)
  ↓
UPLOAD TO BIGQUERY
  ↓
(Table in BQ)
```

**Quick Note:**
What is *Parquet*? [Parquet](https://parquet.apache.org/) is a columnar storage datafile format which is *more efficient* than CSV.



## Airflow Configuration for Setting up on Docker

### Pre-requisites

[Video Setting up GCS](https://www.youtube.com/watch?v=k-8qFh8EfFA)


1. Assume you have a Google Service account credentials JSON file is named `google_credentials.json` and stored in `$HOME/.google/credentials/`. Copy and rename your credentials file to the required path.

Once you've downloaded the *key* you can set the environment variables to point to the auth keys:
i. The environment variable name is GOOGLE_APPLICATION_CREDENTIALS

ii. The value for the variable is the path to the json authentication file you downloaded previously.

iii. Check how to assign environment variables in your system and shell. In bash, the command should be:

```bash
export GOOGLE_APPLICATION_CREDENTIALS="<path/to/authkeys>.json"
```
iv. You can Refesh the token and verify the authentication with the GCP SDK:

```bash
gcloud auth application-default login
```



### Setup

* Setting Up Airflow with Docker

1. Create a new `airflow` subdirectory in your work directory.
2. Download the official Docker-compose YAML file for the latest Airflow version.

    ```bash
    curl -LfO 'https://airflow.apache.org/docs/apache-airflow/2.2.3/docker-compose.yaml'
    ```
    * The official `docker-compose.yaml` file is quite complex and contains [several service definitions](https://airflow.apache.org/docs/apache-airflow/stable/start/docker.html#docker-compose-yaml).
    * For a refresher on how `docker-compose` works, you can [check out this lesson from the ML Zoomcamp](https://github.com/ziritrion/ml-zoomcamp/blob/main/notes/10_kubernetes.md#connecting-docker-containers-with-docker-compose).

3. We now need to [set up the Airflow user](https://airflow.apache.org/docs/apache-airflow/stable/start/docker.html#setting-the-right-airflow-user). For MacOS (if you're cool), create a new `.env` in the same folder as the `docker-compose.yaml` file with the content below:

    ```bash
    AIRFLOW_UID=50000
    ```



4. The *base* Airflow Docker image **won't work with GCP**, so we need to [customize it](https://airflow.apache.org/docs/docker-stack/build.html) to suit our needs. You may download a GCP-ready Airflow Dockerfile [here](https://github.com/devinpowers/zoom-data-engineering/blob/main/Week2-Ingestion/Dockerfile). A few things of note:
    * We use the base Apache Airflow image as the base.
    * We install the GCP SDK CLI tool so that Airflow can communicate with our GCP project.
    * We also need to provide a [`requirements.txt` file](https://github.com/devinpowers/zoom-data-engineering/blob/main/Week2-Ingestion/requirements.txt) to install Python dependencies. The dependencies are:
      * `apache-airflow-providers-google` so that Airflow can use the GCP SDK.
      * `pyarrow` , a library to work with parquet files.

5. Alter the `x-airflow-common` service definition inside the `docker-compose.yaml` file as follows:
   * We need to point to our custom Docker image. At the beginning, comment or delete the `image` field and uncomment the `build` line, or arternatively, use the following (make sure you respect YAML indentation):
      ```yaml
        build:
          context: .
          dockerfile: ./Dockerfile
      ```
    * Add a volume and point it to the folder where you stored the credentials json file. Assuming you complied with the pre-requisites and moved and renamed your credentials, add the following line after all the other volumes:
      ```yaml
      - ~/.google/credentials/:/.google/credentials:ro
      ```
    * Add 2 new environment variables right after the others: `GOOGLE_APPLICATION_CREDENTIALS` and `AIRFLOW_CONN_GOOGLE_CLOUD_DEFAULT`:
      ```yaml
      GOOGLE_APPLICATION_CREDENTIALS: /.google/credentials/google_credentials.json
      AIRFLOW_CONN_GOOGLE_CLOUD_DEFAULT: 'google-cloud-platform://?extra__google_cloud_platform__key_path=/.google/credentials/google_credentials.json'
      ```
    * Add **2 new additional environment variables** for your GCP project ID and the GCP bucket that **Terraform** should have created [in the previous post](https://devintheengineer.com/Zoom-Data-Engineering-Part1/). You can find this info in your *GCP project's dashboard*. Note you have to have Terraform running (setting up infrastructure)

      ```yaml
      GCP_PROJECT_ID: '<your_gcp_project_id>'
      GCP_GCS_BUCKET: '<your_bucket_id>'
      ```

      Here are my Project Id and GCS Bucket that I got from Terraform.
      ```bash
      GCP_PROJECT_ID: 'new-try-zoom-data'
      GCP_GCS_BUCKET: 'dtc_data_lake_new-try-zoom-data'
      ```

  * Change the `AIRFLOW__CORE__LOAD_EXAMPLES` value to `'false'`. This will prevent Airflow from populating its interface with DAG examples.

6. You may find a modified `docker-compose.yaml` file [in this link](https://github.com/devinpowers/zoom-data-engineering/blob/main/Week2-Ingestion/docker-compose.yml).

  * The only thing you need to change in the given `docker-compose.yaml` file is your `GCP_PROJECT_ID` and `GCP_GCS_BUCKET`

7. Additional notes:
    * The YAML file uses [`CeleryExecutor`](https://airflow.apache.org/docs/apache-airflow/stable/executor/celery.html) as its executor type, which means that tasks will be *pushed to workers* (**external Docker containers**) rather than running them locally (as regular processes). You can change this setting by modifying the `AIRFLOW__CORE__EXECUTOR` environment variable under the `x-airflow-common` environment definition.







## Execution Steps

* main Exection steps for both local build and not


1. Build the image (only first-time, or when there's any change in the `Dockerfile`, takes ~15 mins for the first-time):

  ```shell
  docker-compose build
  ```

2. Initialize the Airflow scheduler, DB, and other configurations

  ```shell
  docker-compose up airflow-init
  ```

3. Run Airflow

  ```shell
  docker-compose up -d 
  ```

4. You may now access the Airflow GUI by browsing to `localhost:8080`. Username and password are both airflow



## Creating a DAG


* Add Later About creating DAGS


### Running Dags

DAG management is carried out via Airflow's web UI on our `localhost:8080`

There are 2 main ways to run DAGs:

1. Triggering them manually via the web UI or programatically via API
2. Scheduling them


## Airflow in Action for this Project

We will run two different scripts, one for Ingesting data to our Local Postgres with Airflow and the second for **Ingesting data to GCP**



### Ingesting data to local Postgres with Airflow

* Have to use extra `.yaml` file here Ill liunk

Steps:

1.  Prepare an ingestion script. Here is [mine](https://github.com/devinpowers/zoom-data-engineering/blob/main/Week2-Ingestion/airflow_local/dags/ingest_script.py), put this in our `/dags` subdirectory in my work folder.


2. Create a DAG for the local ingestion. [Here is my DAG file](https://github.com/devinpowers/zoom-data-engineering/blob/main/Week2-Ingestion/airflow_local/dags/data_ingestion_local.py) and copy it to the `/dags` subdirectory in my work folder like above.

  * Some modifictions have been made here since the **AWS** bucket containing the origninal files is gone.
  * Updated to the new `dataset_url` and added `.gz` extension to the since that's the format the files are in!
  * Updated the `parquet_file* as well to account for the `.gz` extension

3. Modify the `.env` file to include:

  ```bash
  AIRFLOW_UID=50000

  PG_HOST=pgdatabase
  PG_USER=root
  PG_PASSWORD=root
  PG_PORT=5432
  PG_DATABASE=ny_taxi
  ```

4. Make sure the Airflow `docker-compose.yaml` file to include the environment variables (`.env`)


5. Modify the Airflow `Dockerfile` to include the `ingest_script.py` (could take time to complie first run)

  * Add this right after installing the requirements.txt file: RUN pip install --no-cache-dir pandas sqlalchemy psycopg2-binary

6. Rebuild the Airflow image with (might take a few when first built):

  ```bash
  docker-compose build 
  ```

7. Initialize the Airflow config with:

  ```bash
  docker-compose up airflow-init
  ```

8. Start Airflow by running (will take 5 or so minutes)

  ```bash
  docker-compose up
  ```

And in a seperate terminal find out what network it is running on by using:

  ```bash
  docker network ls
  ```
It most likely will be something like `airflow_default`.

9. Now were going to **MODIFY** the `docker-compose.yaml` file from Post 1 (which includes Dockerizing our database).  We will also rename the file to `docker-compose-lesson2.yaml`. Here we will add the `shared virtual network` and include it in our `.yaml` file like shown below:

  ```bash
  networks:
    airflow:
      xternal:
        name: airflow_better_default
  ```
[Heres a link to my `docker-compose-lesson2.yaml` file](https://github.com/devinpowers/zoom-data-engineering/blob/main/Week2-Ingestion/airflow_local/docker-compose-lesson2.yaml)

Run the `docker-compose-lesson2.yaml` now:

  ```bash
  docker-compose -f docker-compose-lesson2.yaml up
  ```
We run it like this above because of the *different* name


10. Open Airflow Dashbaord via `localhost:8080` and run the `LocalIngestionDag` by clicking on the Play icon.


11. When both the download and ingest tasks are finished we will log into the database

```bash
pgcli -h localhost -p 5431 -u root -d ny_taxi
```

* Run some queries on our database now to test that it worked!

12. When finish we will run 

```bash
docker-compose down
```

on both the Airflow and Postgres terminals !!



---------------------




### Ingesting data to GCP


Here we will run a more *complex* DAG that will download the NYC taxi trip data, convert it to parquet, upload it to a GCP bucket and ingest it to GCP's BigQuery. Heres a step-by-step visualization:


```bash
(web)
  ↓
DOWNLOAD
  ↓
(csv.gz)
  ↓
PARQUETIZE
  ↓
(parquet)
  ↓
UPLOAD TO GCS
  ↓
(parquet in GCS)
  ↓
UPLOAD TO BIGQUERY
  ↓
(Table in BQ)
```

**Steps:** FOR Uploading to Google Cloud!!!



7. Start Airflow by using `docker-compose up` and on a separate terminal, find out which *virtual network* it's running on with `docker network ls`. It most likely will be something like `airflow_default`.

8. Modify our `docker-compose.yaml` file from post one, remember this includes setting up; here is a [link to .yaml file](https://github.com/devinpowers/zoom-data-engineering/blob/main/Week2-Ingestion/docker-compose-lesson2.yaml)



  * We can `#` out the pgAdmin stuff

9. Run the updated `docker-compose-lesson2.yaml` with `docker-compose -f docker-compose-lesson2.yaml up` . We need to explicitly call the file because we're using a non-standard name.

10. Open the Airflow Dashboard on `localhost:8080` and trigger the ``google-gcc` DAG by clicking on the Play icon.


11. Once the DAG finishes, you can go to your GCP project's dashboard and search for BigQuery. You should see your project ID; expand it and you should see a new trips_data_all database with an external_table table.

12. On finishing your run or to shut down the container/s:

  ```shell
  docker-compose down
  ```