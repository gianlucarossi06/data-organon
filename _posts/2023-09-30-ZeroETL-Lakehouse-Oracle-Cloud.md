---
title: Zero ETL Lakehouse in Oracle Cloud
description: Lakehouse solution in OCI with minimized or no ETL.
---

![](/data-organon/images/2023-09-30-ZEROETL-LAKEHOUSE-OCI/ZERO-ETL.jpg)

## **Introduction**

The **ET-L** (Extract Transform and Load) or **EL-T** (Extract Load and Transform) processes traditionally play a crucial role in data management solutions aimed at the creation, feeding and management of analytic data stores.

Data extraction involves extracting data from homogeneous or heterogeneous sources; data transformation processes data by data cleaning and transforming it into a proper storage format/structure for the purposes of querying and analysis; data loading describes the insertion of data into the final target data store, tipically a data warehouse, a data lake or a lakehouse.

Conventionally, at a high level, an ETL process schema can be designed as follows:

![Conventional ETL process ](/data-organon/images/2023-09-30-ZEROETL-LAKEHOUSE-OCI/conventional-etl.png)

But why do we need ETL processes?
Data are becoming more and more essential for business decisions. Insights derived from valuable information can have a very significant impact on the success of an organization. With the exponential growth of data volume, data sources, and data types, the ability to validate, cleanse, integrate, standardize, curate, and aggregate data in well- orchestrated and governed pipelines has achieved a valuable priority in the data management processes of organizations.

But ETL has also some drawbacks.
Here are some common drawbacks associated with the ETL process:

- **Complexity**: ETL processes can be complex, involving multiple steps and transformations. Designing and implementing these processes can be time-consuming, especially for large and complex data sets.
- **Data Latency**: ETL processes are usually scheduled to run at specific intervals (e.g., nightly batches). This means that the data in the target data store might sometimes have a latency not suited for the business needs.
- **Maintenance Challenges**: ETL processes require ongoing maintenance. As data sources change or evolve, ETL processes need to be updated accordingly. This maintenance can be complex, especially in large organizations with numerous data sources.
- **Dependency on Source Systems**: ETL processes depend on the structure and format of the source data. If the source systems change their data formats or schemas, it can break the ETL processes, requiring significant modifications.
- **Security Concerns**: ETL processes involve moving data between systems, which can raise security concerns. Ensuring data security and integrity during this process is crucial, and any breach can have severe consequences.
- **Difficulty in Error Handling**: ETL processes can encounter errors due to various reasons such as network issues, data inconsistencies, or system failures. Handling errors robustly and ensuring data integrity in the face of errors can be challenging.

That's why in recent years, technologies such as real-time data integration and data lakes/lakehouse have been developed to address some of these limitations, providing alternatives or supplements to traditional ETL processes.

## Zero ETL (or No-ETL) approach

A "Zero ETL" process is a data integration approach that aims to simplify and streamline the traditional ETL process by reducing or eliminating some of its components. The idea is to minimize the complexity of extensive ETL pipelines by leveraging different technologies (singularly or a mix of them), such as:

- **Real-Time Data Integration:** In a Zero ETL process, the emphasis is on real-time or near-real-time data integration. Instead of periodically extracting data from source systems, transforming it, and loading it into a data store as in traditional ETL, data is made available for analysis as soon as it's generated or updated in source systems.
- **Direct Data Access:** Zero ETL often involves direct access to source systems and data streams. Through **query federation** or **data virtualization**, instead of waiting for batch ETL jobs to extract data, users can access data directly from source systems or intermediate data layers, reducing latency and providing access to the most current data.
- **Data Transformation on the Fly:** Rather than performing extensive data transformation as a separate step, a Zero ETL process often relies on on-the-fly transformations. This means that data transformations are applied at the time of query or analysis, reducing the need for pre-processing and data redundancy.
- **Event-Driven Architecture:** Many Zero ETL systems are event-driven, meaning they respond to changes in data sources as they happen. This allows for real-time data ingestion.
- **In-Memory Query Accelerators**: In-memory query accelerators are technologies that store and process data primarily in memory to accelerate complex queries. Many of them offer automatic integration with operational data stores, thus eliminating the need for ETL duplication.

Clearly, Zero ETL approach introduces challenges as well (I will discuss them later on in this post). Anyhow, it might be a valid approach to be evaluated as a substitute (or a supplement) to the traditional ETL, carefully balancing its challenges with the benefits that Zero ETL brings to the solution.

## Zero ETL Lakehouse Architecture in OCI

