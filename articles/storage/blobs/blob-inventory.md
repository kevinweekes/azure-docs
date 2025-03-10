---
title: Azure Storage blob inventory
description: Azure Storage inventory is a tool to help get an overview of all your blob data within a storage account.
services: storage
author: normesta

ms.service: storage
ms.date: 10/11/2021
ms.topic: conceptual
ms.author: normesta
ms.reviewer: klaasl
ms.subservice: blobs
ms.custom: references_regions
---

# Azure Storage blob inventory

The Azure Storage blob inventory feature provides an overview of your containers, blobs, snapshots, and blob versions within a storage account. Use the inventory report to understand various attributes of blobs and containers such as your total data size, age, encryption status, immutability policy, and legal hold and so on. The report provides an overview of your data for business and compliance requirements.

## Inventory features

The following list describes features and capabilities that are available in the current release of Azure Storage blob inventory.

- **Inventory reports for blobs and containers**

  You can generate inventory reports for blobs and containers. A report for blobs can contain base blobs, snapshots, blob versions and their associated properties such as creation time, last modified time. A report for containers describes containers and their associated properties such as immutability policy status, legal hold status.

- **Custom Schema**

  You can choose which fields appear in reports. Choose from a list of supported fields. That list appears later in this article.

- **CSV and Apache Parquet output format**

  You can generate an inventory report in either CSV or Apache Parquet output format.

- **Manifest file and Azure Event Grid event per inventory report**

  A manifest file and an Azure Event Grid event are generated per inventory report. These are described later in this article.

## Enabling inventory reports

Enable blob inventory reports by adding a policy with one or more rules to your storage account. For guidance, see [Enable Azure Storage blob inventory reports](blob-inventory-how-to.md).

## Upgrading an inventory policy

If you are an existing Azure Storage blob inventory user who has configured inventory prior to June 2021, you can start using the new features by loading the policy, and then saving the policy back after making changes. When you reload the policy, the new fields in the policy will be populated with default values. You can change these values if you want. Also, the following two features will be available.

- A destination container is now supported for every rule instead of just being supported for the policy.

- A manifest file and Azure Event Grid event are now generated per rule instead of per policy.

## Inventory policy

An inventory report is configured by adding an inventory policy with one or more rules. An inventory policy is a collection of rules in a JSON document.

```json
{
  "enabled": true,
  "rules": [
  {
    "enabled": true,
    "name": "inventoryrule1",
    "destination": "inventory-destination-container",
    "definition": {. . .}
  },
  {
    "enabled": true,
    "name": "inventoryrule2",
    "destination": "inventory-destination-container",
    "definition": {. . .}
  }]
}
```

View the JSON for an inventory policy by selecting the **Code view** tab in the **Blob inventory** section of the Azure portal.

| Parameter name | Parameter type | Notes | Required? |
|--|--|--|--|
| enabled | boolean | Used to disable the entire policy. When set to **true**, the rule level enabled field overrides this parameter. When disabled, inventory for all rules will be disabled. | Yes |
| rules | Array of rule objects | At least one rule is required in a policy. Up to 100 rules are supported per policy. | Yes |

## Inventory rules

A rule captures the filtering conditions and output parameters for generating an inventory report. Each rule creates an inventory report. Rules can have overlapping prefixes. A blob can appear in more than one inventory depending on rule definitions.

Each rule within the policy has several parameters:

| Parameter name | Parameter type | Notes | Required? |
|--|--|--|--|
| name | string | A rule name can include up to 256 case-sensitive alphanumeric characters. The name must be unique within a policy. | Yes |
| enabled | boolean | A flag allowing a rule to be enabled or disabled. The default value is **true**. | Yes |
| definition | JSON inventory rule definition | Each definition is made up of a rule filter set. | Yes |
| destination | string | The destination container where all inventory files will be generated. The destination container must already exist. |

The global **Blob inventory enabled** flag takes precedence over the *enabled* parameter in a rule.

### Rule definition

