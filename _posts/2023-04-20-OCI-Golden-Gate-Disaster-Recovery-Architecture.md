---
title: "OCI Golden Gate Disaster Recovery Architecture"
date: 2023-04-20
---


# OCI GoldenGate Disaster Recovery Architecture - Part I

A disaster recovery solution is essential to prevent severe loss of data, which can have a serious financial impact and also result in loss of customer confidence, damaging a company's reputation. As data and analytics platforms are more and more perceived as systems whose failure has those critical effects for the business, it is not uncommon that customers need a disaster recovery solution not only to meet the requirements of specific internal or external regulation but also to ensure that data consumers (including critical systems like, for instance, customer-facing applications or anti-fraud systems) can access the service without interruption, whatever may happen.
Lakehouse architecture combines the abilities of a data lake and a data warehouse to provide an innovative data platform that processes any type of data from a broad range of enterprise data resources. You can use this architecture to leverage the data for business analysis, machine learning, and data services.
As such, this architecture includes many service types that allow you to cover all the functionality and features of the data platform: upload, process, persist, analyze, and share data. 
From the point of view of the Lakehouse ecosystem, a disaster recovery architecture deployment must therefore take into consideration from time to time which services must guarantee continuity in the event of a critical failure. Surely you must consider the core components of the persistence layer (i.e., Object Storage and Autonomous Data Warehouse) but you may also need to include the data ingestion and processing services (e.g.: OCI Data Integration, OCI GoldenGate, Oracle Data Integrator, OCI Data Flow) and the data access and interpretation services (e.g., Oracle Analytics Cloud, OCI Data Science, APEX and Oracle Rest Data Services) that are integral parts of the overall solution.

![Fig.1: Sample of physical deployment of Lakehouse with Disaster Recovery](images/OCIGGDR_InitialConfiguration.png)

As a first use case, I analyze a disaster recovery solution for OCI GoldenGate used to replicate data to a target Autonomous Data Warehouse.
OCI GoldenGate is a fully managed, native cloud service that moves data in real-time, at scale. OCI GoldenGate processes data as it moves from one or more data management systems to target systems. You can also design, run, orchestrate, and monitor data replication tasks without having to allocate or manage any compute environments.
In this scenario, OCI GoldenGate extracts data from an Oracle DB and replicates to a target Oracle Autonomous Data Warehouse running on OCI.
To meet Disaster Recovery requirements, another Autonomous Data Warehouse has been configured in a different OCI Region as a StandBy database, aligned by enabling Autonomous Data Guard in the primary ADW.
To provide a full Disaster Recovery solution, also OCI GoldenGate must have a deployment in the other OCI Region ready to ingest data to the Oracle StandBy Database (become Primary)  whenever a disaster or a failure in the primary region occurs.

This first post shows how to configure OCI GoldenGate in a Disaster Recovery solution architecture. The second part of the post goes through the details of the operational tasks to be executed in case of disaster.

## OCI GG DR Architecture Enablement

### Initial configuration: OCI GoldenGate single deployment, ADW with Autonomous Data Guard

The initial configuration is similar to the one described in OCI GoldenGate Live Labs ["Replicate Data Using Oracle Cloud Infrastructure GoldenGate"][https://apexapps.oracle.com/pls/apex/dbpm/r/livelabs/view-workshop?wid=797]. OCI GoldenGate extracts data from an Oracle DB and moves it to a target Oracle Autonomous Data Warehouse on OCI. The Oracle DB used as a source in this example is an Oracle ATP provisioned in a different OCI Region (RegionC _eu-amsterdam-1_, not affected by the Disaster Recovery in this example) but, in reality, it could be any other Database, whether in OCI or on-premises. The initial configuration also includes Autonomous Data Guard for the target Autonomous Data Warehouse database which enables a "StandBy" database in another OCI Region (RegionB - _eu-frankfurt-1_)

_Fig.2: Initial configuration: target ADW with Autonomous Data Guard enabled and OCI GoldenGate with single deployment_

#### RegionA (_eu-milan-01_) initial configurations

1) OCI GoldenGate deployment:

Deployment Name: _ocigg-RegionA_
Assigned Connections:

- _SourceDB_, connection to the Oracle DB (ATP database in a different OCI Region) that has the data to be replicated to Oracle ADW in RegionA.
- _TargetOCIDB_, connection to the Oracle ADW database that is the target of the replicat process
