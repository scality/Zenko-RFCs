# Object Lifecycle Management

## Overview

The purpose is to apply various lifecycle actions to objects after a
certain time has passed since object creation, per lifecycle rules as
specified in AWS S3 APIs.

### Expiration

We support expiration of versioned or non-versioned objects, when a
number of days have passed since their creation, or after a specific
date (for current versions). We can additionally filter target objects
by prefix or by tags.

One important use case is to allow automatic deletion of older
versions of versioned objects to reclaim storage space.

### Transition Policies

Transition policies allow to transition object data from one location
to another one automatically, usually to a slower but cheaper location
for old or infrequently-accessed data.

On versioned buckets, it can apply to current version data, as well as
noncurrent versions, depending on the rule.

AWS-specific transitions, e.g. to STANDARD_IA or GLACIER storage
classes is not yet supported through Zenko, if needed, it has to be
set directly on AWS.

## Design

The implementation of Object Lifecycle Management comprises several
components which, together, provide the functionality.

Most components rely on Kafka and Zookeeper to communicate with each
other.

### Workflow Overview

Here's an example workflow that expires version *v1* of object *obj*
from bucket *foobucket*:

```
 __________________
/                  \
|   S3 client      |
\__________________/
       |
       | PUT /foobucket?lifecycle
       |
 ______v___________
/                  \         ___________
|                  |        /           \
| S3 Connector     |--------|  METADATA |
|                  |        |    log    |
|                  |        \___________/
\__________________/              |
    ^    ^                        |
    |    |                        | entry: "PUT /foobucket?lifecycle"
    |    |                        |
    |    |                 _______v_________                 ___________
    |    |                /                 \  foobucket    /           \
    |    |                | Queue Populator |-------------->|           |
    |    |                \_________________/               | ZOOKEEPER |
    |    |                                   foobucket      |           |
    |    |                              /-------------------\___________/
    |    |                 _____________v___
    |    |                /                 \
    |    |                |   Conductor     | process:foobucket_________
    |    |                \_________________/-------------->/           \
    |    |                                                  |           |
    |    | GET /foobucket?lifecycle                         |   KAFKA   |
    |    | list foobucket (max n) => obj@v1                 |           |
    |    |                __________________   process:     |           |
    |    |               /                  \<--------------\___________/
    |    \---------------|   Lifecycle      |foobucket@marker  ^    ^  |
    |                    | Bucket Processor |                  |    |  |
    |                    \_____________ ____/------------------/    |  |
    |                                  |    expire:foobucket/obj@v1 |  |
    |                                  |                            |  |
    |                                  \----------------------------/  |
    | DELETE foobucket/obj@v1            process:foobucket@(marker+n)  |
    |                     __________________                           |
    \--------------------/                  \<-------------------------/
                         |    Lifecycle     |  expire:foobucket/obj@v1
                         | Object Processor |
                         \__________________/

```

### Lists of actors

#### Lifecycle Components

* Queue populator (extended as explained below)
* Conductor
* Bucket Processor
* Object Processor

#### Kafka topics for lifecycle

* backbeat-lifecycle-bucket-tasks
* backbeat-lifecycle-object-tasks (expiration only)

#### Kafka topics shared between CRR and lifecycle transitions

* backbeat-replication
* backbeat-replication-status

#### Zookeeper paths

* /[chroot_path]/lifecycle/data/buckets/${ownerId}:${bucketUID}:${bucketName}
* /[chroot_path]/lifecycle/run/backlog-metrics/${topic}/${partition}/topic
* /[chroot_path]/lifecycle/run/backlog-metrics/${topic}/${partition}/consumers/${groupId}

### CloudServer

The S3 API provides three calls to manage lifecycle properties per bucket:

* PUT Bucket Lifecycle
* GET Bucket Lifecycle
* DELETE Bucket Lifecycle

See AWS specs for details on the protocol format.

Those calls essentially manage bucket attributes related to lifecycle
behavior, which are stored as part of bucket metadata.

#### Transition policies support in CloudServer

Transition policies rely on the *preferred read location* for an
object.

We introduce an optional per-object *preferred read location*. If set,
it is used to determine which location to read the object from on a
GET request. It overrides the *preferred read location* defined in the
bucket replication configuration (if any).

The attribute may be present in the `replicationInfo` section of
object metadata, as a `"preferred": "<location>"` property, where
`<location>` is one of the replication targets listed in
`replicationInfo`. It should be in `COMPLETED` state in order to make
the object readable without specifying a location header in the GET
request.

