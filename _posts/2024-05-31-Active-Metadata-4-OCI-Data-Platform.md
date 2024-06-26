---
title: Managing Active Metadata with Oracle Data Platform
description: Managing active metadata with OCI Data Catalog for an Oracle Data Platform architecture.
---

<!-- Google tag (gtag.js) -->

<script async src="https://www.googletagmanager.com/gtag/js?id=G-WP81WC62NJ"></script>

<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-WP81WC62NJ');
</script>

![](/data-organon/images/2024-04-24-Active-Metadata-4-OCI-Lakehouse/ACTIVE-METADATA.jpeg)

Sometimes, being active at the right time is crucial to achieving goals successfully.

## **Introduction**

Metadata management is becoming more and more important in Data Platform solutions. Regardless of whether the data architecture is distributed or centralized, users must be able to discover the data, comprehend its technical components, grasp its business significance, learn the quality level, understand the sources of the data, and comprehend how it has been transformed and integrated. Data consumers can use the data to its fullest potential thanks to all of that information, and they also get data trust, which is essential for the solution to be broadly adopted.

The specific categories of metadata known as ***active metadata*** tell us something about the current state of the data. They make reference to things like their most recent update status and time, their availability or unavailability, and their proximity or distance from the SLOs that were previously established for the different validation controls. As a result, they are dynamic metadata by definition. Indeed, active.

In this article, I focus on enabling and managing ***active metadata*** for a Data Platform solution in OCI. I will show how you can automatically update load status property in **OCI Data Catalog** based on the result of **OCI Data Integration** pipeline. Naturally, the same concept could be extended to other types of properties.

## Starting Solution

Let's consider this Data Platform solution in OCI:

![Fig.1: OCI Lakehouse initial solution architecture](/data-organon/images/2024-04-24-Active-Metadata-4-OCI-Lakehouse/OCI-Lakehouse-initial-solution.png)

It's an example of OCI Lakehouse architecture that  initially is made by:

- **Autonomous Data Warehouse**, with a sample data model composed of a schema *ADWUSER01* with four tables for curated data: *REVENUE_TARGET*, *CUSTOMERS_TARGET*,*EMPLOYEES_WEST_MIDWEST*, *EMPLOYEES_WEST_MIDWEST*
- **OCI Object Storage**, with the bucket *DI-Bucket* that stores files used to load curated data into the ADW data model:
  
  ![Fig.3: OCI Lakehouse Object Storage Bucket](/data-organon/images/2024-04-24-Active-Metadata-4-OCI-Lakehouse/DI-bucket.png)
- **OCI Data Integration**, with a workspace which includes tasks, data loads and pipelines that load data into the ADW data model. As an example, the picture below shows the main pipeline with the steps to load the ADW target tables:
  
  ![Fig.4: OCI Data Integration Pipeline](/data-organon/images/2024-04-24-Active-Metadata-4-OCI-Lakehouse/OCIDI-pipeline.png)
- **OCI Data Catalog**, to manage both technical and business metadata of the Lakehouse data assets, and the entity data lineage based on OCI Data Integration loading tasks:
  
  - Example of entity **technical metadata**:
    
    ![Fig.5: OCI Data Catalog Technical Metadata](/data-organon/images/2024-04-24-Active-Metadata-4-OCI-Lakehouse/dcat-customer-technical-metadata.png)
  - **Business Glossary**:
    
    ![Fig.6: OCI Data Catalog Business Glossary](/data-organon/images/2024-04-24-Active-Metadata-4-OCI-Lakehouse/dcat-business-glossary.png)
  - Example of **Data Lineage** metadata:
    
    ![Fig.7: OCI Data Catalog Lineage](/data-organon/images/2024-04-24-Active-Metadata-4-OCI-Lakehouse/dcat-lineage.png)

Now we need to add **active metadata** to promptly inform data consumers about the status of the data assets.

## Adding Active Metadata

When you harvest a data source using Data Catalog, some default properties are created in your data catalog for data entities and attributes. To provide more context to the technical or business metadata, you can leverage OCI Data Catalog Custom Properties. You can create a Custom Property and then link it to the appropriate object type for the metadata enrichment.
As an example, you might want to create Custom Properties to manage active metadata that can provide information about:

- **Availability/unavailability** of entities, folders, schemas, or data assets.
- **Results of the last refresh** of entities, folders, schemas, or data assets.
- **Date-time of the last refresh** of entities, folders, schemas, or data assets.
- **Results of data validation checks** of entities, folders, schemas, or data assets.

