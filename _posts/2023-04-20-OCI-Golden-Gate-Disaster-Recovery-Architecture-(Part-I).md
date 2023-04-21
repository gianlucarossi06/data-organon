## **Introduction**

A disaster recovery solution is essential to prevent severe loss of data, which can have a serious financial impact and also result in loss of customer confidence, damaging a company's reputation. As data and analytics platforms are more and more perceived as systems whose failure has those critical effects for the business, it is not uncommon that customers need a disaster recovery solution not only to meet the requirements of specific internal or external regulation but also to ensure that data consumers (including critical systems like, for instance, customer-facing applications or anti-fraud systems) can access the service without interruption, whatever may happen.

Lakehouse architecture combines the abilities of a data lake and a data warehouse to provide an innovative data platform that processes any type of data from a broad range of enterprise data resources. You can use this architecture to leverage the data for business analysis, machine learning, and data services.

As such, this architecture includes many service types that allow you to cover all the functionality and features of the data platform: upload, process, persist, analyse, and share data. 

From the point of view of the Lakehouse ecosystem, a disaster recovery architecture deployment must therefore take into consideration from time to time which services must guarantee continuity in the event of a critical failure. Surely you must consider the core components of the persistence layer (i.e., Object Storage and Autonomous Data Warehouse) but you may also need to include the data ingestion and processing services (e.g.: OCI Data Integration, OCI GoldenGate, Oracle Data Integrator, OCI Data Flow) and the data access and interpretation services (e.g., Oracle Analytics Cloud, OCI Data Science, APEX and Oracle Rest Data Services) that are integral parts of the overall solution.

![Fig.1: Sample of physical deployment of Lakehouse architecture with Disaster Recovery](/images/2023-04-20-OCI-GG-DR-Part-I/6524352535.png)

As a first use case, I analyze a disaster recovery solution for OCI GoldenGate used to replicate data to a target Autonomous Data Warehouse.

OCI GoldenGate is a fully managed, native cloud service that moves data in real-time, at scale. OCI GoldenGate processes data as it moves from one or more data management systems to target systems. You can also design, run, orchestrate, and monitor data replication tasks without having to allocate or manage any compute environments.

In this scenario, OCI GoldenGate extracts data from an Oracle DB and replicats to a target Oracle Autonomous Data Warehouse running on OCI.

To meet Disaster Recovery requirements, another Autonomous Data Warehouse has been configured in a different OCI Region as a _StandBy_ database, aligned by enabling Autonomous Data Guard in the primary ADW.

To provide a full Disaster Recovery solution, also OCI GoldenGate must have a deployment in the other OCI Region ready to ingest data to the Oracle _StandBy_ Database (become Primary)  whenever a disaster or a failure in the primary region occurs.

This first post shows how to configure OCI GoldenGate in a Disaster Recovery solution architecture. The second part of the post goes through the details of the operational tasks to be executed in case of disaster.

## OCI GG DR Architecture Enablement

### Initial configuration: OCI GoldenGate single deployment

The initial configuration is similar to the one described in OCI GG Live Labs [_Replicate Data Using Oracle Cloud Infrastructure GoldenGate_](https://apexapps.oracle.com/pls/apex/dbpm/r/livelabs/view-workshop?wid=797)_._ OCI GoldenGate extracts data from an Oracle DB and moves it to a target Oracle Autonomous Data Warehouse on OCI. The Oracle DB used as a source in this example is an Oracle ATP provisioned in a different OCI Region (RegionC _eu-amsterdam-1_, not affected by the Disaster Recovery in this example) but, in reality, it could be any other Database, whether in OCI or on-premises. The initial configuration also includes **Autonomous Data Guard** for the target Autonomous Data Warehouse database which enables a "StandBy" database in another OCI Region (_RegionB_ - _eu-frankfurt-1_)

![Fig.2: Initial configuration: target ADW with Autonomous Data Guard enabled and OCI GoldenGate with single deployment](/images/2023-04-20-OCI-GG-DR-Part-I/6050731718.png)

#### RegionA (_eu-milan-01_) initial configurations

**1) OCI GoldenGate deployment:**