| Parameter name | Parameter type | Notes | Required |
|--|--|--|--|
| filters | json | Filters decide whether a blob or container is part of inventory or not. | Yes |
| format | string | Determines the output of the inventory file. Valid values are `csv` (For CSV format) and `parquet` (For Apache Parquet format).| Yes |
| objectType | string | Denotes whether this is an inventory rule for blobs or containers. Valid values are `blob` and `container`. |Yes |
| schedule | string | Schedule on which to run this rule. Valid values are `daily` and `weekly`. | Yes |
| schemaFields | Json array | List of Schema fields to be part of inventory. | Yes |

### Rule filters

Several filters are available for customizing a blob inventory report:

| Filter name | Filter type | Notes | Required? |
|--|--|--|--|
| blobTypes | Array of predefined enum values | Valid values are `blockBlob` and `appendBlob` for hierarchical namespace enabled accounts, and `blockBlob`, `appendBlob`, and `pageBlob` for other accounts. This field is not applicable for inventory on a container, (objectType: `container`). | Yes |
| prefixMatch | Array of up to 10 strings for prefixes to be matched. | If you don't define *prefixMatch* or provide an empty prefix, the rule applies to all blobs within the storage account. A prefix must be a container name prefix or a container name. For example, `container`, `container1/foo`. | No |
| includeSnapshots | boolean | Specifies whether the inventory should include snapshots. Default is `false`. This field is not applicable for inventory on a container, (objectType: `container`). | No |
| includeBlobVersions | boolean | Specifies whether the inventory should include blob versions. Default is `false`. This field is not applicable for inventory on a container, (objectType: `container`). | No |

View the JSON for inventory rules by selecting the **Code view** tab in the **Blob inventory** section of the Azure portal. Filters are specified within a rule definition.

```json
{
  "destination": "inventory-destination-container",
  "enabled": true,
  "rules": [
  {
    "definition": {
      "filters": {
        "blobTypes": ["blockBlob", "appendBlob", "pageBlob"],
        "prefixMatch": ["inventorytestcontainer1", "inventorytestcontainer2/abcd", "etc"],
        "includeSnapshots": false,
        "includeBlobVersions": true,
      },
      "format": "csv",
      "objectType": "blob",
      "schedule": "daily",
      "schemaFields": ["Name", "Creation-Time"]
    },
    "enabled": true,
    "name": "blobinventorytest",
    "destination": "inventorydestinationContainer"
  },
  {
    "definition": {
      "filters": {
        "prefixMatch": ["inventorytestcontainer1", "inventorytestcontainer2/abcd", "etc"]
      },
      "format": "csv",
      "objectType": "container",
      "schedule": "weekly",
      "schemaFields": ["Name", "HasImmutabilityPolicy", "HasLegalHold"]
    },
    "enabled": true,
    "name": "containerinventorytest",
    "destination": "inventorydestinationContainer"
    }
  ]
}
```

### Custom schema fields supported for blob inventory

- Name (Required)
- Creation-Time
- Last-Modified
- Content-Length
- Content-MD5
- BlobType
- AccessTier
- AccessTierChangeTime
- Expiry-Time
- hdi_isfolder
- Owner
- Group
- Permissions
- Acl
- Snapshot (Available and required when you choose to include snapshots in your report)
- VersionId (Available and required when you choose to include blob versions in your report)
- IsCurrentVersion (Available and required when you choose to include blob versions in your report)
- Metadata
- LastAccessTime

### Custom schema fields supported for container inventory

- Name (Required)
- Last-Modified
- LeaseStatus
- LeaseState
- LeaseDuration
- PublicAccess
- HasImmutabilityPolicy
- HasLegalHold
- Metadata

## Inventory run

A blob inventory run is automatically scheduled every day. It can take up to 24 hours for an inventory run to complete. For hierarchical namespace enabled accounts, a run can take as long as two days, and depending on the number of files being processed, the run might not complete by end of that two days. If a run does not complete successfully, check subsequent runs to see if they complete before contacting support. The performance of a run can vary, so if a run doesn't complete, it's possible that subsequent runs will.

Inventory policies are read or written in full. Partial updates aren't supported.