In this article, I create a custom property that gets updated automatically with the outcomes of the most recent data update process. This requires the following steps:

- Adding a **Custom Property** and associate it to the proper Data Catalog objects.
- Creating **OCI Function** that updates the value of the Custom Property.
- Using **OCI Events** to automatically invoke the OCI Function.
- As alternative, adding **Tasks** in the **OCI Data Integration pipeline** to invoke the OCI Function.

### Adding Custom Properties to OCI Data Catalog objects

I create a Custom Property *Data Refresh* that indicates whether the last data update's run as expected.
The property's got three possible values:

- ***Not Known***
- ***Last Update Available***
- ***Last Update Not Available***

During Custom Property creation, you can also specify which type of Data Catalog can be associated to it. In my case I choose:

- All Data Assets
- All Folders
- All Entities

The picture below shows the Custom Properties creation:

![Fig.8: Custom Properties creation](/data-organon/images/2024-04-24-Active-Metadata-4-OCI-Lakehouse/custom-properties-creation.png)

Since it's enabled for all the data catalog types, I can now start using it. For example, I can set its value as *Not Known* for the *Customer* data entity:

![Fig.9: Edit Property Value](/data-organon/images/2024-04-24-Active-Metadata-4-OCI-Lakehouse/setting-property-value.png)

![Fig.10: Custom Property Value](/data-organon/images/2024-04-24-Active-Metadata-4-OCI-Lakehouse/customer-property-notknown.png)

But we don't want to manage this property manually, then let's make it ***Active***!

### Creating OCI Function to update OCI Data Catalog custom properties

You can use the specific Data Catalog object update API to set the value of its Custom Properties.
As an example, I developed OCI Functions to update the Custom Property of a Data Catalog Entity and the Custom Property of a Data Catalog Folder (which, in the case of an Oracle DB asset, corresponds to a DB schema).

#### OCI Function to update Entity Custom Property

In this first example, the function uses the OCI Python SDK to set the value of a Custom Property of Data Catalog Entity.

***oci-datacatalog-update-customproperties-python***:

```
import io
import oci
import json
import logging
from datetime import datetime
from fdk import response

def update_entity(signer, catalog_id, data_asset_key, entity_business_name, entity_key, custom_property_key, custom_property_display_name, custom_property_value):
try:
data_catalog_client = oci.data_catalog.DataCatalogClient(config={},signer=signer)
resp = data_catalog_client.update_entity(
catalog_id = catalog_id,
data_asset_key = data_asset_key,
entity_key = entity_key,
update_entity_details=oci.data_catalog.models.UpdateEntityDetails(
business_name = entity_business_name,
custom_property_members=[
oci.data_catalog.models.CustomPropertySetUsage(
key = custom_property_key,
display_name = custom_property_display_name,
value = custom_property_value,
namespace_name = "Custom Properties")])
)
except (Exception, ValueError) as ex:
logging.getLogger().error(str(ex))
return {"response": str(ex)}

return {"response": str(response)}

def handler(ctx, data: io.BytesIO=None):
signer = oci.auth.signers.get_resource_principals_signer()
resp   = ""
try:
body = json.loads(data.getvalue())
data_asset_key = body["data_asset_key"]
entity_key = body["entity_key"]
entity_business_name = body["entity_business_name"]
custom_property_key = body["custom_property_key"]
custom_property_display_name = body["custom_property_display_name"]
custom_property_value = body["custom_property_value"]
catalog_id = "ocid1.datacatalog.oc1.eu-milan-1.xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxq"

logging.getLogger().info(f'Updating Entity "{entity_business_name}": setting Custom Property "{custom_property_display_name}" value to "{custom_property_value}"')

resp = update_entity(signer, catalog_id, data_asset_key, entity_business_name, entity_key, custom_property_key, custom_property_display_name, custom_property_value)

except (Exception, ValueError) as ex:
logging.getLogger().error(str(ex))
```

Since I need to update a Custom Property of a Data Catalog entity, a function requires at least the following input parameters:

- data_asset_key
- entity_key
- custom_property_key
- custom_property_value
- catalog_id

> Note: the *data_asset_key* and *catalog_id* are information that you can get from Data Catalog UI, while to discover *entity_key* and *custom_properties_key* you need to use the Data Catalog API (I used the *oci-cli* commands directly in the OCI Cloud shell).

I also added:

* entity_business_name
* custom_property_display_name

Just to have more immediate information in the OCI Function execution log.

#### OCI Function to update Folder Custom Property

In this second example, the function uses the OCI Python SDK to set the value of a Custom Property of Data Catalog Folder.