### Queue Populator

The queue populator is an existing component of backbeat that reads
the metadata log and populates entries for CRR.

It is extended for lifecycle management, to maintain a list of buckets
which have to be processed for lifecycle actions, so that only those
buckets are processed.

The list of buckets is stored in Zookeeper nodes with the path:

```
[/chroot_path]/lifecycle/data/buckets/${ownerId}:${bucketUID}:${bucketName}
```

* **ownerId** is the canonical ID of the bucket owner

* **bucketUID** is the globally unique ID assigned to a bucket at
    creation time (avoids conflicts with buckets re-created with the
    same name)

* **bucketName** is the bucket name

The node value is not meaningful, so it's set to `null` (the JSON
`null` value stored as a binary string).

More details on actions taken when specific entries are processed from
the metadata log:

* when processing a log entry that is an update of bucket attributes,
  look at whether any lifecycle action is enabled for that bucket:

  * if any action is enabled, create the Zookeeper node (ignore if
    already exists)

  * if no action is enabled, delete the Zookeeper node (ignore if not
    found)

* when processing a log entry that is a deletion of a bucket, delete
  the Zookeeper node (ignore if not found).

The resulting Zookeeper list should contain a current view of all
buckets that have an attached and enabled lifecycle configuration, up
to the point where the metadata log has been processed.

Note that in the future, we may request the MongoDB database to get
the list of buckets with lifecycle configuration instead of using
Zookeeper nodes.

### Replication Queue Processor (RQP)

The RQP will process replication entries the same way as regular CRR,
although they may come from transition policies processing (generated
by the Lifecycle Bucket Processor).

### Replication Status Processor (RSP)

The existing Replication Status Processor is responsible for updating
object metadata after a CRR target has completed or failed.

#### Transition Policies support in Replication Status Processor

For transition policies, the RSP when processing a CRR location with a
**COMPLETED** status, checks whether this location contains the
`"transition": true` attribute. If so, it does the following extra
actions:

* set the new *preferred read location* in the object metadata, by
  adding or altering the `"preferred": "<location>"` property to the
  `replicationInfo`, where `<location>` is the name of the CRR
  location that just completed,

* mark the previous *preferred read location* as invalid since it
  will be garbage-collected (so not to be able to reuse it for further
  transitions, for example). The previous preferred read location may
  be:

  - the previous object-defined preferred read location if already set

  - otherwise the preferred read location set in the bucket
    replication configuration if any

  - otherwise the primary location of the object if no preferred read
    location exists.

* send a message to the **backbeat-gc** queue containing the data
  location attached to the previous *preferred read location* of the
  object, so that it gets garbage-collected.

### Lifecycle Conductor

The conductor role is to periodically do the following (cron-style task):

* check whether there is a pending backlog of tasks still to be
  processed from Kafka, by looking at nodes under
  `/[chroot_path]/lifecycle/run/backlog-metrics`
  * if for any topic partition for both bucket and object tasks topic
    there is an existing backlog (i.e. topic latest offset (in `topic`
    node) is strictly greater than committed consumer group offset), log
    and skip this round
* get the list of buckets with potential lifecycle actions enabled
  from Zookeeper (getChildren in Zookeeper API)
