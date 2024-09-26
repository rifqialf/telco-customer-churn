# Telco Customer Churn 
Scenario: A major telecommunication company in California, US, is experiencing a concerning rate of customer churn, particularly in certain counties. To address this issue and improve customer retention, the company has decided to develop a comprehensive dashboard that provides actionable insights into customer churn trends.

## Introduction
### Project Requirements
The goal is to create a churn analysis dashboard for a telecom company, showing customer behavior, reasons for churn, and churned revenue across regions. Key metrics include:
1. **Churn Rate** - Percentage of customers who canceled service.
2. Average **Monthly Recurring Revenue (MRR)** for churned customers.
3. **Churned Service Plans** - Service types associated with churn.
4. **Churn Reasons** - Breakdown of why customers left (e.g., price, dissatisfaction, competitors).
5. **Demographics** - Gender, marital status, and age group correlations with churn.
6. **Customer Tenure** - How contract length influences churn.

The visual is to include interactive map with dynamic filter, as well as complementary charts and tables when necessary.

### Dataset
The customer churn dataset is retrieved from [this Kaggle page](https://www.kaggle.com/datasets/aadityabansalcodes/telecommunications-industry-customer-churn-dataset). Five CSV files were used in this project:
1. Telco_customer_churn_status.csv - Customer churn-related information such as churn labels (whether a customer has churned or not), churn scores, and churn reason.
2. Telco_customer_churn_services.csv - Details about the telecom services customers are subscribed to, such as phone, internet, TV, and additional services.
3. Telco_customer_churn_location.csv - Geographic information such as zip code and coordinates, potentially giving insights into how customer churn varies by region.
4. Telco_customer_churn_demographics.csv - Demographic information such as age, gender, and other customer classifications.
5. Telco_customer_churn_population.csv - Data about population in the regions.

The exploratory data analysis (EDA) of the dataset can be explored at the Kaggle page, which are put into consideration but no extra action is deemed necessary for this project. A small modification were made to the customer churn dataset: A part of the original Q3 data is slightly altered, moved into separate files, and then labeled as Q4 data. This is to allow a showcase of how the developed data pipeline accepts new data, update existing data, and apply the updates into the dashboard as end result.

Below are data model of the customer churn data:
<img width="1024" alt="Customer Churn Data Model" src="https://github.com/user-attachments/assets/1a067ad8-8b74-4636-afbc-894b0dd03b7c">

Another dataset is shapefile of California county boundaries which was retrieved from [California Open Data Portal](https://data.ca.gov/dataset/ca-geographic-boundaries).

## Solution Architecture
### Process: Azure Databricks
Azure Databricks is well-suited for batch processing of this project size. It offers scalable, distributed computing through Spark, which can handle complex ETL (Extract, Transform, Load) tasks efficiently.

This Azure Databricks (as well as other Azure resources / services) leverages Pay-As-You-Go subscription with specified region _UK South_, which was selected as one of best CPU decisions during author's time in Europe in terms of resource availability (which is not the case anymore, unfortunately making UK South a costly decision for the project). Three budget alerts were set to anticipate so that the subscription is not exceeding $20 as shown below.

![Budget Alerts](https://github.com/user-attachments/assets/f4eaa77c-55c9-433a-8188-a42621350d85)

The ADLS access security to the Databricks is **Managed Identity** using Azure Access Connector for Databricks, which is more convenient security access to allow scalable project compared to other accesses (access keys, SAS token, and service principal).

In the workspace, a cluster was configured as shown below:

<img width="1024" alt="Databricks Cluster Configuration" src="https://github.com/user-attachments/assets/678e172d-132b-4d2c-b49f-86da79cb3ec3">
 
Justification for the configuration are listed below:
1. Single user of Single Node - the most suitable configuration in terms of cost and workload for the project.
2. 15.4 LTS runtime - latest runtime option that enables Unity Catalog to be used in the workspace.
3. Standard_DS3_v2 with 4 Cores - for cost-wise decision suitable with the subscription's CPU quota.
4. Auto Termination: 20 minutes - to quickly terminate the cluster and to minimize cost.
5. Cluster Policy: Unrestricted - cluster policy is not necessary as only one cluster is being created in this project.

### Storage: Azure Data Lake Storage Gen2 + Delta Lake + Unity Catalog
Azure Data Lake Storage Gen2 provides scalable and secure storage for massive datasets, with high throughput and optimized performance through its hierarchical namespace. The ADLS is treated as Delta Lake to improve data reliability and performance by offering ACID transactions, schema enforcement, and time travel capabilities, which are crucial for handling evolving datasets. Meanwhile, Unity Catalog allows for easier management of data access policies across different teams while ensuring consistent and secure data usage across all Databricks workspaces.

In this project, two Delta Lakes are used:
1. Project ADLS Storage Delta Lake: The CSV files of telecom customer information are stored in the Delta Lake in the bronze layer.
2. External ADLS Storage Delta Lake: California county boundaries' shapefile data are stored in and accessed via Unity Catalog's external volumes.

The setup above is to showcase how the Azure Databrick workspace can ingest data from multiple (external) Delta Lake. Both Delta Lakes were setup with Access Connector for Databricks (namely managed identity) that were assigned as "Storage Blob Data Contributor" to let the ADLSs and the corresponding Unity Catalog to be used in the Databricks workspace.

***-- ADLS Credential Illustration - Which services access which services --***

This project adopts **Medallion Architecture**: structuring the data flow into three layers—bronze, silver, and gold—where raw data is first ingested into the bronze layer, refined in the silver layer, and served as analytics-ready datasets in the gold layer. This architecture ensures cleaner and organised structure of data in each phase.

### Orchestration: Azure Data Factory
Azure Data Factory is a great choice for dealing with periodic data ingestion and processing. It allows you to build pipelines that automate the ETL process, integrate data, and execute Azure Databricks notebooks on a scheduled basis. Linked services were created to connect the Databricks workspace (using managed identity) and the two ADLSs (using simple access keys) to allow the pipeline to run accordingly. 

### Visualization: Power BI
Power BI is an excellent visualization tool for presenting advanced and interactive churn insights to stakeholders especially with its ability to easily connect to Azure-based solutions. In this project, Unity Catalog tables from Azure Databricks were imported (without DirectQuery) into Power BI Desktop using Personal Access Key.

## Data Engineering
In the Databricks workspace, the project codebase is organised into 5 folders:
1. Ingestion - Contains scripts for extracting and loading data from sources.
2. Transformation - Includes code for cleaning, transforming, and structuring the ingested data to prepare it for analysis.
3. Presentation - Focuses on presenting the processed data.
4. Includes - Holds reusable modules or utility functions that are shared across different scripts.
5. Setup - Contains configuration scripts or initialization code needed to set up the environment.

***-Picture of folders-***


### Data Ingestion
The ingestion folder mainly contains Python-based notebooks to ingest each of the 5 raw files. One master notebook runs all of the ingestion notebooks in sequence (not in parallel as to not overload the cores, although it is capable of doing parallel ingestion in this project).

The main steps of data ingestion for all files are similar:
1. Create schema using StructField and StructType. This approach seems to provide tidier code and easy to maintain once the schema evolves.
2. Read the raw file as PySpark Dataframe using the built schema. 
3. Perform small transformations to the data such as removing unuseful columns, renaming columns for consistency, and add *ingestion_date* column.
4. Write the processed data as silver managed tables using function to create new delta table or upsert the data into existing table. Managed table for silver layer data is decided to allow simplified management and built-in data governance features, such as data quality checks and lineage tracking.

Several points to point out:
1. The performed transformations in this phase is determined as relatively small transformations, so that it is okay to be done during ingestion phase.
2. The *configuration* notebook is executed in all ingestion notebooks to provide pre-defined variables of bronze, silver, and gold folder paths in the project's ADLS.
3. The *common_functions* notebook is executed in all ingestion notebooks to provide pre-built functions: adding *ingestion_date* and write data.
4. A parameter *file_date* must be provided to the Notebook widgets created with dbutils before executing the notebooks, which should match with folder name of the raw data. This parameter will be especially useful in the orchestration in Azure Data Factory.


***-Picture of raw data to ingest-***
***-Picture of ingsestion notebooks-***
***-Picture of ingestion result-***


### Data Transformation
The transformation folder contains two transformation notebooks and one master notebook to run the two in sequence. 
1. Filter churned customer - using SQL semi-join to Status silver table to only get churned customer data from all tables excluding Population table (not required for the requirement).
2. Intersect with county - using Geopandas (installed in cluster) to perform spatial join between the silver layer location table with county shapefile data.

The result from this phase are tables into the gold layer, ready for presentation. 

***-Picture of transformation notebooks-***
***-Picture of geopandas installed in cluster-***


## Azure Data Factory (ADF)
Each data ingestion and data transformation have their own pipeline. Each of those pipelines use master notebook to run.

<img width="260" alt="ADF Pipelines and Datasets" src="https://github.com/user-attachments/assets/5ecda3e0-63f6-4143-b4ee-1326c3e8e9bc">

One master pipeline run both in sequence: *Execute Ingestion --> Execute Transformation*. The setup is designed in a way to make transformation run after ingestion has completed. 

<img width="480" alt="ADF Pipelines Trigger" src="https://github.com/user-attachments/assets/b59a7f4e-4e93-496e-8903-66fff9c62830">

A Schedule Trigger *monthly_customer_churn* was set to run the master pipeline every 5th day of new month, during which the pipeline will look for folder in the bronze directory that are named with the date of the run e.g. *2024-09-05* or *2025-01-05*. In the data ingestion pipeline, a logic is created. The trigger takes off at 20:00 to ensure no other processes that might overlap.

<img width="480" alt="ADF Pipelines Trigger" src="https://github.com/user-attachments/assets/ff37f4f4-19b5-412b-819c-1b9b2a9ce635">

In data ingestion pipeline, it starts with using *Get Metadata* activity to read the trigger and pipeline parameter for the run date. An if-condition waits for that activity to be completed, and look into the project's ADLS for the aforementioned raw data folder and check whether a folder named with the run date exists. If exists, activity to run data ingestion master notebook will start, otherwise an activity to return 404 error will run. Meanwhile, transformation pipeline only contains activity to run transformation master notebook.

<img width="480" alt="ADF Ingestion Pipelines" src="https://github.com/user-attachments/assets/cbda79a4-7355-4d0b-a5c9-f417f07989c1">

<img width="240" alt="ADF Transformation Pipelines" src="https://github.com/user-attachments/assets/f7f377be-1bf2-4ee3-a351-8a22e8745974">

## Power BI

Table transformation:
- Explain service unpivot - transform data
- Explain measures using DAX
The end result visualizations
- Explain all are interconnected to each other based on data relationships
- Explain each visual + all used columns
- Explain edit interaction
What insights to be extracted:
- When click on a region:
    - Get all demographics info of people who churned
    - etc.



After tables are stored in gold layer from Azure Databricks, they are imported to Power BI via providing Databricks Server Hostname, HTTP path, and Personal Access Key.

<img width="720" alt="Power BI data import" src="https://github.com/user-attachments/assets/f7e1c30e-2f59-4ae7-a69f-14a9555e4e7f">
<img width="360" alt="Databricks Server Hostname and HTTP path" src="https://github.com/user-attachments/assets/ddd7c6f6-3a5d-492e-9867-94c1b83eadfb">

The data is modelled using Star Schema, which is a relatively scalable and maintainable approach compared to normalized table.
<img width="360" alt="Star schema" src="https://github.com/user-attachments/assets/ee453f17-46aa-4970-aa1b-9e922fd1ec76">

Several modifications are shown in the Star Schema figure above.
1. Unpivotted Service Table - This table is result of unpivoting service_churn table based on all columns regarding service plan types e.g. Internet Type, Tech Support, and Streaming services.
2. 




## Cost Analysis
Set up cost
Reason
- UK South
- Trial on master notebook and ADF


![costanalysis_charts](https://github.com/user-attachments/assets/d90e33b1-25c7-4900-b8fe-86ca08a7ddf3)

