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

* Create a bucket with name `encrypted-bucket`, and another one `unencrypted-bucket`
* With the AWS CLI:

  * Create a bucket encryption configuration in JSON format:

  ```
  cat > encryption.json <<EOF
  {
    "Rules": [
      {
        "ApplyServerSideEncryptionByDefault": {
            "SSEAlgorithm": "AES256"
        }
      }
    ]
  }
  EOF
  ```

  * Apply the bucket encryption configuration to the encrypted bucket:

  ```
  aws s3api put-bucket-encryption \
      --bucket encrypted-bucket-1 \
      --server-side-encryption-configuration file://encryption.json
  ```

  * Create an empty object `empty-obj` in the encrypted bucket
  * Create a 1KB object `1kb-obj` in the encrypted bucket with random data
  * Create a 1MB object `1mb-obj` in the encrypted bucket with random data
  * Create a 1GB MPU object `1gb-obj` in the encrypted bucket with random data
  * Do a HEAD on each object
  * Do a GET on each object
  * Create a 1KB object `1kb-encrypted-obj` in the unencrypted bucket,
    provide the `--sse-customer-algorithm=AES256` option to `aws`
    command
  * Create a 1GB MPU object `1gb-encrypted-obj` in the unencrypted
    bucket, provide the `--sse-customer-algorithm=AES256` option to
    `aws` command
  * Do a HEAD on those two objects
  * Do a GET on those two objects

#### Expected Results

* Bucket encryption configuration was successfully applied
* Each object metadata from the HEAD request shows that it is encrypted using AES256.
* Each object data from the GET matches what has been put.
* Fetch bucket metadata (e.g. from the raft log), look at the
  `serverSideEncryption.masterKeyId` value, check on the web interface
  of the CipherTrust app, under "Key & Access Management -> Keys" that
  a key named `ks-{{masterKeyId}}` has been created
* Sproxyd keys contain a different (encrypted) payload than the
  payload of the uploaded object
* Cloudserver and Gemalto plugin logs show no error nor warning
* KMS logs (in `/opt/keysecure/logs/keysecure.system.log`) show no
  error during operations

## Upgrade Tests

### Encryption must still work for old buckets after upgrade

#### Actions

* Install S3C 7.9
* Create an encrypted bucket the old way (`create_encrypted_bucket.js`)
* Upgrade to S3C 7.10

#### Expected Results

* Encryption must still work for the old bucket

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

## Failure Tests

### KMS Single Instance Failure, Object Operations

#### Actions

* Setup a KMS instance on the S3C deployment
* Create a bucket `encrypted-bucket` with a default encryption of type AES256
* run a 128-worker cosbench workload that does a mixed workload of 33%
  PUT, 33% GET, 33% DELETE continuously for about 10 minutes, with
  size 32kB
* Stop the KMS instance during the cosbench test, for one minute
* Trigger a bucket listing
* Restart the KMS instance
* DELETEs the remaining objects

#### Expected Results

* Cosbench shows errors in the run summary due to the single-instance
  KMS being down
* There should be errors in cloudserver logs, but no exception or
  crash should have occurred
* The bucket listing triggered while the KMS was down should have
  succeeded

### KMS Single Instance Failure, Bucket Creation

#### Actions

* Setup a KMS instance on the S3C deployment
* Stop the KMS instance
* Create a bucket `encrypted-bucket` with a default encryption of type AES256

#### Expected Results

* The bucket cannot be created successfully due to the KMS instance
  being down
* There should be errors in cloudserver logs, but no exception or
  crash should have occurred

### KMS Multi Instance Failure

Note: not planned for now as part of 7.10 release as discussion about
load balancing has not been clarified: should we support and test
failover in S3C, or rely on a load balancer setup by the customer that
does the failover, in which case we rely on a single highly available
KMS endpoint?

#### Actions

* Setup a KMS cluster with 3 instances on the S3C deployment
* Create a bucket `encrypted-bucket` with a default encryption of type AES256
* run a 128-worker cosbench workload that does a mixed workload of 33%
  PUT, 33% GET, 33% DELETE continuously for about 10 minutes, with
  size 32kB
* Stop one of the KMS instances during the cosbench test, for one minute
* Restart the stopped KMS instance
* DELETEs the remaining objects

#### Expected Results

* Cosbench should show no error in the run summary due to a
  single-instance KMS being down
* There should not be client errors in cloudserver logs, only internal
  logs describing the fallback situation
