# Utapi - Utilization API

## Overview

Utapi tracks metrics of a service's usage. Metrics provided by Utapi include
the number of incoming and outgoing bytes, the number of objects being stored,
the storage utilized in bytes, and a count of operations performed on a
service's resources. Operations supported by Utapi include APIs offered by
Scality's S3 Server. Metrics can be retrieved for a given time range in a
service's history.

## Problem Description

Utapi needs to provide a scalable solution to accurately track metrics across
any number of nodes while providing reliable storage of its historic data. The
system should be highly available to avoid losing data and affecting the
quality of the provided metrics. It should be able to tolerate node and service
failures and be able to repair it's data in the event of a failure.

## Technical Details

#### Ingestion Flow

```
                        +--------------------------------+
                        |         Local Redis            |
                        +--------------------------------+
                              ^                  |
                              |                  |
                              |                  v
+-------------+         +------------+   +---------------+         +-------+
| Cloudserver | +-----> |Utapi Server|   |Background Task| +-----> |Warp 10|
+-------------+         +------------+   +---------------+         +-------+

```

### **Data Model and Operations**

#### Redis

Utapi uses Redis to cache events before inserting them into Warp 10. This is to
reduce write overhead as batch inserts are more efficient. Each event is JSON
serialized and stored in a key. Based on its timestamp the event is then sharded
into a 10 second block and its key is added to a corresponding Set.

**Key Schema**

Events are stored using `<prefix>:events:<event uuid>`
Shards are stored using `<prefix>:shards:<timestamp>`
`timestamp` is a UNIX style timestamp indicating the beginning of the shard

#### Warp 10

Warp 10 is used for long term storage and analysis of events in Utapi. In Warp 10
parlance each event is a `measure` consisting of a timestamp, `class`, key value
pair `labels`, and an associated value.

Utapi defines several classes:

 * `utapi.events` used for ingested events
 * `utapi.checkpoints` used for incremental checkpoints of events
 * `utapi.checkpoints.master` used to record checkpoint creation
 * `utapi.snapshots` used for incremental snapshots of checkpoints
 * `utapi.snapshots.master` used to record snapshot creation
 * `utapi.repairs` 
 * `utapi.repairs.master` 

**High level overview**

```
   Snapshots         Checkpoints            Events

 +-----------+      +-----------+      +-------------+

                                         +---------+
                                         |   -1    |
                                         +---------+
                                         +---------+
                                         |   -1    |
                                         +---------+
                      +-------+ <------+ +---------+
                      |  -3   |    |     |   -1    |
                      +-------+    |     +---------+
                                   +---+ +---------+
                                   |     |   -1    |
                                   |     +---------+
                                   +---+ +---------+
                                         |   -1    |
                                         +---------+
   +-------+ <------+ +-------+ <------+ +---------+
   |   5   |    |     |  -1   |    |     |   -1    |
   +-------+    |     +-------+    |     +---------+
                |                  +---+ +---------+
                |                  |     |   -1    |
                |                  |     +---------+
                |                  +---+ +---------+
                |                        |   +1    |
                |                        +---------+
                +---+ +-------+ <------+ +---------+
                |     |  +3   |    |     |   +1    |
                |     +-------+    |     +---------+
                |                  +---+ +---------+
                |                  |     |   +1    |
                |                  |     +---------+
                |                  +---+ +---------+
                |                        |   +1    |
                |                        +---------+
                +---+ +-------+ <------+ +---------+
                      |  +3   |    |     |   +1    |
                      +-------+    |     +---------+
                                   +---+ +---------+
                                   |     |   +1    |
                                   |     +---------+
                                   +---+ +---------+
                                         |   +1    |
                                         +---------+

```

#### Events

Utapi stores four integers, using Warp 10's multivariate data type, for every
operation it ingests, `objectDelta`, `bytesDelta`, `ingress`, and `egress`.
They reflect how an operation affects the computed metric counters and can be
both positive and negative.

For example, an upload of a 100 byte object to a new key would have the values:

```json
{
  "objectDelta": 1,
  "bytesDelta": 100,
  "ingress": 100,
  "egress": 0
}
```

Deleting the same object would have

```json
{
  "objectDelta": -1,
  "bytesDelta": -100,
  "ingress": 0,
  "egress": 0
}
```

Each event can also have attached metadata such as the `account` , `bucket`,
or `location` of the affected resource. These are attached as Warp 10 `labels`
to the event and are used during checkpoint creation to create the various
levels of metrics.

#### Checkpoints

