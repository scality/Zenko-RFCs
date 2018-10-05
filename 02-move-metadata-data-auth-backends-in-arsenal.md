# ZIP#2 (Zenko Improvement Proposal #2) Move Metadata/Data/Auth Backends To Arsenal

## Overview

This document is a proposal for an architecture change on the metadata, data and auth backends of the Cloudserver component.

### Problem Description

Currently Cloudserver requires 3 types of backends:
* Metadata backends: Backends required to store the buckets metadata (e.g. MongoDB).
* Data backends: Backends required to store the objects data (e.g. local files or public clouds).
* Auth backends: Backends required to store the credentials such as access and secret keys (e.g. in local files).

Note data backends are separated in 2 sub-classes:
* By key backends: Very simple backends which does not preserve namespace (e.g. key/value stores).
* By path backends: More complex backends which preserve the namespace (the path of objects), e.g. public clouds, NAS filers, etc.

Currently metadata backends have already been moved to Arsenal, the common library between all Zenko nodejs based 
components such as Cloudserver and Backbeat.

The goal is now to move data and auth backends.

### Motivation and Use Cases

#### Separation of Concerns

Cloudserver should only care about the S3 protocol API parsing.

Some remarks:
* The separation of concerns permits a simplification in the Cloudserver component.
* Cloudserver shouldn’t have to manage and make decisions about where the data is actually stored.
* The external storage quirks should be hidden from the Cloudserver perspective. 
* Data encryption (KMS) is a secondary concern which may be delegated to a lower level component.

#### Direct Backbeat Access to Data

Currently, when Backbeat needs to perform some I/O on data backends it goes through
Cloudserver in specific routes called "Backbeat routes". 

Instead, Backbeat could directly use the data backends code in Arsenal to perform I/O.

The problems currently are:
* Backbeat potentially overloads Cloudserver.
* Architecturally it is difficult to explain why Backbeat contacts Cloudserver for some operations.

Both Cloudserver and Backbeat should benefit from these changes.

#### A Cleaner and Simpler Abstraction for Writing New Backends

We can assume it will be much simpler for people to contribute new backends in Arsenal, because:

* We will provide a much cleaner documentation for writing a metadata, data or auth backends
* Contributors could test their backends unitarily in Arsenal (e.g. without having to also test the S3 protocol layer front-end).

### Proposed Changes

It is not really a refactor, more moving code, it will nevertheless requires to change the following :

* Change the path in 'require' clauses in the code which is moved.
* Change the path in 'require' clauses in the code using the code which is moved.
* Config parametes shall be passed as params structures.

Other minor benefits:

* The number of exports from Arsenal will be reduced as we will only exports wrappers.
* Some backends directly import the global config, so it will force us to transform those into params.

#### Components

* Cloudserver: https://github.com/scality/cloudserver
* Arsenal: https://github.com/scality/arsenal
* Backbeat: https://github.com/scality/backbeat
