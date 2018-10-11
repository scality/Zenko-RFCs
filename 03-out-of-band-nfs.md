# ZIP#3 (Zenko Improvement Proposal) Out-of-Band NFS Metadata Ingestion

## Overview

This document is a proposal for a new feature that allows for metadata ingestion of filesystem data through NFS mounts. It will continuously sync both pre-existing and newly created files on a filesystem, and allow their management through Zenko.

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

Utilizing rclone, each CronJob will deploy a pod that has the duty of syncing the metadata from the NFS mount to Cloudserver on the schedule provided. These jobs will run until completion and will not schedule another job until the previous has finished unless the manager determines that the job needs to be re-run earlier based on the contents of the geosync log.

## Alternatives

#### Mount the NFS export directly into the pods.

- **Option 1: Requires special privileges**
  With SYS_ADMIN or similar capabilities applied to each of the cloudserver containers, they would be able to directly “mount” to an NFS directly which would greatly simplify configuration. However, this opens the door to potentially severe security risk because cloudserver is an externally exposed service (in fact, the only one).

- **Option 2: A userspace NFS client**
  With a userland NFS client, such as [nfusr](https://github.com/facebookincubator/nfusr), the Cloudserver pods would be able to (ideally) safely mount the the NFS exports. This is based on the assumption that a userspace NFS mounts doesn’t require privileges. However, this hasn’t been fully validated.

## Design Diagram

```
          +---------------------------------------------------------------------------------+
          |                                                                                 |
          |                  +-------------------------------------------------------+      |
          |                  |                                                       |      |
+-------+ | +----------------v-----------------+             +--------------+        |      |
|       | | |                                  |             |              |        |      |
|       | | |                                  |             |              |        |      |
| Orbit +--->           CloudServer            <-------------+ CloudServer  |        |      |
| (UI)  | | |            Pods (x50)            |             | Deployment   |   +----v----+ |
|       | | |                                  |             |              |   |         | |
|       | | |                                  |             |              |   |         | |
+-------+ | +------^-------------------^---^---+             +-------^------+   |         | |
          |        |                   |   |                         |          | MongoDB | |
          |        |                   |   |    +-----------+        |          |         | |
          |        |                   |   +----+Rclone Jobs<----+   |          |         | |
          |        |                   |        +-----------+    |   |          |         | |
          |        |                   |                         |   |          +----^----+ |
          | +------v-------+    +------v-------+                 |   |               |      |
          | |              |    |              |                 |   |               |      |
          | | /export1 PVC |    | /export2 PVC <------+          |   |               |      |
          | |              |    |              |      |      +---+---+------+        |      |
          | +------^-------+    +------^-------+      |      |              | Online |      |
          |        |                   |              +------+              <--------+      |
          |        |                   |                     |  Atmoshpere  |               |
          | +------v-------+    +------v-------+      +------+              <--------+      |
          | |              |    |              |      |      |              | Offline|      |
          | |    NFS PV    |    |    NFS PV    |      |      +--------------+        |      |
          | |   /export1   |    |   /export2   <------+                              |      |
          | |              |    |              |                   Zenko             |      |
          | +------^-------+    +------^-------+                   (k8s)             |      |
          |        |                   |                                             |      |
          +---------------------------------------------------------------------------------+
                   |                   |                                             |
          +---------------------------------------------------------+          +-----+------+
          |        |                   |                            |          |            |
          | +------v-------+    +------v-------+                    |          |            |
          | |   /export1   |    |   /export2   |     NFS Server     |          |  Helm/Cli  |
          | +--------------+    +--------------+                    |          |            |
          |                                                         |          |            |
          +---------------------------------------------------------+          +------------+
```
