Checkpoints are the first step in converting the stream of `events` into a form
we can use to compute metrics. They are created using a timestamp and a list of
`label` names to index. Each provided `label` creates a "level" of metrics and
a checkpoint will be created for each unique `label` name/value pair
encountered.  Starting at the end timestamp a previously created master
checkpoint is search for, its timestamp will be used for the start of the
checkpointed range. If no previous master checkpoint is found a timestamp of
`0` is used, causing all events to be scanned. Events during the calculated
time range are retrieved and iterated over. If any of the specified `labels`
are seen the event's values are taken and summed with any previous values
matching the same `label` value. A count of each value of the label
`operationId` is also kept for every checkpoint.

For example, the following `events` (shown in `GTS` format for brevity) if
used with the label `bucket` will produce two checkpoints tagged with `bucket0`
and `bucket1` respectively.

```
1000000000000000 utapi.events{operatioinId=putObject bucket=bucket0 key=obj0} [ 1 100 100 0 ]
1000000000000001 utapi.events{operatioinId=putObject bucket=bucket0 key=obj1} [ 1 100 100 0 ]
1000000000000002 utapi.events{operatioinId=putObject bucket=bucket1 key=obj0} [ 1 100 100 0 ]
1000000000000003 utapi.events{operatioinId=putObject bucket=bucket1 key=obj2} [ 1 100 100 0 ]
1000000000000003 utapi.events{operatioinId=deleteObject bucket=bucket1 key=obj2} [ -1 -100 0 0 ]
```

```
1000000000000010 utapi.checkpoints{bucket=bucket0} [ 2 200 200 0 { "ops": { "putObject": 2 } } ]
1000000000000010 utapi.checkpoints{bucket=bucket1} [ 1 100 200 0 { "ops": { "putObject": 2, "deleteObject": 1 } } ]
```

After the `events` are scanned each checkpoint is stored as a measure in Warp 10
using the class `utapi.checkpoints`. A single label is attached with the label
name and value being the level and group of the checkpoint (ie `bucket` and
`bucket0`) from the example above. This allows us to easily query for a
specific level's checkpoints. Once all checkpoints have been stored a "master"
checkpoint is created with the class `utapi.checkpoints.master` The passed end
timestamp is used for the master checkpoint along with any created checkpoints
to signify the end of the scanned range.

#### Snapshots

Snapshots are used to convert the delta style checkpoints into concrete numbers
and provide a starting point for calculating the final metrics rather than
processing every checkpoint. Snapshots are created using only an end timestamp
signifying the end of the time range of included checkpoints. Like checkpoint
creation the latest master snapshot is searched for and all checkpoints with
timestamp lying within the master snapshots timestamp and the passed timestamp
are fetched. If no previous master snapshots are found a timestamp of `0` is
used causing all checkpoints to be included. The checkpoints are then iterated
over and summed together based upon their labels using the values from the
previous snapshot as a base. 

For example the two checkpoints from the previous example:

```
1000000000000010 utapi.checkpoints{bucket=bucket0} [ 2 200 200 0 { "ops": { "putObject": 2 } } ]
1000000000000010 utapi.checkpoints{bucket=bucket1} [ 1 100 200 0 { "ops": { "putObject": 2, "deleteObject": 1 } } ]
```

Will produce these two snapshots:

```
1000000000000010 utapi.snapshots{bucket=bucket0} [ 2 200 200 0 { "ops": { "putObject": 2 } } ]
1000000000000010 utapi.snapshots{bucket=bucket1} [ 1 100 200 0 { "ops": { "putObject": 2, "deleteObject": 1 } } ]
```

#### Repairs

In a distributed system node or network failures can cause events to not be
immediately ingested into Utapi and can cause problems for a system rely on a
static ordering of historic events. To guard against this Utapi implements a
system for asynchronous repairs of checkpoints and snapshots. During the
Redis -> Warp 10 transition Utapi performs some sanity checks to determine if
the inserted metrics could have already included in a checkpoint. If so the
repair process is started using the beginning and end of the time range of the
inserted events as the beginning and end of the repair range. Effected
checkpoints are fetched and recalculated to include the new events. Snapshots
including the updated checkpoints are then fetched and recalculated. 

```
   +-------+ <--+---+ +-------+ <--+---+ +---------+
   |   3   |    |     |  -1   |    |     |   -1    |
   +-------+    |     +-------+    |     +---------+
       ^        |                  +---+ +---------+
       |        |                  |     |   -1    |
    Snapshot    |                  |     +---------+
    updated     |                  +---+ +---------+
                |                        |   +1    |
                |                        +---------+
                +---+ +-------+ <--+---+ +---------+
                |     |  +1   |    |     |   +1    |
Checkpoint +--------> +-------+    |     +---------+
is updated      |                  +---+ +---------+
                |                  |     |   +1    |
                |                  |     +---------+
                |                  |---+ +---------+
                |                  |     |   -1    | <----+ New events are inserted
                |                  |     +---------+      |
                |                  |---+ +---------+      |
                |                  |     |   -1    | <----+
                |                  |     +---------+
                |                  +---+ +---------+
                |                        |   +1    |
                |                        +---------+
                +---+ +-------+ <--+---+ +---------+
                      |  +3   |    |     |   +1    |
                      +-------+    |     +---------+
                                   +---+ +---------+
                                   |     |   +1    |
                                   |     +---------+
                                   +---+ +---------+
                                         |   +1    |
                                         +---------+
```

