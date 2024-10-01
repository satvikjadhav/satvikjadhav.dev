---
title: "Data Lake"
date: 2022-05-07T18:39:03+05:30
# weight: 1
# aliases: ["/first"]
author: Satvik Jadhav
tags: ["data lake", "data"]
categories: ["data"]
ShowToc: true
TocOpen: false
---

### What is a Data Lake

*   Data lake is a central repository that holds big data from many different data sources
*   Can be structured, semi-structured or unstructured data
*   To ingest data as quickly as possible and make it available asap
*   Used extensively for machine learning and analytical solutions
*   Has to be secure and scale
*   Hardware should be inexpensive so you can store as much data as possible
*   Idea is to store as much data as possible so it can be made available to others, and they can make use of it later.
*   R&D on data products
*   Cannot always define the structure of the data

### Data Lake vs Data Warehouse

| Data Lake                                                  | Data Warehouse                                                  |
|------------------------------------------------------------|-----------------------------------------------------------------|
| Data is unstructured                                       | Data is structured                                              |
| Data Scientists or Data Analysts                           | Business Analysts                                               |
| Stores data on the scale of petabytes                      | Used for batch processing, Business Intelligence, and Reporting |
| Stream Processing, Machine learning and real time analysis | Data size is generally small                                    |
| Data is undefined, no relation between data                | Data Warehouses contain historic and relational data            |



### Gotchas of Data Lake

*   Starts with a good intention, but soon turns into a Data Swamp: Very hard to be useful  
    
*   No versioning: Incompatible schemas between the data, and Different file types
*   No metadata associated; what's the usefulness of the data
*   Joins not possible between different data sets

### ELT vs. ETL

| ELT                                                                       | ETL                                                                       |
|---------------------------------------------------------------------------|---------------------------------------------------------------------------|
| Extract, Load, and Transform                                              | Extract, Transform, and Load                                              |
| Used for large amount of data                                             | Used for small amount of data                                             |
| Provides data lake support (Schema on read)                               | Data warehouse solution                                                   |
| Data lake solution                                                        | Schema on write                                                           |
| Write the data first, and then determine the schema when loading the data | We define the schema, define the relationships, and then we load the data |


### Alternatives to components (S3/HDFS, Redshift, Snowflake etc.)

*   GCP -> Cloud Storage
*   AWS -> S3
*   Azure -> Azure Blob