*   **Deployment Name**: _ocigg-RegionA_
*   **Assigned Connections**:
    *   _SourceDB_, connection to the Oracle DB (ATP database in a different OCI Region) that has the data to be replicated to Oracle ADW in _RegionA_.
    *   _TargetOCIDB_, connection to the Oracle ADW database that is the target of the replicat process

         ![](/images/2023-04-20-OCI-GG-DR-Part-I/6276959885.png)

         ![Fig.3: RegionA, connections assigned to OCI GoldenGate deployment](/images/2023-04-20-OCI-GG-DR-Part-I/6276959907.png)

*   **Deployment backup**: Only OCI GG automatic backup
*   **Processes**: This deployment has one extract (_UAEXT_) process that extracts data from the _SourceDB_ and one replicat process (_REP_) that replicates the data to the _TargetOCIDB._

       ![Fig.4: RegionA: OCI GoldenGate extract and replicat process](/images/2023-04-20-OCI-GG-DR-Part-I/6276960234.png)

**2) Autonomous Data Warehouse:**

*   **Instance Name**: _OCIDBTARGET01_
*   **Autonomous Data Guard:** the primary instance has a remote _Standby_ database (_OCIDBTARGET01\_Remote)_ in Region B (_eu-frankfurt-1_)

         ![Fig.5: RegionA, Autonomous Data Guard enablement](/images/2023-04-20-OCI-GG-DR-Part-I/6050737112.png)

         ![Fig.6: RegionA, Autonomous Data Warehouse Primary instance for disaster recovery](/images/2023-04-20-OCI-GG-DR-Part-I/6050737123.png)

         ![Fig.7: RegionB, Autonomous Data Warehouse Standby instance for disaster recovery](/images/2023-04-20-OCI-GG-DR-Part-I/6275398387.png)

## Configuration steps to enable Disaster Recovery for OCI GoldenGate

At high level, the OCI GoldenGate disaster recovery configuration leverages a cross-site replication of OCI GoldenGate backup files to share the OCI GoldenGate configuration among different deployments. OCI GG backup file stored in a ObjectStorage bucket in _RegionA_ is replicated by OCI ObjectStorage cross-site replication to a OCI Object Storage bucket in _RegionB_. From there it is made available as a backup file ready to be restored by the OCI GoldenGate deployment in _RegionB_.

Logical architecture that shows the configurations and components that enable the OCI GG disaster recovery solution:

![Fig.8: Disaster recovery architecture including OCI GoldenGate](/images/2023-04-20-OCI-GG-DR-Part-I/6050731718.png)

### RegionA (_eu-milan-01_) configurations

**1) Create Object Storage Bucket to store OCI GG backup files: Y**ou need to use an object storage bucket to store the OCI GG backup files to be replicated in the secondary OCI Region. You create a bucket (_GG-backup-RegionA)_ with a standard tier.

![Fig.9: Bucket to store OCI GoldenGate manual backups](/images/2023-04-20-OCI-GG-DR-Part-I/6275381367.png)

**2) Enable OCI GoldenGate manual backup:** the solution is based on the possibility to share OCI GG backup files between OCI GG deployments that are in different OCI Regions. As a standard, OCI GoldenGate deployments perform an automatic backup once a day. You can use those backups to restore your deployment, but you don't have direct access to the related backup files. You need to manually create an OCI GG backup in order to be able to save and manage the related files in an Object Storage bucket. You can do it through the OCI console, API, SDK or OCI-CLI commands. You can leverage an OCI Compute VM with Linux OS in which you install the oci-cli libraries (see [Oracle documentation](https://docs.oracle.com/en-us/iaas/Content/API/SDKDocs/cliinstall.htm) for details) and create a script to execute the OCI GG deployment backup command.

The oci-cli command is:

_oci goldengate deployment-backup create_

Mandatory parameter:

*   \-_\-bucket-name \[text\]_  
    (Name of the bucket where the object is to be uploaded in the object storage)

*   _\--compartment-id, -c \[text\]_  
    (The OCID of the compartment being referenced)

*   _\--deployment-id \[text\]_  
    (The OCID of the deployment being referenced)

*   _\--display-name \[text\]_  
    (An object’s Display Name. It is the name displayed in the OCI Console in the backup section of the OCI GG deployment)

*   _\--namespace-name \[text\]_  
    (Name of namespace that serves as a container for all of your buckets)

*   _\--object-name \[text\]_  
    (Name of the object to be uploaded to object storage. It’s the name of the file)


Connected to the Linux VM, you create a shell script with the oci-cli command to store the OCI GG deployment backup file _bkp-ocigg_ in the bucket _GG-backup-RegionA_

![Fig.10: Shell script example with the "oci-cli" command to store OCI GoldenGate manual backups in Object Storage](/images/2023-04-20-OCI-GG-DR-Part-I/6228856711.png)

Then you can schedule it with Linux crontab to be executed, for instance, every four hours:

![Fig.11: Scheduling of the shell script with Linux crontab](/images/2023-04-20-OCI-GG-DR-Part-I/6228856788.png)

**NOTE**: The OCI GG backup purpose is to save the configuration of the deployment. As such, you may want to schedule it less frequently, depending on the size of the backup. A good practice is also to generate a manual backup every time a configuration change is made, while maintaining the automatically scheduled backup for safety.

**3) Enable Object Storage cross-region replication policy:** You need to automatically replicate the OCI GG backup files to _RegionB_. You enable the Object Storage cross-region replication on the bucket _GG-backup-RegionA_ having as a target the bucket _Cross-GG-backup-RegionA_ previously created in _RegionB_. With this cross-region replication, whenever a new backup file of the _ocigg-RegionA_ deployment is stored in the bucket _GG-Backup-RegionA_ a corresponding file is replicated in the _Cross-GG-backup-RegionA_ bucket in _RegionB_.

![Fig.12: Object Storage bucket replication policy](/images/2023-04-20-OCI-GG-DR-Part-I/6209012559.png)

### Region B (_eu-frankfurt-01_) configurations

**1) Create OCI GoldenGate secondary deployment.** It's the OCI GG "Standby" deployment in _RegionB_. Initially it can be empty, with no extract or replicat process configured. In case of disaster in _RegionA_, it will restore the proper deployment configuration from the backup file of the of _RegionA_ OCI GG deployment.

**NOTE**: Once created, to save OCI credit consumption, you may consider turning the OCI GoldenGate secondary deployment off.

*   **Deployment Name**: _ocigg-RegionB_
*   **Assigned Connections**:
    *   _SourceDB_, connection to the same Oracle DB source for _RegionA_ (ATP database in _RegionC_), since it's the source for both Regions.
    *   _TargetOCIDB_, connection to the Oracle ADW database that will be the target of the replicat process in case of disaster. The target DB is the _Standby_ database of the ADW primary database in _RegionA._

         ![](/images/2023-04-20-OCI-GG-DR-Part-I/6275395167.png)

         ![Fig.13: RegionB, connections assigned to OCI GoldenGate deployment](/images/2023-04-20-OCI-GG-DR-Part-I/6275395178.png)

*   **Manual OCI GG backup**: You need to create a manual backup for this deployment to produce a backup file that can be overwritten by the backup files replicated from _RegionA_**.** First, you create the bucket _GG-backup-RegionB_ as the target bucket for backup files. Then, using the OCI Console, you can create the manual OCI GG backup (_ocigg-RegionB-manual_)_:_

        <!--[Fig.14: RegionB, first OCI GoldenGate manual backup](/images/2023-04-20-OCI-GG-DR-Part-I/6275381382.png)-->
           <img src="/images/2023-04-20-OCI-GG-DR-Part-I/6275381382.png" alt="Fig.14: RegionB, first OCI GoldenGate manual backup" width="250"/>

           ![Fig.15: RegionB, OCI GoldenGate backup](/images/2023-04-20-OCI-GG-DR-Part-I/6276947759.png)

