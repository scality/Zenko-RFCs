# Bucket Encryption Test Plan

## Deployment

### Deployment through Federation

#### Actions

* Deploy the KMS appliance to test in a single instance using [these instructions]
  (https://documentation.scality.com/S3C/7.9.0/installation/install_s3c/Configuring_the_S3_Cluster/additional_features/encryption+key_mgmt_configuration.html).

#### Expected Results

* Successful deployment
* The KMS is up and running

## Functional

### Bucket Encryption Correctness

#### Actions

* Create a bucket with name `encrypted-bucket`
* With the AWS CLI:

  * Create a bucket encryption configuration in JSON format:

  ```
  {
    "Rules": [
      {
        "ApplyServerSideEncryptionByDefault": {
            "SSEAlgorithm": "AES256"
        }
      }
    ]
  }
  ```

  * Apply the bucket encryption configuration to the bucket:

  ```
  aws s3api put-bucket-encryption --bucket encrypted-bucket-1 --server-side-encryption-configuration file:///tmp/encryption.json
  ```

  * Create an empty object `empty-obj` in the bucket
  * Create a 1KB object `1kb-obj` in the bucket with random data
  * Create a 1MB object `1mb-obj` in the bucket with random data
  * Create a 1GB MPU object `1gb-obj` in the bucket with random data
  * Do a HEAD on each object
  * Do a GET on each object

#### Expected Results

* Bucket encryption configuration was successfully applied
* Each object metadata from the HEAD request shows that it is encrypted using AES256.
* Each object data from the GET matches what has been put.

## Performance

In the following steps, make sure that the cosbench drivers run on a separate
host to avoid any interference.

### Bandwidth

#### Actions

* Setup a KMS cluster on the S3C deployment, sized with performance in mind
* Create a bucket `unencrypted-bucket` without encryption
* Create a bucket `encrypted-bucket` with a default encryption of type AES256
* For each bucket, run a 64-worker cosbench workload that:

  * PUTs objects continuously for about 10 minutes, with size 1MB
  * GETs these objects continuously for about 10 minutes
  * DELETEs these objects

#### Expected Results

* Cosbench shows no errors in the run summary.
* Latencies are relatively stable (some variance is ok because of the big
  object size).
* R/W Bandwidth is stable.
* Either one of the following resources (depending on the platform used) is
  close to saturation:

  * Network link between the cosbench driver and cloudserver
  * Network link between cloudserver and the KMS cluster
  * Disk bandwidth
  * CPU usage of all cloudserver replicas
  * CPU usage of the KMS cluster processes
  * Cosbench driver host egress/ingress/CPU/...

* Memory usage on the node returns to idle levels (needs subjective assessment
  before final guidance can be given).
* Observe platform metrics and focus on the overhead of encryption:

  * Compare the raw bandwidth obtained by each cosbench run, with and
    without encryption, derive a raw % of overhead
  * Observe how much more CPU is consumed (cloudserver and KMS in
    particular)for the bucket with encryption enabled. Is it CPU
    bound?
  * Observe the specific overheads of:

    * PUT with encryption compared to PUT without encryption
    * GET with encryption compared to GET without encryption

* There should be no or very few errors in cloudserver logs, errors
  should be explained if any (e.g. network is saturated)
* Observe the KMS cluster logs, there should be no error, or errors
  should be identified and explained

### IOPS

#### Actions

* Setup a KMS cluster on the S3C deployment, sized with performance in mind
* Create a bucket `unencrypted-bucket` without encryption
* Create a bucket `encrypted-bucket` with a default encryption of type AES256
* For each bucket, run a 128-worker cosbench workload that:

  * PUTs objects continuously for about 10 minutes, with size 32kB
  * GETs these objects continuously for about 10 minutes
  * DELETEs these objects

#### Expected Results

* Cosbench shows no errors in the run summary.
* Latencies are relatively stable and smaller than 8ms, without much variation
  (as even 1ms variation around 8ms is 12.5%).
* R/W Bandwidth is stable, even if low.
* Either one of the following resources (depending on the platform used) is
  close to saturation:

  * Network link between the cosbench driver and cloudserver (unlikely
    with small objects)
  * Network link between cloudserver and the KMS cluster
  * Disk bandwidth
  * Disk IOPS
  * CPU usage of all cloudserver replicas
  * CPU usage of the KMS cluster processes
  * Cosbench driver host egress/ingress/CPU/...

* Memory usage on the node returns to idle levels (needs subjective assessment
  before final guidance can be given).
* Observe platform metrics and focus on the overhead of encryption:

  * Compare the raw IOPS obtained by each cosbench run, with and
    without encryption, derive a raw % of overhead
  * Observe how much more CPU is consumed (cloudserver and KMS in
    particular) for the bucket with encryption enabled. Is it CPU
    bound?
  * Observe the specific overheads of:

    * PUT with encryption compared to PUT without encryption
    * GET with encryption compared to GET without encryption

* There should be no or very few errors in cloudserver logs, errors
  should be explained if any (e.g. network is saturated)
* Observe the KMS cluster logs, there should be no error, or errors
  should be identified and explained