* for each of them, do the following:
  * publish a message to the Kafka topic
    *backbeat-lifecycle-bucket-tasks* (see [bucket tasks](#bucket-tasks)
    message format)

The conductor only generates messages to *start* a new listing. Later
on, if the listing is truncated, new messages are generated by the
Bucket Processor to create new tasks to continue the listing. This
way, processing of large buckets can be efficiently parallelized
across Bucket Processors (see [Lifecycle Bucket
Processor](#lifecycle-bucket-processor) section).

Note that as an optimization, when lifecycle rules only apply to a
particular object prefix in a bucket, it can be interesting to list
only the specific range belonging to this prefix. For this we can add
the necessary info in the message JSON data to limit the listing to
this range.

### Lifecycle Bucket Processor (LBP)

LBP's responsibility is to list objects in lifecycled buckets, and
publish to Kafka queues the actual lifecycle actions to execute on
objects (i.e. expiration or transition actions).

More specifically, the LBP is part of a Kafka consumer group that
processes items from the *backbeat-lifecycle-bucket-tasks* topic. For
each of them, it does the following:

* Get the lifecycle configuration from S3 connector of the target
  bucket (GET /foobucket?lifecycle)
  * If no lifecycle configuration is associated to the bucket, skip
    the rest of bucket processing

* List the object versions in the bucket, starting from the
  *keyMarker*/*versionIdMarker* specified in the Kafka entry, or from
  the beginning if not set. The limit of entries is 1000 (list
  objects hard limit), but a lower limit may be set if needed.

  * If the listing is not complete, publish a new entry to the
    *backbeat-lifecycle-bucket-tasks* topic, similar to the one
    currently processed but with the *keyMarker* and
    *versionIdMarker* fields set from what's returned by the current
    finished listing. Then another LBP instance will keep going with
    the listing and processing of this bucket.

  * For each object listed, match it against the lifecycle rules
    logic. For each lifecycle rule matching, publish to the relevant
    topic(s) the action(s) to take:

    - for expiration rules, publish to the
      *backbeat-lifecycle-object-tasks* Kafka topic an entry that tells
      the LOP to delete the object or the noncurrent version (see
      [object tasks](#object-tasks) message format).

    - for transition rules, publish to the *backbeat-replication*
      topic an object entry updated with the transition target
      location, as a **PENDING** entry plus the `"transition": true`
      property, as a new extra item in the `replicationInfo` attribute
      (note that this location does not have to be part of the bucket
      replication configuration as it is usually the case)

Periodically (every minute in the current default settings) the
internal consumer of the bucket tasks topic publishes its current
committed offset and the latest topic offset in Zookeeper under
`/[chroot_path]/lifecycle/run/backlog-metrics/...`, for checking in
the conductor.

#### Note 1

Following Amazon's behavior:

> If a replication configuration is enabled on the source bucket,
> Amazon S3 puts any lifecycle actions on hold until it marks the
> objects status as either COMPLETED or FAILED.

It's probably wise to apply the same logic, to guarantee we're not
processing lifecycle on an object version with a pending
replication. This is not currently implemented.

#### Note 2

We may do search requests on MongoDB to query the exact list of
objects where a lifecycle rule applies, instead of filtering based on
the full listing result, for better efficiency.

### Lifecycle Object Processor (LOP)

LOP's responsibility is to execute the individual lifecycle actions
per object that has been matched against lifecycle rules.

It's part of a Kafka consumer group that processes items from the
*backbeat-lifecycle-object-tasks* topic. For each of them, it does the
following:

* Check the "action" field, and execute the corresponding action:
  * for "deleteObject" action:

    * do a HEAD object with a "If-Unmodified-Since" condition on the
      last-modified date stored in the "details" of the Kafka entry

    * if the request succeeds, do a DELETE request on the object

    * NOTE: this has an intrisic race condition, we'll be working on
      implementing the If-Unmodified-Since directly on the DELETE
      operation to get rid of this race.

  * We may consider adding more types of operations in the future

Periodically (every minute in the current default settings) the
internal Kafka consumer of the object tasks topic publishes its
current committed offset and the latest topic offset in Zookeeper
under `/[chroot_path]/lifecycle/run/backlog-metrics/...`, to let the
conductor know the size of the backlog.

### Message formats

#### Bucket tasks

A message in *backbeat-lifecycle-bucket-tasks* topic represents a
listing task for one bucket, it has the following format:

```
{
    "action": "processObjects",
    "target": {
        "owner": "ownerID",
        "bucket": "bucketname"
    },
    "details": {
        ["prefix": "someprefix"]
        ["keyMarker": "somekeymarker"]
        ["versionIdMarker": "someversionidmarker"]
        ["uploadIdMarker": "someuploadidmarker"]
        ["marker": "somemarker"]
    }
}
```

* **prefix** may be set to limit the listing to a prefix
* **keyMarker** and **versionIdMarker** are set when resuming a
   listing from where it ended in a previous listing task.

#### Object tasks

The message format for the *backbeat-lifecycle-object-tasks* Kafka
topic is the following:

```
{
    "action": "actionname",
    "target": {
        "owner": "ownerID",
        "bucket": "bucketname",
        "key": "objectkey",
        "version": "objectversion"
    }
    "details": {
        "lastModified": "..."
        [...]
    }
}
```

* **action**: type of lifecycle action, e.g. "deleteObject"
* **details**: additional info related to the action

## Links

* AWS Lifecycle Reference:
  https://docs.aws.amazon.com/AmazonS3/latest/dev/object-lifecycle-mgmt.html
