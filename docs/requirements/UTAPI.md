# [S3C-2862](https://scality.atlassian.net/browse/S3C-2862): UTAPI redesign requirements document

## Context & Problem to solve

The current design of the UTAPI is not able to meet the needs of our customers, especially the large company like service providers. Their main goals are:

* integrate Scality Products with a reporting and/or billing systems
* be able to create create a bill like AWS, the leader in public cloud
* be able to monitor in near real-time the consumption of the services offered by Scality products

Today, they are using UTAPI to **bill and monitor** the consumption of the services provided by S3C connectors to their customers. The main issues reported are:

* the impossibility to compute a monthly bill with Hour or even Day granularity
* the error rate of the amount of data consumed reported by the UTAPI(up to 200%)
* the incidents caused by UTAPI resources consumption(e.g: Redis process reach 100% CPU utilization that leads an entire S3C server out of services)

## Requirements

Requirements are written using user story format to keep the customer context in mind during the redesign.

### High Level

As a Storage Manager, I want to have a near real-time and accurate view of Storage Account service consumption metrics(described bellow) grouped by service consumption metrics Groups(described below) with a configurable granularity(between 1m to 1h), in order to monitor and bill the usage of the Services.

As a Storage Account Owner, or Data Consumer, I want to have a near real-time and accurate view of Storage Account service consumption metrics grouped by service consumption metrics Groups(described below) with a configurable granularity(between 1m to 1h), in order to observe and monitor the usage of the Services.

Service consumption metrics:

* Storage capacity
* Number of objects
* Network utilization transferred ingoing
* Network utilization transferred outgoing
* Number of operations

Service consumption metrics groups:

* Service
* Accounts
* Users
* Buckets
* Operation types
* Region <sup>1<sup>
* Source IP to allow our customer to bill according to source network <sup>1<sup>
* Storage class <sup>2<sup>

Operation types [extracted from AWS S3 operations][aws_s3_operations]:

* AbortMultipartUpload
* CompleteMultipartUpload
* CopyObject
* CreateBucket
* CreateMultipartUpload
* DeleteBucket
* DeleteBucketAnalyticsConfiguration
* DeleteBucketCors
* DeleteBucketEncryption
* DeleteBucketInventoryConfiguration
* DeleteBucketLifecycle
* DeleteBucketMetricsConfiguration
* DeleteBucketPolicy
* DeleteBucketReplication
* DeleteBucketTagging
* DeleteBucketWebsite
* DeleteObject
* DeleteObjects
* DeleteObjectTagging
* DeletePublicAccessBlock
* GetBucketAccelerateConfiguration
* GetBucketAcl
* GetBucketAnalyticsConfiguration
* GetBucketCors
* GetBucketEncryption
* GetBucketInventoryConfiguration
* GetBucketLifecycle
* GetBucketLifecycleConfiguration
* GetBucketLocation
* GetBucketLogging
* GetBucketMetricsConfiguration
* GetBucketNotification
* GetBucketNotificationConfiguration
* GetBucketPolicy
* GetBucketPolicyStatus
* GetBucketReplication
* GetBucketRequestPayment
* GetBucketTagging
* GetBucketVersioning
* GetBucketWebsite
* GetObject
* GetObjectAcl
* GetObjectLegalHold
* GetObjectLockConfiguration
* GetObjectRetention
* GetObjectTagging
* GetObjectTorrent
* GetPublicAccessBlock
* HeadBucket
* HeadObject
* ListBucketAnalyticsConfigurations
* ListBucketInventoryConfigurations
* ListBucketMetricsConfigurations
* ListBuckets
* ListMultipartUploads
* ListObjects
* ListObjectsV2
* ListObjectVersions
* ListParts
* PutBucketAccelerateConfiguration
* PutBucketAcl
* PutBucketAnalyticsConfiguration
* PutBucketCors
* PutBucketEncryption
* PutBucketInventoryConfiguration
* PutBucketLifecycle
* PutBucketLifecycleConfiguration
* PutBucketLogging
* PutBucketMetricsConfiguration
* PutBucketNotification
* PutBucketNotificationConfiguration
* PutBucketPolicy
* PutBucketReplication
* PutBucketRequestPayment
* PutBucketTagging
* PutBucketVersioning
* PutBucketWebsite
* PutObject
* PutObjectAcl
* PutObjectLegalHold
* PutObjectLockConfiguration
* PutObjectRetention
* PutObjectTagging
* PutPublicAccessBlock
* RestoreObject
* SelectObjectContent
* UploadPart
* UploadPartCopy

Footnotes:

1. not available in the current UTAPI design
2. not available in the Scality Products

### CLI