### **Failure Recovery**

Failure scenarios are mitigated using caching with retries at various levels of
the system with specially formatted logs messages being used as a last resort.
Below are some specific failure scenarios that have been considered.

**Local Redis Unavailable**

In the case of the local redis being unavailable Cloudserver will cache events
in memory up to a limit and then will begin flushing events as special log
messages. After redis is available, events will be flushed and a repair
triggered if needed.

**Warp 10 Unavailable**

If Warp 10 is unavailable events are persisted in redis until it
becomes available and then batch inserted.

### Migrations

Migrations from previous customer deployments will require a post upgrade
tool to be used. This will migrate the historic utapi data in Redis into
Warp 10 and make it available under the new system..

### Soft Limit

A specific disk usage monitoring process will periodically check the
disk space available on the Warp10 data partition. If the soft
limit is reached then some records will be deleted and some physical
space will be reclaimed whenever possible.

1. A first parameter will be how often the monitoring process will be launched (e.g 5mn).
2. A second parameter will be the actual soft limit expressed in size (e.g. 300GB).
3. A third parameter will define the retention period expressed in days (e.g. 60 days).

Definitions:
- LevelDB: Warp10 uses LevelDB as a storage.
- SST file: An SST file is an immutable data file of LevelDB of 2MB.
- Blocks: An SST file contains blocks of 4K. Since we use Snappy as a block compression engine, the size of our blocks are ~1.5K compressed. Therefore there are about ~1500 blocks in a SST in average. Each block contains between 20 and 50 records.
- A record can be a low level Warp10 metadata or a low level Warp10 records.
- Metadata are stored under `'M' + <CLASS ID> + <LABELS ID>`.
- Records (or data points) are stored under under `'R' + <CLASS ID> + <LABELS ID> + <REVERSED TS>`.

We encode all UTAPIv2 records (snapshots, checkpoints, events, see the
exhaustive list of classes above in the previous sections) as low
level Warp10 records, and we distinguish them with CLASS IDs, e.g.
the `utapi.event` CLASS ID.

The traditional deletion process of LevelDB is "lazy" in the sense
that an actual deletion of a record will not immediately reclaim space
but write a tombstone in the DB. Eventually the record marked with a
tombstone will be physically deleted during a compaction when reaching
the final level.

This behavior is incompatible with the design of the soft limit
feature. Most situations require an immediate reclaim of disk space.

Warp10 have a specific extension called "blazing fast deletion" which
can actually examine SST files of the deepest level (with the commands
SSTREPORT and SSTFIND) and determine their key range.

Once SST files are deemed safe to delete they can be purged (with the
command SSTPURGE). The operation is inline (no need to close the DB)
and blazing fast (a few seconds) since it only synchronously rewrites
the LevelDB MANIFEST. The actual physical deletion of SST files (2MB
each) being done immediately but asynchronously.

The Warp10 format is particularly adapted in our case to reclaim
events and repair events since they constitute most of the records and
they belong to a few number of classes and labels (2 per server) and
are reverse-ordered by timestamp. We could check that >90% of SST
files contains exclusively events when the DB has at least 500 SST
files.

We also know there are some records that do not occupy enough disk
space to be easily reclaimed because they are few of them and they
will be intermixed together (e.g. metadata, snapshots and
checkpoints). Fortunately they won't consume extensive space (e.g. for
snapshots 1 SST file will be generated every 10 days).

At any given time, most of the SSTs files will be in the deepest
level, so ready to be reclaimed. And the large majority of those files
will contain timestamp ordered events (events and repair events). So
we will not attempt to delete them with a tombstone but identify those
in using SSTFIND, which will decode the boundaries keys. If the SST
files are exclusively containing events (aka non-hybrid) and their
timestamp boundaries are beyond the retention period then we will
purge them directly with SSTPURGE. The space will be immediately
reclaimed .

For all other records than events and repair events, a regular delete
tombstone process will be used, and the space will be reclaimed
eventually. If for some (unlikely) reason it is needed to force the
reclaim of the disk space used by those records, we will document a
procedure that will take the Warp10 container offline for some time
and use the command WarpCompact that will be used with a range. The
procedure could be executed server by server.

Side notes:

- The monitoring process is launched exactly once on the cluster
  thanks to the [Ballot
  launcher](https://github.com/scality/citadel/blob/development/1.0/docs/design/ballot-launcher.md).
