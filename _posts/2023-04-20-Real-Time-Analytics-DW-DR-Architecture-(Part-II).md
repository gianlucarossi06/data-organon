---
title: Real Time Analytics DW DR Architecture (Part II)
<!--description: Recovery Operations.-->
---

![](/data-organon/images/2023-04-20-OCI-GG-DR-Part-II/DISASTER-RECOVERY-2.jpg)

## **Introduction**

In the first part of the blog [_Real Time Analytics DW DR Architecture (Part I)_](https://gianlucarossi06.github.io/data-organon/2023/04/20/Real-Time-Analytics-DW-DR-Architecture-(Part-I\).html), I described how to enable a disaster recovery architecture for a Real-Time Data Warehouse solution in OCI as depicted in the picture below:

![Fig.1:  OCI GoldenGate disaster recovery architecture](/data-organon/images/2023-04-20-OCI-GG-DR-Part-II//6310899344.png)

In summary, with this architecture configuration, you have:

In _RegionA:_

*   An ADW primary instance (_OCIDBTARGET01_) that, with Autonomous Data Guard, continuously aligns the _Standby_ remote instance _OCIDBTARGET01\_Remote_ in _RegionB._
*   An OCI GG deployment (_ocigg-RegionA_) up and running that replicates data to the target ADW (_OCIDBTARGET01_).
*   An oci-cli command scheduled on a Linux VM that periodically creates a manual backup of the OCI GG deployment and stores the backup files in the bucket _GG-backup-RegionA_.
*   The OS bucket _GG-backup-RegionA_ with cross-region replication enabled has as a target the bucket _cross-GG-backup-RegionA_ in _RegionB_.

In _RegionB:_

*   An ADW "Standby" instance (_OCIDBTARGET01\_Remote_) aligned with the primary instance _OCIDBTARGET01_ in _RegionA_.
*   An OCI GG secondary deployment (_ocigg-RegionB_) that can be restored with the backup files stored in the bucket _GG-backup-RegionB_.
*   The OS bucket _cross-GG-backup-RegionA_ that contains the OCI GG backup files (_ocigg-backup_) replicated from _RegionA_.
*   An OCI Function triggered by an Event Rule whenever a backup file is created or updated in the bucket _cross-GG-backup-RegionA_, that copies the backup files in the bucket _GG-backup-RegionB_.

In the second part of the blog, I simulate a disaster in _RegionA_ and describe the operational tasks that you need to perform to properly start the workload in _RegionB_.

## **Real-Time DW failure simulation**

### **Initial State**

Before a failure occurs, the workload is up and running properly in the primary OCI Region (RegionA, _eu-milan-1_). OCI GoldenGate processes are running and ready to update in real-time the primary ADW with changes that occur in the SourceDB. In the secondary OCI Region (RegionB, _eu-frankfurt-1_) the _Standby_ ADW instance is active and constantly aligned with the primary instance, while the OCI GoldenGate secondary deployment is inactive.

RegionA, Autonomous Data Warehouse Primary instance:
![_Fig.2: RegionA, Autonomous Data Warehouse Primary instance_](/data-organon/images/2023-04-20-OCI-GG-DR-Part-II//6483744915.png)

RegionB, Autonomous Data Warehouse Standby instance:
![_Fig.3: RegionB, Autonomous Data Warehouse Standby instance_](/data-organon/images/2023-04-20-OCI-GG-DR-Part-II//6483744918.png)

RegionA, OCI GoldenGate primary deployment:
![_Fig.4: RegionA, OCI GoldenGate primary deployment_](/data-organon/images/2023-04-20-OCI-GG-DR-Part-II//6483744921.png)

Fig.5: RegionB, OCI GoldenGate secondary deployment:
![_Fig.5: RegionB, OCI GoldenGate secondary deployment_](/data-organon/images/2023-04-20-OCI-GG-DR-Part-II//6483744927.png)

Backup of the OCI GoldenGate primary deployment in RegionA is periodically created in an Object Storage bucket in RegionA (_GG-backup-RegionA_), replicated in a bucket in RegionB (cross-_GG-backup-RegionA_) and finally copied in the Object Storage bucket where secondary OCI GoldenGate stores the manual backup files (_GG-backup-RegionB_).

RegionA, OCI GoldenGate manual backup (triggered by the _oci-cli_ command script):
![_Fig.6: RegionA, OCI GoldenGate manual backup (triggered by the oci-cli command script)_](/data-organon/images/2023-04-20-OCI-GG-DR-Part-II//6471379681.png)

RegionA, OCI GoldenGate manual backup file in the Object Storage bucket:
![_Fig.7: RegionA, OCI GoldenGate manual backup file in the Object Storage bucket_](/data-organon/images/2023-04-20-OCI-GG-DR-Part-II//6471379695.png)

RegionB, OCI GoldenGate manual backup file from RegionA replicated in the Object Storage bucket in RegionB:
![_Fig.8: RegionB, OCI GoldenGate manual backup file from RegionA replicated in the Object Storage bucket in RegionB_](/data-organon/images/2023-04-20-OCI-GG-DR-Part-II//6471379707.png)

RegionA, OCI GoldenGate backup file copied (by an OCI Function) in the Object Storage bucket reserved for OCI GG manual backup in RegionB:
![_Fig.9: RegionA, OCI GoldenGate backup file copied (by an OCI Function) in the Object Storage bucket reserved for OCI GG manual backup in RegionB](/data-organon/images/2023-04-20-OCI-GG-DR-Part-II//6471379719.png)

### **Simulation of failure in RegionA**

To simulate a failure in RegionA, I just turn off the primary OCI GoldenGate deployment and the primary ADW instance in RegionA.

RegionA, OCI GoldenGate deployment inactive:
![_Fig.10: RegionA, OCI GoldenGate deployment inactive_](/data-organon/images/2023-04-20-OCI-GG-DR-Part-II//6483745326.png)

RegionA, Autonomous Data Warehouse primary instance stopped:
![_Fig.11: RegionA, Autonomous Data Warehouse primary instance stopped_](/data-organon/images/2023-04-20-OCI-GG-DR-Part-II//6483745328.png)

The picture below shows how the architecture deployment would appear in real-life immediately after the failure in RegionA.
![_Fig.12: Deployment status after failure in RegionA_](/data-organon/images/2023-04-20-OCI-GG-DR-Part-II//6310899891.png)

After the failure simulation, I insert a record in a table in the SourceDB that should be replicated in the target ADW. This is to show how, after OCI GoldenGate and ADW will be recovered in RegionB, it will also be possible to recover the transactions that have taken place in the source database in the meantime.

As an example, I insert a row in the table _SRC\_REGION_ of _SourceDB_ configured in the OCI GoldenGate extract parameter file.

Connecting with SQL Developer Web to the _SourceDB_, I insert a record and commit the transaction:
![_Fig.13: Insert record in the SourceDB table (SRC\_REGION)_](/data-organon/images/2023-04-20-OCI-GG-DR-Part-II//6486268289.png)

The new record is then visible in the _SRC\_REGION_ table of SourceDB:
![_Fig.14: New record present in the SourceDB table (SRC\_REGION)_](/data-organon/images/2023-04-20-OCI-GG-DR-Part-II//6486268307.png)

### **Real-Time DW Recovery Operational Tasks**

You need to execute some operational tasks to properly start the workload in RegionB:

1.  Execute ADW Switchover.
2.  Start OCI GoldenGate deployment.
3.  Restore OCI GoldenGate deployment.
4.  Update the connection assigned to the OCI GoldenGate deployment.
5.  Start OCI GoldenGate _Extract_ and _Replicat_ process.

Steps to recover the workload in RegionB:
![_Fig.15: steps to recover the workload in RegionB_](/data-organon/images/2023-04-20-OCI-GG-DR-Part-II//6520923337.png)

**1)** **Execute ADW Switchover:** From the OCI console, you can execute the ADW switchover. In this way, the ADW instance in RegionB (_OCIDBTARGET01\_Remote_) changes its role to _Primary_:

Switchover of the ADW instance in RegionB:
![_Fig.16: Switchover of the ADW instance in RegionB_](/data-organon/images/2023-04-20-OCI-GG-DR-Part-II//6483749713.png)

ADW instance in RegionB is now primary:
![_Fig.17: ADW instance in RegionB is now primary_](/data-organon/images/2023-04-20-OCI-GG-DR-Part-II//6483749712.png)

By querying the _SRC\_REGION_ table in the replicated schema (_SRCMIRROR\_OCIGGLL_) you can see that the new record inserted in the source DB during the downtime, as expected, is not yet present in the target ADW:

RegionB, _OCI\_REGION_ table not yet aligned with SourceDB after switchover:
![_Fig.18: RegionB, OCI\_REGION table not yet aligned with source DB after switchover_](/data-organon/images/2023-04-20-OCI-GG-DR-Part-II//6524346004.png)

**2) Start OCI GoldenGate secondary deployment:** From the OCI Console, start the OCI GoldenGate deployment (_oci-GG-RegionB_).

OCI GoldenGate deployment in RegionB is now active:
![_Fig.19: OCI GoldenGate deployment in RegionB now active._](/data-organon/images/2023-04-20-OCI-GG-DR-Part-II//6511847199.png)

Once active, you can see that, since it's never been used so far, at this moment the deployment has no extract or replicat process.

RegionB, OCI GoldenGate extract and replicat processes before restoring:
![_Fig.20: RegionB, OCI GoldenGate extract and replicat processes before restoring._](/data-organon/images/2023-04-20-OCI-GG-DR-Part-II//6511847214.png)


And it has two assigned connections. Connection to the source database (_SourceDB_) and the connection to the target database (_TargetOCIDB_) in RegionB (_eu-frankfurt-1_) which is the target ADW database that has now become primary. 

RegionB, OCI GoldenGate associated connections before restoring:
![_Fig.21: RegionB, OCI GoldenGate associated connections before restoring._](/data-organon/images/2023-04-20-OCI-GG-DR-Part-II//6520908505.png)


**3) Restore OCI GoldenGate secondary deployment**

You must restore the OCI GoldenGate deployment in RegionB using the manual backup replicated from RegionA. You need to browse the deployment backups to find the manual backup that has the file _bkp-ocigg_ as Object name and the bucket _GG-backup-RegionB_ as Backup location. Then you can restore from that file.

OCI GoldenGate deployment backup to restore:
![_Fig.22: OCI GoldenGate deployment backup to restore._](/data-organon/images/2023-04-20-OCI-GG-DR-Part-II//6523687214.png)

RegionB, OCI GoldenGate deployment restoring:
![_Fig.23: RegionB, OCI GoldenGate deployment restoring_](/data-organon/images/2023-04-20-OCI-GG-DR-Part-II//6524343106.png)

**4) Update the connection assigned to the OCI GoldenGate secondary deployment**

After restoring, you need to update the connections in the OCI GoldenGate configuration because now:

*   The OCI GoldenGate backup from RegionA has the connection to Source DB (_SourceDB_) with the _COLOCATION_ and _WALLET\_DIRECTORY_ parameters pointing to RegionA (_eu-milan-1_)
*   The OCI GoldenGate backup from RegionA has the connection to Target DB (_TargetOCIDB_) pointing to ADW in RegionA (_eu-milan-1_), in addition to the wrong _COLOCATION_ and _WALLET\_DIRECTORY_ parameters_.

RegionB, OCI GoldenGate connections after restoring:
![_Fig.24: RegionB, OCI GoldenGate connections after restoring._](/data-organon/images/2023-04-20-OCI-GG-DR-Part-II//6511851608.png)

To update connections, you can reassign (_Unassign_ and then _Assign_ again) one of the two connections to the OCI GoldenGate deployment using the OCI Console. 

Update OCI GoldenGate connections, first step: unassign the connection:
![_Fig.25: Update OCI GoldenGate connections, first step: unassign the connection._](/data-organon/images/2023-04-20-OCI-GG-DR-Part-II//6520912838.png)

Update OCI GoldenGate connections, second step: reassign the same connection:
![_Fig.26: Update OCI GoldenGate connections, second step: reassign the same connection._](/data-organon/images/2023-04-20-OCI-GG-DR-Part-II//6520912839.png)

Reassigning a connection to the OCI GoldenGate deployment forces a refresh of the connections used by the OCI GoldenGate processes. As you can see from the GoldenGate console, the OCI GoldenGate deployment again has the proper connection for RegionB:

OCI GoldenGate updated connections:
![_Fig.27: OCI GoldenGate connections updated_ ](/data-organon/images/2023-04-20-OCI-GG-DR-Part-II//6520908505.png)

Note: Alternatively, to update the connections, you can also manually edit the connections using the OCI GoldenGate console (in the _Configuration_ section). To do so, you need to manually enter the connection string (keep a copy of connection strings and passwords for ggadmin users before restoring).

**5) Start OCI GoldenGate _Extract_ and _Replicat_ process**

Now you are ready to start the OCI GoldenGate _Extract_ and _Replicat_ processes. Before starting those processes, you need to check the current CSN (Commit Sequence Number) in the checkpoint table of the replicat process. In order not to lose any transactions that occurred in the meantime in the source DB, _Extract_ and _Replicat_ processes must start from CSN immediately following the one specified in the _CHECKTABLE_ of the target ADW.

You need to verify the CSN in the checkpoint table of the target ADW for your replicat process (_REP_ in this case):

Checking the CSN of the target ADW:
![_Fig.28: Checking the CSN of the target ADW_](/data-organon/images/2023-04-20-OCI-GG-DR-Part-II//6523691710.png)

Then you can start the _Extract_ and _Replicat_ processes with the proper CSN:

Start Extract process with the start point being after the CSN of the target ADW:
![_Fig.29: Start Extract process with the start point being after the CSN of the target ADW_](/data-organon/images/2023-04-20-OCI-GG-DR-Part-II//6523691755.png)

Start Replicat process with the start point being after the CSN of the target ADW:
![_Fig.30: Start Replicat process with the start point being after the CSN of the target ADW_](/data-organon/images/2023-04-20-OCI-GG-DR-Part-II//6523691773.png)

The OCI GoldenGate deployment is now active in RegionB with all the processes properly updating the target ADW with changes starting from the ones that occurred during the failure.

RegionB, OCI GoldenGate processes up and running:
![_Fig.31: RegionB, OCI GoldenGate processes up and running_](/data-organon/images/2023-04-20-OCI-GG-DR-Part-II//6523692016.png)

As an example, with a query against the _SRC\_REGION_ table in the schema _SRC\_OCIGGLL_ of the target ADW, we can see the record inserted in the source database during the unavailability of the workload processes.

RegionB, record inserted during unavailability in the SourceDB, recovered in the target ADW table:
![_Fig.32: RegionB, record inserted during unavailability in the SourceDB, recovered in the target ADW table._](/data-organon/images/2023-04-20-OCI-GG-DR-Part-II//6524351951.png)


## **Possible Enhancements**

### **Automation Available**

The five workload recovery steps listed above can be automated by leveraging the OCI and GoldenGate APIs.

Using _Rest API_, _SDKs_ or _oci-cli_ commands you can automatically:

**1)** **Perform** **ADW Switchover**

**2) Start OCI GoldenGate deployment**

**3) Restore OCI GoldenGate deployment**

**4) Update the connection assigned to the OCI GoldenGate deployment**

For more information, please refer to this Oracle documentation:

*   [OCI GoldenGate API Reference and Endpoints](https://docs.oracle.com/en-us/iaas/api/#/en/goldengate/20200407/)
*   [SDKs and Command Line Interface](https://docs.oracle.com/en-us/iaas/Content/API/Concepts/sdks.htm)

Regarding Steps #5 (**Start OCI GoldenGate _Extract_ and _Replicat_ process**), you need to perform a couple of operations:

 *   Check the CSN of the Replicat process in the checkpoint table of the target DB. To do this, you can query directly the DB table or call an Oracle REST API created on that purpose with Oracle Rest Data Service.
 *   Start the Extract or Replicat processes by leveraging GoldenGate REST API. The GoldenGate REST API endndpont can be found by running the following command:

 _curl -i -X GET -u username:password -H request-header:value https://<OCI GG host>/path/resource-path_

 For more information, please refer to this Oracle documentation:

 *   [Oracle Rest Data Services on ADW](https://docs.oracle.com/en/cloud/paas/autonomous-database/adbsa/ords-rest.html)
 *   [GoldenGate REST API](https://docs.oracle.com/en-us/iaas/Content/API/Concepts/sdks.htm)

## _Credits_

Valuable ideas, suggestions, and reviews for this post have been kindly provided by:

   - **Claudia Filippini**, Oracle Account Cloud Engineer
   - **Eloi Lopez**, Oracle Domain Specialist Cloud Engineer - Data Management
   - **Alessandro Stella**, Oracle Account Cloud Engineer