As a Storage Manager, I want a CLI in order to extract Storage Account's service consumption metrics in order to bill my Storage Account Owner programmatically.

As a Storage Account Owner, I want a CLI in order to extract my Storage Account service consumption metrics in order to create a report programmatically.

### API
As a Storage Manager, I want an API to extract Storage Account's service consumption metrics in order to bill my Storage Account programmatically.

As a Storage Account Owner, I want an API in order to extract my Storage Account service consumption metrics in order to create a report programmatically.

### UI

As a Storage Manager, I want to have a near real-time display of a Storage Account service consumption metrics in order to observe the usage of a Storage Account.

As a Storage Account Owner, I want to have a near real-time display of my Storage Account service consumption metrics in order to control my usage.

As a Storage Account Owner, I want to extract service consumption metrics displayed within the UI choosing the format(e.g: JSON or CSV) and the list of metrics in order to quickly extract the data and create my customer report.

### Prometheus Exporter

As a Storage Manager, I want a Prometheus Exporter for service consumption metrics in order to pull metrics from a Prometheus server.

As a Storage Manager, I want Prometheus Exporter metrics name prefixed by the exporter name and written in [snake_case][snake_case_description] in order to allow someone who is familiar with Prometheus but not our system to make a good guess as to what a metric means.

As a Storage Manager, I want Prometheus Exporter metrics in base units (e.g. seconds, bytes) and ratio instead of percentage in order to leave converting them to something more readable to graphing tools and comply with the [Writing Exporters best practices][writing_exporters].

### Configuration

As a Storage Administrator, I want to configure the granularity of the Storage Consumption metrics knowing the impacts on my Solution Instance in order to bill my customers according to my company's reporting and/or billing systems.

As a Storage Administrator, I want to configure a downsampling period of service consumption metrics on my Solution Instance knowing the impacts in order to reduce the footprint of the service consumption metrics by reducing the precision of the oldest metrics.

As a Storage Administrator, I want to configure a retention period of service consumption metrics on my Solution Instance knowing the impacts in order to reduce the footprint of the service consumption metrics by deleting oldest metrics.

### Backup/Restore

As a Storage Administrator, I want to be able to backup UTAPI configuration, in order to be able to retrieve it if I have an issue after a configuration change

As a Storage Administrator, I want to be able to version backup configuration, in order to be able to retrieve it if I have an issue after a configuration change.

As a Storage Administrator, I want to be alerted of any impacting configuration change, in order to be able to react.

### Tests

The new design of the UTAPI must be tested with the use-cases of our customers that are facing issues with the current design.

Scenarios :

* few objects within 150 million buckets
* 35 billion objects within a bucket
* 8000 op/s
* 1000 Accounts
* buckets with the versionning feature activated
* buckets with CRR feature activated
* buckets with a lifecycle policie activated

### Documentation

As a Storage Manager, I want to know how the service consumption metrics are computed in order to create my service offer on top of Scality Products.

As a Storage Manager, I want documentation where I can try out the API in order to accelerate the integration of the UTAPI in my company information system.

### Updagrade/Downgrade

As a Storage Administrator, I want to upgrade my platform keeping existing metrics in order to take advantage of the UTAPI redesign.

## Open question

* do we have to add metrics for CANCELLED operations?

## Appendices

## Divers
### Billing and Usage Reports

