# Process & Ingest Open Weather Map Data In DWH

## Tech Stack

1. Open Weather Map Data (API Source)
2. S3
3. Airflow
4. Glue
5. Redshift
6. CI/CD with AWS CodeBuild

## Process Type

Batch

## Objective

We will be using an open API to fetch weather data for various cities and countries across the globe. This data will be sourced from an API endpoint hosted on systems like [Open Weather Map Data](https://openweathermap.org/) or VEEVA Vault. By utilizing parameters such as city and country, we will call the API and fetch the data through a Python script.

The retrieved data will be dumped into an S3 bucket. Then, a Glue job will read the data from S3 and subsequently load it into Redshift.

## Architecture Diagram

![Architecture Diagram](images/image11.png)

We are not using **any crawler or Glue catalog tables** here. We are directly fetching the data from hive-like partition on a daily basis as a CSV file, applying transformation, and loading it to S3.

Tips: For incremental load or fetching only the new records, enable job bookmarks in the Glue job.

## Low-Level Design

Airflow will have two DAGs where dagB is dependent on dagA. DagA will fetch the data from the source through an API call, convert the JSON data, and write that as a CSV file in S3.

Once DagA completes, DagB will create a Glue job and run the Glue job. The Glue job will read, transform, and ingest the data into Redshift.

![Low-Level Design](images/image9.png)

### Why do we have two different DAGs?

Let's say if writing or dumping data to the Redshift table fails for some reason, does it mean fetching data from the API endpoint also fails?

So we will separate these two operations into two separate DAGs and will monitor or debug separately and execute only the failed DAG. Moving forward, any kind of backfilling or any ad-hoc run we want to perform, it is better to separate these two.

## CICD Workflow

Airflow DAG code and Glue script will get stored in S3, and this will be used to create DAG and Glue job respectively.

![CICD Workflow](images/image4.png)

We have one main branch in GitHub. We will cut a new branch named `dev` from `main` and commit all our local code to `dev`. From `dev`, we will raise a PR to merge the changes to the `main` branch. Once merged to `main`, the code build pipeline will get triggered, and deploy the changes in the AWS environment, and our application will be ready to run.

Note: When we create Glue via console, scripts and other details will get stored in the S3 folder automatically after creating the S3 folder automatically.

![S3 Folder Structure](images/image16.png)

Temporary folder, script folder, and sparkHistory logs folder are created automatically.

Here in our project, we will create an S3 folder and store the Glue scripts there and make use of those scripts to create the Glue job through the CICD pipeline. Similarly for Airflow DAG as well.

## Let's Get Started with the Implementation

### Step 1: API Source Account Creation

Create an account in [Open Weather Map](https://openweathermap.org/) and get an API key. At a later stage, since it is very sensitive data, we will be storing this in the Airflow variable. It is using this API key in the API scripts that we will be fetching the data from this API source.

This is how the response from the API endpoint looks like.

![API Response](images/image5.png)

### Step 2: Create a Redshift Connection in Glue Console

Create a Redshift connection in the Glue console for connecting to Redshift.

### Step 3: Configure and Make Necessary Changes

Configure and make necessary changes in the Glue script and Airflow DAG script provided.

We haven't yet created any table in Redshift or Glue job in AWS. All this will be done dynamically using the scripts itself.

Code from the Glue job script `weather_data_ingestion.py` file that is creating the Redshift table.

![Glue Script](images/image2.png)

Code from the Airflow DAG script in `transform_redshift_load.py` file that is creating the Glue job. This Glue job will contain the code from the above image or file named `weather_data_ingestion.py`. You can see in the below image we are mentioning the above code path in the script location while creating the Glue job through Airflow DAG code.

![Airflow DAG Script](images/image10.png)

### Step 4: Build a CodeBuild to Perform CICD

![CodeBuild](images/image3.png)

### Step 5: Commit All Files to GitHub

Commit all files to GitHub and raise a PR to `main` from `dev`, so that our codebuild will get started and run the `buildspec.yaml` file and copy the DAG and Glue scripts to corresponding S3 locations.

Once we merge the PR, the build gets started.

![Build Started](images/image12.png)

### Step 6: Create Airflow Environment

Follow the steps in this video to set up Amazon Managed Workflows for Apache Airflow.

Don't forget to add the `requirements.txt` file S3 path that we copied using `buildspec.yaml` while setting up the Airflow.

![Airflow Setup](images/image1.png)

Enable the public network and configure the security group, VPC, and subnets accordingly.

The path that you are mentioning for DAGS should be the same as the one you are using in the `buildspec.yaml` file, where you are copying the DAG script.

![DAG Path](images/image8.png)

Airflow DAGS get created.

![Airflow DAGS](images/image7.png)

Now configure the API security key and AWS default connection that we are using in the scripts for getting connected to API endpoint and AWS respectively.

Now enable the DAG. We have scheduled the DAG to run at midnight 12 AM (`schedule_interval="@once"`). For now, we can manually trigger the DAG.

The first DAG ran successfully, it fetched the data from the source, uploaded the data to S3 in CSV format, and triggered the next dependent DAG.

![DAG Run](images/image15.png)

Now our second DAG also gets triggered and the transform task (Create and execute Glue job) is running now.

![Transform Task](images/image13.png)

A new Glue job (`glue_transform_task`) is created and it is running.

![Glue Job Running](images/image14.png)

Finally, data is loaded into the Redshift table as we wanted.

![Data Loaded](images/image6.png)