***oci-datacatalog-update-customproperties-folder-ok-python:***

```
import io
import oci
import json
import logging
from datetime import datetime
from fdk import response

def update_folder(signer, catalog_id, data_asset_key, folder_display_name, folder_key, custom_property_key, custom_property_display_name, custom_property_value):
    try:
        data_catalog_client = oci.data_catalog.DataCatalogClient(config={},signer=signer)
        resp = data_catalog_client.update_folder(
               catalog_id = catalog_id,
               data_asset_key = data_asset_key,
               folder_key = folder_key,
               update_folder_details=oci.data_catalog.models.UpdateFolderDetails(
               display_name = folder_display_name,
               custom_property_members=[
               oci.data_catalog.models.CustomPropertySetUsage(
               key = custom_property_key,
               display_name = custom_property_display_name,
               value = custom_property_value,
               namespace_name = "Custom Properties")])
        )
    except (Exception, ValueError) as ex:
        logging.getLogger().error(str(ex))
        return {"response": str(ex)}

    return {"response": str(response)}


def handler(ctx, data: io.BytesIO=None):
    signer = oci.auth.signers.get_resource_principals_signer()
    resp   = ""
    try:
        #body = json.loads(data.getvalue())
        data_asset_key = "1e85bda4-xxxx-xxxx-xxxx-e34229eed014"
        folder_display_name = "ADWUSER01"
        folder_key = "84a8a0e3-xxxx-xxxx-xxxx-73c3f97ae1da"
        custom_property_key = "a66ad67c-xxxx-xxxx-xxxx-257e6acf744a"
        custom_property_display_name = "Data Refresh"
        custom_property_value = "Last Update Available"
        catalog_id = "ocid1.datacatalog.oc1.eu-milan-1.xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxq"

        logging.getLogger().info(f'Updating Folder "{folder_display_name}": setting Custom Property "{custom_property_display_name}" value to "{custom_property_value}"')

        resp = update_folder(signer, catalog_id, data_asset_key, folder_display_name, folder_key, custom_property_key, custom_property_display_name, custom_property_value)
    except (Exception, ValueError) as ex:
        logging.getLogger().error(str(ex))
```

Now I can use those functions to update the active metadata *Data Refresh* according to the results of the data update process.

### Option 1: Updating Active Metadata with OCI Events

Rules that enable the automation of the active metadata update can be built using events produced by OCI Data Integration.
In this example, I want to update the Custom Property *Data Refresh* of the ADW schema *ADWUSER01* according to the status of the OCI Data Integration pipeline task in charge of updating all the schema's tables.
Therefore, I create two event rules, one that executes an OCI Function in case the pipeline task runs successfully, and the other that executes another OCI Function in case the same task ends with errors.

> Note: When an OCI Event invokes an OCI Function, you cannot specify input parameters. Then, in these examples, I used two OCI Function: the one described in the above paragraph, and another one (named *oci-datacatalog-update-customproperties-folder-ko-python*) that is very similar. The only difference is that it updates the Folder Custom Property according to the event of an error.

Rule ***oci-data-integration-pipeline-ADW-success***:

* **Event Type**:
  * Service: *Data Integration*
  * Event Type: *Execute Task - End*
* **Attributes**:
  * taskKey: *177322d6-xxxx-xxxx-xxxx-1fd15b182d25* (Key of the *Load DWH Pipeline Task*, visible in the OCI Data Integration UI)
  * TaskStatus: *SUCCESS* (uppercase)
* **Action**:
  * Function *oci-datacatalog-update-customproperties-folder-ok-python*

![Fig.11: Event Task Success](/data-organon/images/2024-04-24-Active-Metadata-4-OCI-Lakehouse/event-rule-success.png)

Rule ***oci-data-integration-pipeline-ADW-error***:

* **Event Type**:
  * Service: *Data Integration*
  * Event Type: *Execute Task - End*
* **Attributes**:
  * taskKey: *177322d6-xxxx-xxxx-xxxx-1fd15b182d25* (key of the *Load DWH Pipeline Task*)
  * TaskStatus: *ERROR* (uppercase)
* **Action**:
  * Function *oci-datacatalog-update-customproperties-folder-ko-python*

![Fig.12: Event Rule Task Error](/data-organon/images/2024-04-24-Active-Metadata-4-OCI-Lakehouse/event-rule-error.png)

Good, now running the OCI Data Integration pipeline task *Load DWH Pipeline Task* will update my ADW Schema tables and its active metadata accordingly.