Leveraging Oracle Cloud services for Analytical Data Platform, you can build Lakehouse solutions that combine the abilities of a data lake and a data warehouse to process a broad range of enterprise and streaming data for business analysis and machine learning (please visit [Oracle Architecture Center](https://https://docs.oracle.com/en/solutions/data-platform-lakehouse/index.html#GUID-A328ACEF-30B8-4595-B86F-F27B512744DF) for a complete description of the reference architecture).

For a OCI Lakehouse solution in the context of Zero ETL, I will focus on specific capabilities and features of some of services descripted in the reference architecture.
Let's consider the following simplified logical architecture:

![Initial Scenario - Potential Data Sources](/data-organon/images/2023-09-30-ZEROETL-LAKEHOUSE-OCI/initial-scenario-zeroetl-lakehouse-oci.png)

It shows many potential data sources for an OCI Analytical Data Platform that is based on Oracle Autonomous Database as data server engine.
What are the features of the **OCI Data Platform** that you could use to leverage those sources for your analytics needs following a **Zero ETL approach**?

Let's begin with minimizing data movement and latency:

![OCI Lakehouse - Minimizing data movement and latency capabilties](/data-organon/images/2023-09-30-ZEROETL-LAKEHOUSE-OCI/extract-scenario-zeroetl-lakehouse-oci.png)

* **Direct Data Access**: you can query data sources directly with:
  
  * **Oracle-Managed Heterogeneous Connectivity**: allows you to easily create database links to non-Oracle databases. When you use database links with Oracle-managed heterogeneous connectivity, Autonomous Database configures and sets up the connection to the non-Oracle database. You can then leverage all the **built-in analytical capabilities** of Oracle Autonomous Database (machine learning, graph, spatial, pattern matching, analytic views, text analytics) to analyze data of **Non-Oracle databases**.
  * **Cloud Links**: Cloud Links provide a cloud-based method to remotely access read only data on an Autonomous Database instance. With Cloud Links the data owner registers a table or view for remote access for a selected audience defined by the data owner. The data is then instantaneously accessible by everybody who got remote access granted at registration time. Whoever is supposed to see and access those data will be able to **discover** and **work with the data** made available to them.
  * **Autonomous Data Sharing**: Oracle Autonomous Database supports the **Delta Sharing protocol** as a Data Provider and a **Data Recipient**, enabling secure and seamless data exchange with Oracle and external non-Oracle systems
  
  In addition, the OCI Lakehouse based on Autonomous Data Warehouse allows to connect to external **Object Storage** (OCI Object Storage, AWS S3, Azure Blob Storage, Google Cloud Storage) to directly **query data** stored in many **different types of files** (csv, AVRO, Parquet, ORC, JSON). This enables schema-on-read approach that can highly increase the agility and the flexibility of the Zero ETL solutions.
* **Data Federation**: We have files in Object Storage, links to objects in heterogeneus systems, different data share end-points, and we have **Autonomous Data Warehouse**, the engine that can query all those different sources and **federate the results** in a single output.
* **Event-Driven Ingestion**: event-driven, real-time data ingestion and processing are enabled by Oracle GoldenGate. You can capture only the changes in data sources and replicate it to the target (Autonomous Data Base or Object Storage in this case). Source and target systems are aligned in real-time minimizing data movement and data latency.

So far we have access to quite a complete bunch of data sources with no or minimal data latency and movement and we can query them. That's good. We've managed to minimize the *Extract* part but, what about the *Transform* one? Let's add some functionality to the architecture design:

![OCI Lakehouse - Minimizing physical transformations](/data-organon/images/2023-09-30-ZEROETL-LAKEHOUSE-OCI/full-scenario-zeroetl-lakehouse-oci.png)

Well, we know that *Transform* should be minimized and on-the-fly as much as possible to please a Zero ETL approach. In that perspective, we can leverage the capabilties highlighted in the picture above, namely:

* **Analytics Views in Oracle Autonomous Data Warehouse**: with Autonomous Data Warehouse Studio you can visually create **Analytics Views** (alternatively, you can manually create Analytics Views with the related Oracle SQL DDL statements). They are implemented as materialized views and provide a way to simplify complex SQL queries, especially those involving **analytical and reporting functions**. They can involve multiple tables and can join them to provide a comprehensive dataset for analysis. This is particularly useful in a Zero ETL approach, when you need to **analyze** data from **different sources** or different parts of your database schema.
* **Oracle Analytics Cloud Semantic Model**: Oracle Analytics Cloud provides a full set of capabilities to explore and perform collaborative analytics (visualizations, dashboarding, self-service analytics, augmented analytics, self-service data preparation, and others). With OAC you can also build a **semantic model**. A semantic model is a metadata model that contains physical database objects that are abstracted and modified into logical objects. In a semantic model you can create a logical structures (combining and joining different physcal data structures), create logical calculations and logical aggregations. A semantic model then acts like a translation layer between your logical/semantic objects and your underlying data structures.

I haven't mentioned yet **Data Transform** and **Stream Processing**, highlighted in yellow... 