This table explain all the metrics that are available for an AWS Account Owner with the associated granularity and the units.
source [Understanding Billing and Usage Reports][aws_billing_usage_types]
UsageType|Units|Granularity|Description
---|---|---|---
region1-region2-AWS-In-Bytes|Bytes|Hourly|The amount of data transferred in to AWS Region1 from AWS Region2
region1-region2-AWS-Out-Bytes|Bytes|Hourly|The amount of data transferred from AWS Region1 to AWS Region2
region-BatchOperations-Jobs|Count|Hourly|The number of S3 Batch Operations jobs performed.
region-BatchOperations-Objects|Count|Hourly|The number of object operations performed by S3 Batch Operations.
region-DataTransfer-In-Bytes|Bytes|Hourly|The amount of data transferred into Amazon S3 from the internet
region-DataTransfer-Out-Bytes|Bytes|Hourly|The amount of data transferred from Amazon S3 to the internet1
region-C3DataTransfer-In-Bytes|Bytes|Hourly|The amount of data transferred into Amazon S3 from Amazon EC2 within the same AWS Region
region-C3DataTransfer-Out-Bytes|Bytes|Hourly|The amount of data transferred from Amazon S3 to Amazon EC2 within the same AWS Region
region-S3G-DataTransfer-In-Bytes|Bytes|Hourly|The amount of data transferred into Amazon S3 to restore objects from S3 Glacier or S3 Glacier Deep Archive storage
region-S3G-DataTransfer-Out-Bytes|Bytes|Hourly|The amount of data transferred from Amazon S3 to transition objects to S3 Glacier or S3 Glacier Deep Archive storage
region-DataTransfer-Regional-Bytes|Bytes|Hourly|The amount of data transferred from Amazon S3 to AWS resources within the same AWS Region
StorageObjectCount|Count|Daily|The number of objects stored within a given bucket
region-CloudFront-In-Bytes|Bytes|Hourly|The amount of data transferred into an AWS Region from a CloudFront distribution
region-CloudFront-Out-Bytes|Bytes|Hourly|The amount of data transferred from an AWS Region to a CloudFront distribution
region-EarlyDelete-ByteHrs|Byte-Hours2|Hourly|Prorated storage usage for objects deleted from, S3 Glacier storage before the 90-day minimum commitment ended3
region-EarlyDelete-GDA|Byte-Hours2|Hourly|Prorated storage usage for objects deleted from S3 Glacier Deep Archive storage before the 180-day minimum commitment ended 3
region-EarlyDelete-SIA|Byte-Hours|Hourly|Prorated storage usage for objects deleted from STANDARD_IA before the 30-day minimum commitment ended4
region-EarlyDelete-ZIA|Byte-Hours|Hourly|Prorated storage usage for objects deleted from ONEZONE_IA before the 30-day minimum commitment ended4
region-EarlyDelete-SIA-SmObjects|Byte-Hours|Hourly|Prorated storage usage for small objects (smaller than 128 KB) that were deleted from STANDARD_IA before the 30-day minimum commitment ended4|region-EarlyDelete-ZIA-SmObjects|Byte-Hours|Hourly|Prorated storage usage for small objects (smaller than 128 KB) that were deleted from ONEZONE_IA before the 30-day minimum commitment ended4
region-Inventory-ObjectsListed|Objects|Hourly|The number of objects listed for an object group (objects are grouped by bucket or prefix) with an inventory list
region-Requests-S3 Glacier-Tier1|Count|Hourly|The number of PUT, COPY, POST, InitiateMultipartUpload, UploadPart, or CompleteMultipartUpload requests on S3 Glacier objects
region-Requests-S3 Glacier-Tier2|Count|Hourly|The number of GET and all other requests not listed on S3 Glacier objects
region-Requests-SIA-Tier1|Count|Hourly|The number of PUT, COPY, POST, or LIST requests on STANDARD_IA objects
region-Requests-ZIA-Tier1|Count|Hourly|The number of PUT, COPY, POST, or LIST requests on ONEZONE_IA objects
region-Requests-SIA-Tier2|Count|Hourly|The number of GET and all other non-SIA-Tier1 requests on STANDARD_IA objects
region-Requests-ZIA-Tier2|Count|Hourly|The number of GET and all other non-ZIA-Tier1 requests on ONEZONE_IA objects
region-Requests-Tier1|Count|Hourly|The number of PUT, COPY, POST, or LIST requests for STANDARD, RRS, and tags
region-Requests-Tier2|Count|Hourly|The number of GET and all other non-Tier1 requests
region-Requests-Tier3|Count|Hourly|The number of lifecycle requests to S3 Glacier or S3 Glacier Deep Archive and standard S3 Glacier restore requests
region-Requests-Tier4|Count|Hourly|The number of lifecycle transitions to INTELLIGENT_TIERING, STANDARD_IA, or ONEZONE_IA storage
region-Requests-Tier5|Count|Hourly|The number of Bulk S3 Glacier restore requests
region-Requests-GDA-Tier1|Count|Hourly|The number of PUT, COPY, POST, InitiateMultipartUpload, UploadPart, or CompleteMultipartUpload requests on DEEP Archive objects
region-Requests-GDA-Tier2|Count|Hourly|The number of GET, HEAD, and LIST requests
region-Requests-GDA-Tier3|Count|Hourly|The number of S3 Glacier Deep Archive standard restore requests
region-Requests-GDA-Tier5|Count|Hourly|The number of Bulk S3 Glacier Deep Archive restore requests
region-Requests-Tier6|Count|Hourly|The number of Expedited S3 Glacier restore requests
region-Bulk-Retrieval-Bytes|Bytes|Hourly|The number of bytes of data retrieved with Bulk S3 Glacier or S3 Glacier Deep Archive requests
region-Requests-INT-Tier1|Count|Hourly|The number of PUT, COPY, POST, or LIST requests on INTELLIGENT_TIERING objects
region-Requests-INT-Tier2|Count|Hourly|The number of GET and all other non-Tier1 requests for INTELLIGENT_TIERING objects
region-Select-Returned-INT-Bytes|Bytes|Hourly|The number of bytes of data returned with Select requests from INTELLIGENT_TIERING storage
region-Select-Scanned-INT-Bytes|Bytes|Hourly|The number of bytes of data scanned with Select requests from INTELLIGENT_TIERING storage
region-EarlyDelete-INT|Byte-Hours|Hourly|Prorated storage usage for objects deleted from INTELLIGENT_TIERING before the 30-day minimum commitment ended
region-Monitoring-Automation-INT|Objects|Hourly|The number of unique objects monitored and auto-tiered in the INTELLIGENT_TIERING storage class
region-Expedited-Retrieval-Bytes|Bytes|Hourly|The number of bytes of data retrieved with Expedited S3 Glacier requests
region-Standard-Retrieval-Bytes|Bytes|Hourly|The number of bytes of data retrieved with standard S3 Glacier or S3 Glacier Deep Archive requests
region-Retrieval-SIA|Bytes|Hourly|The number of bytes of data retrieved from STANDARD_IA storage
region-Retrieval-ZIA|Bytes|Hourly|The number of bytes of data retrieved from ONEZONE_IA storage
region-StorageAnalytics-ObjCount|Objects|Hourly|The number of unique objects in each object group (where objects are grouped by bucket or prefix) tracked by storage analytics
region-Select-Scanned-Bytes|Bytes|Hourly|The number of bytes of data scanned with Select requests from STANDARD storage
region-Select-Scanned-SIA-Bytes|Bytes|Hourly|The number of bytes of data scanned with Select requests from STANDARD_IA storage
region-Select-Scanned-ZIA-Bytes|Bytes|Hourly|The number of bytes of data scanned with Select requests from ONEZONE_IA storage
region-Select-Returned-Bytes|Bytes|Hourly|The number of bytes of data returned with Select requests from STANDARD storage
region-Select-Returned-SIA-Bytes|Bytes|Hourly|The number of bytes of data returned with Select requests from STANDARD_IA storage
region-Select-Returned-ZIA-Bytes|Bytes|Hourly|The number of bytes of data returned with Select requests from ONEZONE_IA storage
region-TagStorage-TagHrs|Tag-Hours|Daily|The total of tags on all objects in the bucket reported by hour
region-TimedStorage-ByteHrs|Byte-Hours|Daily|The number of byte-hours that data was stored in STANDARD storage
region-TimedStorage-S3 GlacierByteHrs|Byte-Hours|Daily|The number of byte-hours that data was stored in S3 Glacier storage
region-TimedStorage-GlacierStaging|Byte-Hours|Daily|The number of byte-hours that data was stored in S3 Glacier staging storage
region-TimedStorage-GDA-ByteHrs|Byte-Hours|Daily|The number of byte-hours that data was stored in S3 Glacier Deep Archive storage
region-TimedStorage-GDA-Staging|Byte-Hours|Daily|The number of byte-hours that data was stored in S3 Glacier Deep Archive staging storage
region-TimedStorage-INT-FA-ByteHrs|Byte-Hours|Daily|The number of byte-hours that data was stored in the frequent access tier of INTELLIGENT_TIERING storage
region-TimedStorage-INT-IA-ByteHrs|Byte-Hours|Daily|The number of byte-hours that data was stored in the infrequent access tier of INTELLIGENT_TIERING storage
region-TimedStorage-RRS-ByteHrsb|Byte-Hours|Daily|The number of byte-hours that data was stored in Reduced Redundancy Storage (RRS) storage
region-TimedStorage-SIA-ByteHrs|Byte-Hours|Daily|The number of byte-hours that data was stored in STANDARD_IA storage
region-TimedStorage-ZIA-ByteHrs|Byte-Hours|Daily|The number of byte-hours that data was stored in ONEZONE_IA storage
region-TimedStorage-SIA-SmObjects|Byte-Hours|Daily|The number of byte-hours that small objects (smaller than 128 KB) were stored in STANDARD_IA storage
region-TimedStorage-ZIA-SmObjects|Byte-Hours|Daily|The number of byte-hours that small objects (smaller than 128 KB) were stored in ONEZONE_IA storage

[writing_exporters]: https://prometheus.io/docs/instrumenting/writing_exporters/
[snake_case_description]: https://en.wikipedia.org/wiki/Snake_case
[aws_billing_usage_types]: https://docs.aws.amazon.com/AmazonS3/latest/dev/aws-usage-report-understand.html
[aws_s3_operations]: https://docs.aws.amazon.com/AmazonS3/latest/API/API_Operations_Amazon_Simple_Storage_Service.html
