<h1 align="center">
ZIP#3 (Zenko Improvement Proposal)<br>The "Cosmos" Storage Backend
</h1>

### Overview

This document is a proposal for an extensible framework (a.k.a Cosmos) that will allow Zenko to manage data stored on various kinds of *backends* such as filesystems, block storage devices, and any other storage platform to which there is an available Kubernetes PersistentVolume plugin. Pre-existing data on these storage systems and data not created through Zenko will be chronologically ingested/synchronized.

### Problem Description

Currently, Zenko can only manage data that is stored locally, in the cloud, or in the Scality RING. In order for it to manage data stored in other *backends* (such as NFS, SOFS, and SMB), the data needs to be uploaded directly through the CloudServer API. However, this is an unnecessarily complex and time consuming procedure, which consists in duplicating data rather than allowing Zenko to manage existing one.

### Use-cases Description

Leverage Zenko features (multi-cloud replication, metadata search, lifecycle policies, and such) on data stored in any of the following backends transparently:

- NFS

- SMB

- CephFS

- FlexVolume

- [and more](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistent-volumes)

### Technical Details

In short, a manager called *Atmosphere* will watch MongoDB for configuration updates. If a new storage location of type "filesystem" is added, then *Atmoshpere* will provision a *Cosmos* pod and a persistent volume of the desired type (see above). This pod will then serve that data underlying the PersistentVolume over HTTP to Cloudserver, similarly to how S3-Data does it. Additionally, this pod will syncrhonize/ingest pre-existing data and data not created through Zenko chronologically.

##### Atmosphere

The role of Atmosphere is to create *Kubernetes Persistent Volumes* (PVs) and apply a new deployment of pods that will have the appropriate claim (PVC) on the newly provisioned PV. In the future we would want the pod deployments to be taken over by the Zenko Operator and let Atmoshphere just handle the PV dynamic provisioning.

##### Cosmos

The Cosmos pod consists of 2 containers: 

- FS-DATA: A RESTful HTTP server that can stream data from its underlying filesystem to whomever requests for it (i.e. Cloudserver).

- RClone-Daemon: A daemon will run an rclone process in a cronjob fashion to sync data on the filesystem with MongoDB. It would do so by making requests to cloudserver, and each object put will be have a special header to signify it is a *filesystem-backed* file. This header will include the MD5 and size of a file.

These two pods will share a mount which corresponds to a PersistentVolume of the desired backend type (NFS, SMB, and so on). 

**Note:** Optimizations can be made in *v2* to allow the rclone process to run sooner based on things like geosync logs for hinting and listing/attribute cache to speed up the listing.

### Alternatives

Currently, there are **NO** alternatives for the "Cosmos" framework. However, there are some alternatives for supporting "NFS" compatible storage systems as a backend and/or being able to ingest data from them. Here is a list of them:

- **Option 1: Mount CloudServer Pods**
  With SYS_ADMIN or similar capabilities applied to each of the cloudserver containers, they would be able to directly “mount” to an NFS which would greatly simplify configuration. However, this opens the door to potentially severe security risk because cloudserver is an externally exposed service (in fact, the only one).

  **Pros:**

  - Easy to implement

  **Cons:**

  - Requires special privileges

  - If the NFS server fails, Cloudserver will be unresponsive

- **Option 2: A userspace NFS client**
  With a userland NFS client, such as [node-nfsc](https://github.com/scality/node-nfsc), the Cloudserver pods would be able to safely mount on the the NFS exports.

  **Pros:**

  - It doesn't require privileges

  - If the NFS server fails, Cloudserver will be still be responsive

  **Cons:**

  - Currently, there is no support NFSv4 and some NFS server may not be compatible.

  - Not as stable as the kernel NFS client

  - We would have to implement correct connection pooling and parallelism to have credible performance.

- **Option 3: Live update daemon**

  - Listing with prefix: compare both mtimes for the directory on the filesystem and in Mongodb for that prefix (there should be a placeholder). if up to date, cloudserver proceeds with the listing from mongodb. if not uptodate, cloudserver finds the changes and update mongodb's content for that particular prefix, then proceeds with the listing (still from mongodb).
  - HEAD or GET on an object: stat(3) the targeted file, update the object's representation in mongodb if relevant, then cloudserver proceeds with serving the file
  - Listing without prefix: proceed with the recursive listing of the filesystem (modulo the maxKeys parameter) and update the objects/"directory placeholders" in mongodb if relevant (I just made it up, I'm not certain about this part).

  **Pros:**

  - No need for rclone.

  **Cons:**

  - There is a complication when file system clients perform partial updates of the files.

  - Only happens with specific file system workloads, e.g. random writes. And this constraint can be easily understood (or stated as not optimized).

  - The first HEAD/GET on the partially modified file the mongodb image gets updated.

  - Races between readdir and stat are more commonly understood, which can be ok as long as the former returns the truth about the file.

### Roadmap

- v1: Readonly capabilites. The goal is only to be able to ingest data from a NAS.

- v2: Support for deleting files through Zenko (i.e. lifecycle policies).

- v3: Write functionality. Full control of NAS-backed data through Zenko.

### Design Diagram

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
|        |         +---------->    Hubble Gateway   <-------------------+         |
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