I initially set the value of *Data Refresh* for the ADW Schema *ADWUSER01* as *"Not Known"*:

![Fig.13: Data Refresh Initial Status](/data-organon/images/2024-04-24-Active-Metadata-4-OCI-Lakehouse/adw-schema-data-refresh-initial-status.png)

I run the pipeline task that completes successfully:

![Fig.14: Pipeline Task Runs Successfully](/data-organon/images/2024-04-24-Active-Metadata-4-OCI-Lakehouse/pipeline-load-dwh-success.png)

The OCI Events log shows that there's been an event rule match:

![Fig.15: Matching Event for Successful run](/data-organon/images/2024-04-24-Active-Metadata-4-OCI-Lakehouse/event-success-execution-log.png)

As a consequence, the Custom Properties is updated accordingly:

![Fig.16: Custom Properties Updated on Success](/data-organon/images/2024-04-24-Active-Metadata-4-OCI-Lakehouse/schema-data-refresh-available.png)

As a test, I rerun the pipeline making it fail intentionally (the ADW instance is not active):

![Fig.17: Pipeline Task Run with errors](/data-organon/images/2024-04-24-Active-Metadata-4-OCI-Lakehouse/pipeline-load-dwh-error.png)

The OCI Event created to intercept the error state, shows that there's been an event rule match:

![Fig.15: Matching Event for Error run](/data-organon/images/2024-04-24-Active-Metadata-4-OCI-Lakehouse/event-error-execution-log.png)

And the active metadata now shows the value expected in the event of an error:

![Fig.17: Custom Properties Updated on Error](/data-organon/images/2024-04-24-Active-Metadata-4-OCI-Lakehouse/schema-data-refresh-not-available.png)

### Option 2: Updating Active Metadata with OCI Data Integration task

By including OCI Data Integration Rest tasks that make use of OCI Functions endpoints, updating the Custom Properties can also be part of the data pipeline process in OCI Data Integration.

For this article, I create two OCI Data Integration Rest tasks that invoke the OCI Function that updates an entity's Custom Properties.
As example, below you can see the Rest task for the successful updates of the *Customer* entity:

![Fig.18: Rest task OCI Data Integration](/data-organon/images/2024-04-24-Active-Metadata-4-OCI-Lakehouse/oci-di-rest-api-task.png)

The details of the RestAPI shows the OCI Function endpoint configured as URL for the POST method and, in the request section, the JSON structure with the OCI Function input parameters values:

![Fig.19: OCI DI Rest task details](/data-organon/images/2024-04-24-Active-Metadata-4-OCI-Lakehouse/rest-task-details.png)

Finally, I add two tasks to the pipeline to update the *Customer* active metadata according to the results of the loading process:

![Fig.20: Load DWH Pipeline with Active Metadata update](/data-organon/images/2024-04-24-Active-Metadata-4-OCI-Lakehouse/pipeline-active-metadata-task.png)

## Conclusion

We have seen how to enable dynamic updating of the Oracle Data Platform's active metadata. The choice between the two suggested methods, use of OCI Events or OCI DI Tasks, might depend on the weight you want to give to a couple of aspects that characterize the two solutions:

**OCI Event**:

- No impact for existing or new OCI DI Tasks.
- You cannot use the OCI Function request payload (that means no usage of parameters, thus a higher number of OCI Function deployments, potentially).

**OCI DI Task**:

- Impacts on OCI DI Tasks (new Rest tasks and modification of existing Pipeline ones).
- You can use request payload when invoking OCI Function.

Or it might depend on the different fundamental approach that implicitly the two solutions suggest: must the metadata update be part of the data update? Do you want the entire process to result in errors if the metadata update fails for whatever reason, or may it be acceptable that the outcome of the data process is positive regardless of whether their active metadata has been successfully updated?

Open discussion.

## References

- **OCI Data Platform**: [Oracle Architecture Center](https://docs.oracle.com/en/solutions/data-platform-lakehouse/index.html#GUID-A328ACEF-30B8-4595-B86F-F27B512744DF)
- **OCI Data Catalog**: [Oracle Documentation](https://docs.oracle.com/en-us/iaas/data-catalog/home.htm)
- **OCI Python SDK Reference**: [Oracle Documentation](https://docs.oracle.com/en-us/iaas/tools/python/2.126.4/api/landing.html)
- **OCI Function**: [Oracle Documentation](https://docs.oracle.com/en-us/iaas/Content/Functions/home.htm)
- **oracle-functions-samples**: [Oracle Github Repository](https://github.com/oracle-samples/oracle-functions-samples)

