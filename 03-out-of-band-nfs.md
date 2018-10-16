<h1 align="center">
ZIP#3 (Zenko Improvement Proposal)<br>The "Cosmos" Storage Backend
</h1>

### Overview

This document is a proposal for an extensible framework (a.k.a Cosmos) that will allow Zenko to manage data stored on various kinds of *backends* such as filesystems, block storage devices, and any other storage platform to which there is an available Kubernetes PersistentVolume plugin. Pre-existing data on these storage systems and data not created through Zenko will be chronologically ingested/synchronized.

### Problem Description

Currently, Zenko can only manage data that is stored locally, in the cloud, or in the Scality RING. In order for it to manage data stored in other *backends* (such as NFS, SOFS, and SMB), the data needs to be uploaded directly through the CloudServer API. However, this is an unnecessarily complex and time consuming procedure, which consists in duplicating data rather than allowing Zenko to manage existing one.

### Use-cases Description

Leverage Zenko features (multi-cloud replication, metadata search, lifecycle policies, and such) on any of the following backends transparently:

- NFS

- SMB

- CephFS

- FlexVolume

- [and more](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistent-volumes)

### Technical Details

In short, a manager called Atmosphere will *watch* MongoDB for configuration updates. If a new storage location of type *"filesystem"* is added, then Atmoshpere will provision a "Cosmos" pod and a persistent volume of the desired type (see above). This pod will then serve that data underlying the PersistentVolume over HTTP to Cloudserver, similarly to how S3-Data does it. Additionally, this pod will syncrhonize/ingest pre-existing data and data not created through Zenko chronologically.

##### Atmosphere

TODO

##### Hubble

Hubble is an API Gateway. It is the only known endpoint by Cloudserver that corresponds to the "Cosmos" backends. Whenever Atmosphere provisions a PV and a Cosmos pod, it will update Hubble so that it routes Cloudserver requests to the appropiate Cosmos pod based on a storage location header (or the bucket name).  

##### Cosmos

The Cosmos pod consists of 2 containers: 

- FS-DATA: A RESTful HTTP server that can stream data from its underlying filesystem to whomever requests for it (i.e. Cloudserver).

- RClone-Daemon: TODO

These two pods will share a mount which corresponds to a PersistentVolume of the desired backend type (NFS, SMB, and so on). 

**Note:** Optimizations can be made in *v2* to allow the rclone process to run sooner based on things like geosync logs for hinting and listing/attribute cache to speed up the listing.

### Alternatives

Currently, there are **NO** alternatives for the "Cosmos" framework. However, there are some alternatives for supporting "NFS" compatible storage systems as a backend and/or being able to ingest data from them. Here is a list of them:

- **Option 1: Mount CloudServer Pods**
  With SYS_ADMIN or similar capabilities applied to each of the cloudserver containers, they would be able to directly “mount” to an NFS which would greatly simplify configuration. However, this opens the door to potentially severe security risk because cloudserver is an externally exposed service (in fact, the only one).

  **Pros:**

  -    Easy to implement

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

### Roadmap



### Design Diagram

```sh
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
























