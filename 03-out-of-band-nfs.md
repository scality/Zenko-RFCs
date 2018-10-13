# ZIP#3 (Zenko Improvement Proposal) Out-of-Band NFS File System Ingestion

## Overview

This document is a proposal for a new feature that allows existing file system listings, content, and hierarchy to be visible to Zenko. This is done through the ingestion of filesystem attributes which will continuously sync both pre-existing and newly created files on a filesystem, and allow their management through Zenko.

## Problem Description

Currently, the only way to manage any data on Zenko that you have on a file system (SOFS, NFS, etc.) is by uploading it through the API. However, this is an unnecessarily complex procedure and it will be duplicating data rather than allowing Zenko to manipulate existing one.

## Use-cases Description

Leverage Zenko features - multi-cloud replication, metadata search, lifecycle policies, and such - on your filesystem data transparently.

## Technical Details

A manager/operator will leverage the Kubernetes API that will dynamically provision NFS PVs, PVCs, and modify the Cloudserver deployment to set up volume mounts.

#### The Manager (Atmosphere)

Having a stateless in-cluster manager that will apply a configuration specified via helm values for an updated NFS config to deploy a CronJobs per export. This manager will have Kubernetes API privileges for scheduling resources and updating configurations. Logging and metrics for the ingestion can be centrally exported via this manager. Also, the manager allows us to scale up or down the number of ingestion jobs with every additional NFS export added or removed (through orbit or the values.yaml file).

#### Accessing NFS

The manager will update the configuration of the Cloudserver containers to mount a ReadWriteMany PV that is backed by a given NFS export. This would allow for the Cloudservers to access data as soon as it is ingested. The ingestion itself would be handled by Kubernetes CronJobs defined by the manager that runs an rclone process to completion on the specified interval. This interval can be altered or delayed by the manager as needed.

#### CronJobs

Utilizing rclone, each CronJob will deploy a pod that has the duty of syncing the metadata from the NFS mount to Cloudserver on the schedule provided. Rclone will sync each object using a special header to signify the NFS backed files that includes an MD5 and size. These jobs will run until completion and will not schedule another job until the previous has finished.

Optimizations can be made in V2 to allow the manager to run sooner based on things like geosync logs for hinting and listing/attribute cache to speed up the listing.



## Alternatives

#### Mount the NFS export directly into the pods.

- **Option 1: Requires special privileges**
  With SYS_ADMIN or similar capabilities applied to each of the cloudserver containers, they would be able to directly “mount” to an NFS directly which would greatly simplify configuration. However, this opens the door to potentially severe security risk because cloudserver is an externally exposed service (in fact, the only one).

- **Option 2: A userspace NFS client**
  With a userland NFS client, such as [node-nfsc](https://github.com/scality/node-nfsc), the Cloudserver pods would be able to safely mount the the NFS exports. Reading and writing would
  Issues:
    - Performance: We would have to implement correct connection pooling and parallelism to be credible
    - NFS compatibility: it would be hard to be compatible with all NFS servers existing, compared to using the Linux in kernel NFS client which is proven
    - No support for NFSv4

- **Option 3: Live update daemon**
  - listing with prefix: compare both mtimes for the directory on the filesystem and in mongodb for that prefix (there should be a placeholder). if up to date, cloudserver proceeds with the listing from mongodb. if not uptodate, cloudserver finds the changes and update mongodb's content for that particular prefix, then proceeds with the listing (still from mongodb).
  - HEAD or GET on an object: stat(3) the targeted file, update the object's representation in mongodb if relevant, then cloudserver proceeds with serving the file
  - listing without prefix: proceed with the recursive listing of the filesystem (modulo the maxKeys parameter) and update the objects/"directory placeholders" in mongodb if relevant (I just made it up, I'm not certain about this part)

## Design Diagram

```ascii
                              +---------------------+
                              |        Orbit        |
                              +----------^----------+
                                         |
+---------------------------------------------------------------------------------+
|                                        |                                        |
|                             +----------v----------+             +-------------+ |
|                             |                     <------------->             | |
|        +-------------------->     CloudServer     |             |   MongoDB   | |
|        |                    |                     <----------+  |             | |
|        |                    +----------^----------+          |  +------+------+ |
|        |                               |                     |         |        |
|        |                               |             +-----------------+        |
|        |                    +----------v----------+  |       |                  |
|        |         +---------->   FS API Gateway    <-------------------+         |
|        |         |          +----------^----------+  |       |        |         |
|        |         |                     |             |       |        |         |
|        |         |                     |     +-------+       |        |         |
|    +---+----+----v----+                |     |          +----+---+----v----+    |
|    |        |         |     +----------+-----v----+     |        |         |    |
|    | RClone | FS-Data |     |                     |     | RClone | FS-Data |    |
|    |        |         <-----+     Atmosphere      +----->        |         |    |
|    +--------+---------+     |                     |     +--------+---------+    |
|    |       PVC        |     +----+------------+---+     |       PVC        |    |
|    +--------^---------+          |            |         +--------^---------+    |
|             |                    |            |                  |              |
|             |                    |            |                  |              |
|    +--------v---------+          |            |         +--------v---------+    |
|    |                  |          |            |         |                  |    |
|    |    In-tree PV    <----------+            +--------->  Out-of-tree PV  |    |
|    |    (i.e. NFS)    |                                 |    (i.e. SMB)    |    |
|    |                  |                                 |                  |    |
|    +--------^---------+                                 +--------^---------+    |
|             |                                                    |              |
+---------------------------------------------------------------------------------+
              |                                                    |
     +--------v---------+                                 +--------v---------+
     |                  |                                 |                  |
     |    NFS Server    |                                 |    SMB Server    |
     |                  |                                 |                  |
     +------------------+                                 +------------------+
```
