> [!IMPORTANT]
> If you enable firewall rules for your storage account, inventory requests might be blocked. You can unblock these requests by providing exceptions for trusted Microsoft services. For more information, see the Exceptions section in [Configure firewalls and virtual networks](../common/storage-network-security.md#exceptions).

## Inventory completed event

The `BlobInventoryPolicyCompleted` event is generated when the inventory run completes for a rule. This event also occurs if the inventory run fails with a user error before it starts to run. For example, an invalid policy, or an error that occurs when a destination container is not present will trigger the event. The following json shows an example `BlobInventoryPolicyCompleted` event.

```json
{
  "topic": "/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/BlobInventory/providers/Microsoft.EventGrid/topics/BlobInventoryTopic",
  "subject": "BlobDataManagement/BlobInventory",
  "eventType": "Microsoft.Storage.BlobInventoryPolicyCompleted",
  "id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "data": {
    "scheduleDateTime": "2021-05-28T03:50:27Z",
    "accountName": "testaccount",
    "ruleName": "Rule_1",
    "policyRunStatus": "Succeeded",
    "policyRunStatusMessage": "Inventory run succeeded, refer manifest file for inventory details.",
    "policyRunId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    "manifestBlobUrl": "https://testaccount.blob.core.windows.net/inventory-destination-container/2021/05/26/13-25-36/Rule_1/Rule_1.csv"
  },
  "dataVersion": "1.0",
  "metadataVersion": "1",
  "eventTime": "2021-05-28T15:03:18Z"
}
```

The following table describes the schema of the `BlobInventoryPolicyCompleted` event.

|Field|Type|Description|
|---|---|
|scheduleDateTime|string|The time that the inventory policy was scheduled.|
|accountName|string|The storage account name.|
|ruleName|string|The rule name.|
|policyRunStatus|string|The status of inventory run. Possible values are `Succeeded`, `PartiallySucceeded`, and `Failed`.|
|policyRunStatusMessage|string|The status message for the inventory run.|
|policyRunId|string|The policy run ID for the inventory run.|
|manifestBlobUrl|string|The blob URL for manifest file for inventory run.|

## Inventory output

Each inventory rule generates a set of files in the specified inventory destination container for that rule. The inventory output is generated under the following path:
`https://<accountName>.blob.core.windows.net/<inventory-destination-container>/YYYY/MM/DD/HH-MM-SS/<ruleName` where:

- *accountName* is your Azure Blob Storage account name.
- *inventory-destination-container* is the destination container you specified in the inventory rule.
- *YYYY/MM/DD/HH-MM-SS* is the time when the inventory began to run.
- *ruleName* is the inventory rule name.

### Inventory files

Each inventory run for a rule generates the following files:

- **Inventory file:** An inventory run for a rule generates one or more CSV or Apache Parquet formatted files. If the matched object count is large, then multiple files are generated instead of a single file. Each such file contains matched objects and their metadata. 

  > [!NOTE]
  > Reports in the Apache Parquet format present dates in the following format: `timestamp_millis [number of milliseconds since 1970-01-01 00:00:00 UTC`.

  For a CSV formatted file, the first row is always the schema row. The following image shows an inventory CSV file opened in Microsoft Excel.

  :::image type="content" source="./media/blob-inventory/csv-file-excel.png" alt-text="Screenshot of an inventory CSV file opened in Microsoft Excel":::

  > [!IMPORTANT]
  > The blob paths that appear in an inventory file might not appear in any particular order. 

- **Checksum file:** A checksum file contains the MD5 checksum of the contents of manifest.json file. The name of the checksum file is `<ruleName>-manifest.checksum`. Generation of the checksum file marks the completion of an inventory rule run.

- **Manifest file:** A manifest.json file contains the details of the inventory file(s) generated for that rule. The name of the file is `<ruleName>-manifest.json`. This file also captures the rule definition provided by the user and the path to the inventory for that rule. The following json shows the contents of a sample manifest.json file.

  ```json
  {
  "destinationContainer" : "inventory-destination-container",
  "endpoint" : "https://testaccount.blob.core.windows.net",
  "files" : [
    {
      "blob" : "2021/05/26/13-25-36/Rule_1/Rule_1.csv",
      "size" : 12710092
    }
  ],
  "inventoryCompletionTime" : "2021-05-26T13:35:56Z",
  "inventoryStartTime" : "2021-05-26T13:25:36Z",
  "ruleDefinition" : {
    "filters" : {
      "blobTypes" : [ "blockBlob" ],
      "includeBlobVersions" : false,
      "includeSnapshots" : false,
      "prefixMatch" : [ "penner-test-container-100003" ]
    },
    "format" : "csv",
    "objectType" : "blob",
    "schedule" : "daily",
    "schemaFields" : [
      "Name",
      "Creation-Time",
      "BlobType",
      "Content-Length",
      "LastAccessTime",
      "Last-Modified",
      "Metadata",
      "AccessTier"
    ]
  },
  "ruleName" : "Rule_1",
  "status" : "Succeeded",
  "summary" : {
    "objectCount" : 110000,
    "totalObjectSize" : 23789775
  },
  "version" : "1.0"
  }
  ```

## Pricing and billing

Pricing for inventory is based on the number of blobs and containers that are scanned during the billing period. As an example, suppose an account contains one million blobs, and blob inventory is set to run once per week. After four weeks, four million blob entries will have been scanned.

Billing for blob inventory begins on October 1, 2021. Regional pricing will be published at that time. The baseline price without regional adjustment is approximately $0.0025 USD per million entries scanned for blob storage and $0.0035 USD if Data Lake Storage Gen2 is enabled. After inventory files are created, additional standard data storage and operations charges will be incurred for storing, reading, and writing the inventory-generated files in the account.

After an inventory report is complete, additional standard data storage and operations charges are incurred for storing, reading, and writing the inventory report in the storage account.

If a rule contains a prefix that overlaps with a prefix of any other rule, then the same blob can appear in more than one inventory report. In this case, you are billed for both instances. For example, assume that the `prefixMatch` element of one rule is set to `["inventory-blob-1", "inventory-blob-2"]`, and the `prefixMatch` element of another rule is set to `["inventory-blob-10", "inventory-blob-20"]`. An object named `inventory-blob-200` appears in both inventory reports.

Snapshots and versions of a blob also count towards billing even if you've set `includeSnapshots` and `includeVersions` filters to `false`. Those filter values don't affect billing. You can use them only to filter what appears in the report.

For more information about pricing for Azure Storage blob inventory, see [Azure Blob Storage pricing](https://azure.microsoft.com/pricing/details/storage/blobs/).

## Feature support

This table shows how this feature is supported in your account and the impact on support when you enable certain capabilities.

| Storage account type | Blob Storage (default support) | Data Lake Storage Gen2 <sup>1</sup> | NFS 3.0 <sup>1</sup> | SFTP <sup>1</sup> |
|--|--|--|--|--|
| Standard general-purpose v2 | ![Yes](../media/icons/yes-icon.png) | ![Yes](../media/icons/yes-icon.png)  <sup>2</sup>              | ![Yes](../media/icons/yes-icon.png) <sup>2</sup> | ![Yes](../media/icons/yes-icon.png)  <sup>2</sup> |
| Premium block blobs | ![Yes](../media/icons/yes-icon.png)| ![Yes](../media/icons/yes-icon.png)  <sup>2</sup> | ![Yes](../media/icons/yes-icon.png)  <sup>2</sup> | ![Yes](../media/icons/yes-icon.png)  <sup>2</sup> |

<sup>1</sup> Data Lake Storage Gen2, Network File System (NFS) 3.0 protocol, and Secure File Transfer protocol (SFTP) support all require a storage account with a hierarchical namespace enabled.

<sup>2</sup> Feature is supported at the preview level.

## Known issues

This section describes limitations and known issues of the Azure Storage blob inventory feature.

### Inventory job fails to complete for hierarchical namespace enabled accounts

The inventory job might not complete within 2 days for an account with hundreds of millions of blobs and hierarchical namespace enabled. If this happens, no inventory file is created. If a job does not complete successfully, check subsequent jobs to see if they complete before contacting support. The performance of a job can vary, so if a job doesn't complete, it's possible that subsequent jobs will.

### Inventory job cannot write inventory reports

An object replication policy can prevent an inventory job from writing inventory reports to the destination container. Some other scenarios can archive the reports or make them immutable when they are partially completed. This can lead to inventory job failure.

## Next steps

- [Enable Azure Storage blob inventory reports](blob-inventory-how-to.md)
- [Calculate the count and total size of blobs per container](calculate-blob-count-size.md)
- [Manage the Azure Blob Storage lifecycle](./lifecycle-management-overview.md)