As expected the file produced by the manual backup, named _bkp-ocigg_, is visible in the bucket _GG-backup-RegionB_:

          ![Fig.16: RegionB, OCI GoldenGate backup file in the Object Storage bucket](/images/2023-04-20-OCI-GG-DR-Part-I/6276947784.png)

**NOTE:** That file can be overwritten by the OCI GG backup files replicated from the _RegionA_ deployment.

*   **Processes**: Initially, the OCI GG secondary deployment in RegionB does not have any extract/replicat process.

          ![Fig.17: RegionB, OCI GoldenGate deployment without any extract/replicat process](/images/2023-04-20-OCI-GG-DR-Part-I/6275394992.png)

**2) Create Object Storage bucket for cross-region replicat from RegionA:** This bucket (_cross-GG-backup-RegionA_) is the target for the cross-region replicat of the bucket _GG-backup-RegionA_ in _RegionA_. 

**3) Deploy an OCI Function to copy files to the bucket _GG-backup-RegionB_:** To make _RagionA_ OCI GG backup files available to the OCI GG secondary deployment in _RegionB_, you need to automatically copy files from the bucket _cross-GG-backup-RegionA_ to the bucket _GG-backup-RegionB_. Since the files to be copied have the same name as the backup file manually created for OCI GG in _RegionB_, this operation will overwrite the backup file with the content of the backup files from the OCI GG deployment in _RegionA_.

As first step you need to create the application and deploy the Function, then you can create an Event Rule to trigger the Function when files are created or updated in the _cross-GG-backup-RegionA_ bucket.

Create the application (_a-os-replicat)_:

![Fig.18: RegionB, OCI Function application creation](/images/2023-04-20-OCI-GG-DR-Part-I/6275387637.png)


Then launch the OCI Cloud Shell and set up the fn CLI Cloud Shell:

*   Set the context for _RegionB_:

_fn use context eu-frankfurt-1_

*   Update the context with the function Compartment ID:

_fn update context oracle.compartment-id <my compartment OCID>_

*   Set a unique repository name prefix (_ggdr_) to distinguish my function images:


_fn update context registry [fra.ocir.io/frqap2zhtzbe/ggdr](http://fra.ocir.io/frqap2zhtzbe/ggdr)_

*   Generate an Auth Token using the OCI Console 
*   Log into the registry using the token you just generated:  

_docker login -u 'frqap2zhtzbe/oracleidentitycloudservice/myfirstname.mylastname@[oracle.com](http://oracle.com)' [fra.ocir.io](http://fra.ocir.io)_

For details about preparing the OCI environment to deploy an OCI Function see [Function Quick Start Guide](https://docs.oracle.com/en-us/iaas/Content/Functions/Tasks/functionsquickstartcloudshell.htm#functionsquickstart_cloudshell).

Then you need to deploy the function. You can start from a Function example present in the Oracle Github repository (the Function _oci-objectstorage-copy-objects-python_):

*   In the Cloud Shell clone the Github repository:

_git clone [https://github.com/oracle/oracle-functions-samples.git](https://github.com/oracle/oracle-functions-samples.git)_

*   Then go into the _samples/oci-objectstorage-copy-objects-python_ folder_:_

_cd oracle-functions-samples/samples/oci-objectstorage-copy-objects-python_

*   Edit the _func.py_ file to set the proper namespace and target bucket:

           ![Fig.19: RegionB, copy object OCI Function details](/images/2023-04-20-OCI-GG-DR-Part-I/6275387590.png)

**NOTE**: as an alternative, instead of fixed values, you may want to configure variables for both the namespace and the destination bucket referred in the Function.

*   Finally, you deploy the function in the "_a-os-replicat_" application:

_fn -v deploy --app a-os-replicat_

**4) Create an Event Rule to trigger the OCI Function:** You need to create an Event Service rule that triggers the OCI Function _oci-objectstorage-copy-objects-python_ whenever an object is created or updated in the bucket _cross-GG-backup-RegionA._

Create the rule _e-os-file-replicat_ with the following conditions:

<table class="wrapped confluenceTable"><colgroup><col><col><col></colgroup><tbody><tr><th class="confluenceTh">Condition</th><th class="confluenceTh"><span style="color: rgb(26,24,22);">Service/Attribute Name</span></th><th class="confluenceTh">Attribute Value</th></tr><tr><td class="confluenceTd">Event Type</td><td class="confluenceTd">Object Storage</td><td class="confluenceTd">Object-Create, Object-Update</td></tr><tr><td class="confluenceTd">Attribute</td><td class="confluenceTd">compartmentName</td><td class="confluenceTd"><em>&lt;my compartment name&gt;</em></td></tr><tr><td class="confluenceTd">Attribute</td><td class="confluenceTd">bucketName</td><td class="confluenceTd"><em>GG-backup-RegionB</em></td></tr></tbody></table>

And with this action:

<table class="wrapped confluenceTable"><colgroup><col><col><col><col></colgroup><tbody><tr><th class="confluenceTh">Action Type</th><th class="confluenceTh">Function Compartment</th><th class="confluenceTh">Function Application</th><th class="confluenceTh">Function</th></tr><tr><td class="confluenceTd">Functions</td><td class="confluenceTd"><em>&lt;my compartment name&gt;</em></td><td class="confluenceTd"><span><em>a-os-replicat</em></span></td><td class="confluenceTd"><span><span style="color: rgb(22,21,19);"><em><span style="color: rgb(26,24,22);">oci-objectstorage-copy-objects-python</span></em></span></span></td></tr></tbody></table>



Creation of the Event Service rule:

![Fig.20: RegionB, event rule configuration](/images/2023-04-20-OCI-GG-DR-Part-I/6283046930.png)

Whenever an OCI GG backup file is replicated (through Object Storage Cross-Region Replication) from the bucket _GG-backup-RegionA_ in _RegionA_ to the bucket _cross-GG-backup-RegionA_ in _RegionB,_ this rule triggers the OCI Function that copies the file in the bucket _GG-backup-RegionB_, ready to be restored by the OCI GG secondary deployment.

## Additional Notes

### Regions specular configuration

In this post, I start with the primary region (_RegionA_) configurations, and then I describe the secondary region (_RegionB_) ones. Configurations differ, but in reality, primary region and secondary region should have specular configurations because primary and secondary are only temporary concepts. A primary region can switch to secondary region, and vice versa, at any time.

Therefore you should consider to add:

In _RegionB_:

*   A manual backup for the OCI GG deployment ready to be scheduled once the deployment becomes primary.
*   An Object Storage cross-region replication to replicate the OCI GG _RegionB_ manual backup files to _RegionA._ This should only be enabled after _RegionB_ becomes primary.

In _RegionA_:

*   An Object Storage bucket to be used as a target for cross-region replication from RegionB (e.g. a bucket named c_ross-GG-backup-RegionB)_
*   An OCI Function to copy the files from the cross-region backup bucket to the bucket that stores the manual backup files (_GG-backup-RegionA_)
*   An Event Rule that triggers the OCI Function to automatically copy the backup files from the cross-region backup bucket to the bucket _GG-backup-RegionA._

The final complete target architecture looks like the picture below:

![Fig.21: Full primary/secondary Region architecture configuration](/images/2023-04-20-OCI-GG-DR-Part-I/6284983879.png)

### Active secondary OCI GG deployment

In this post, I suggest keeping off the secondary OCI GG deployment to save Oracle Cloud credits. But if the RTO of the solution is a priority, you may want to keep active the OCI GG secondary deployment (eventually scaled down) and automatically restore a backup file whenever a new backup is copied in the bucket _GG-backup-RegionB._ To do this, you need to create an OCI Function that executes an OCI GG restore and an Event Rule that triggers that Function when an object create/update event occurs on the bucket _GG-backup-RegionB._
