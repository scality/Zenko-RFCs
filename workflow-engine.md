
# ZIP#1 (Zenko Improvement Proposal #1) - Workflow Engine

## Overview

The workflow engine is about allowing executing "functions" to process data as they go
through Zenko.

### Problem Description

We want to be able to call "functions" when users or system processes do Zenko operations
on the platorm such as:

* New data events: such as PUT or DELETE requests on the system
* Searches: cron batches can be scheduled to perform searches (basic listings or Zenko searches) triggering functions

It would be possible to trigger events on read requests such as GET or HEAD but we do not
address this case yet.

### Anatomy of a Workflow Execution

A workflow is supposed to represent the flow of a context from an event source traversing multiple
functions possibly altering the context or creating new content in the object storage platform.

```
+---------+      +----------+   +----------+    +----------+
|Context  +----->+Function 1+-->+Function 2+--->+Terminator|
|generator|      |          |   |          |    |          |
+---------+      +----------+   +----------+    +----------+
```

The context which is transmitted along the different functions is typically:

* Bucket name
* Object name
* Tags: every object in the object storage system has zero or more tags.
* User metadata: e.g. x-amz-meta-* headers of an object
* System metadata: Such as owners, users, timestamps, sizes, user-defined attributes.
* Transport meta-information: Such as IP addresses, client TLS certificates, etc.
* System metrics: Such as the current number of TCP connections, etc which can be used to implement a
quota policy.
* Credentials: to eventually read object data or create new content (e.g. new object version).

Note that the advantage of tags is the context of an S3 compatible object storage platform is that updating them does not generate a new version of the object in a versioned bucket, contrarily to user metadata.


### General User Experience

The user shall be able to manipulate workflows in a specific UI (e.g. integrated in Orbit).

The typical actions are:

* Create a new workflow with a name
* List workflows
* Edit a workflow
* Start/stop a workflow: mark the specified workflow as active
* Pause/Resume a workflow: pause or resume an active workflow

The workflow edition allows the user to create boxes representing steps of the workflow. Boxes
have zero or more input ports and zero or more output ports. An input port is supposed to received
links from other boxes output ports. An output port is supposed to depart links towards other boxes input ports.

```
        +--------------------+
        |                    |
      +-+                    +-+
input | |      Function      | | output
port  +-+                    +-+ port
        |                    |
        +--------------------+
```

Some boxes have only output ports because their role is to generate contexts. Some other boxes
have only input ports because they are "terminators".

Here is a non-exhaustive list of boxes:

* A "new data" box with 1 output port: will generate a new context every time an object version is created or deleted.
* A "search" box with 1 output port: will generate a new context periodically by listing or searching in the object storage metadata sub-system.
* A "function" box with 1 input port and 1 output port: which is able to call external functions, such as Fission, Knative, Azure functions, AWS Lambda functions, Google Cloud Functions. Those external function are provided a context and are able to return a new context (typically a set of tags than can be merged or replace existing tags).
* A "tag" box with 1 input port and 1 output port which is able to simply add a tag or execute a custom script  to modify the context.
* A "decision" box with 1 input port and 2 output ports: This box is able to execute a basic condition on the tags set (or execute a more complex script) and orient the flow towards output1 or output2.
* An "update" box with 1 input port: this box is supposed to replace the context on the storage.
* A "terminator" box with 1 input port: this box is supposed to simply terminate the execution without any action.

Note that all boxes are supposed to alter the context in "memory", the context is to be updated in the final "update" box.

Functions can eventually generate new content (typically new object versions).

### Synchronization between Boxes

Any number of arrows can originate from an output port, and any number of arrows can target
an input port.

```
                                +-----------------+
                                |                 |
                              +-+     Step 2      +-+
                        ----->| |                 | |-
                       |      +-+                 +-+ |
+----------------+    -         |                 |    -       +-----------------+
|                |   |          +-----------------+     |      |                 |
|                +-+-                                    --->+-+     Step 4      |
|     Step 1     | |                                         | | Synchronization |
|                +-+-                                    --->+-+ Point           |
|                |   |          +-----------------+     |      |                 |
+----------------+    -         |                 |    -       +-----------------+
                       |      +-+                 +-+ |
                        ----->| |     Step 3      | |-
                              +-+                 +-+
                                |                 |
                                +-----------------+
```

In the example above, step 1 generate 2 contexts which causes step 2 and step 3 to be
executed in parallel. Step 4 is the synchronization point: execution is blocked until
the execution of step 2 and step 3 is finished. The resulting tags of the 2 operations
are merged before feeding step 4.

### Synchronous and Asynchronous Functions

Workflow functions can be executed synchronously or asynchronously.

Synchronous functions can be applied if the execution time does not
hinder the global service, e.g. calling a function that checks for object content type
can be executed reasonably fast for being called directly though the HTTP interface.

Some more advanced functions, such as complex AI processing,
or semi-structured data analysis (e.g. using Spark) cannot be executed
synchronously because it would be too slow for an HTTP request. Instead they
are executed through a queue trigger (e.g. Kafka).


### Synchronous (fast) and Asynchronous Workflows

Similarly entire workflows can be executed synchronously or asynchronously.

Obviously a workflow containing an asynchronous function cannot be executed synchronously.

Although generally workflows are executed after an operation is inserted in the operation journal of the metadata database, so they are in fact asynchronous. Every event generated between 2 boxes is persisted in a persistent queue (Kafka).

Synchronous workflows can be triggered directly from the HTTP frontend of cloudserver, every event generated between 2 boxes will be persisted in memory (for fast non invasive execution).


### Other Restrictions in Workflows

There can be only one source box per workflow. Because otherwise it would lead to impossible synchronization issues.

### Use Cases

#### Notification

A user may want to be notified when an object is created.

How: The user creates a workflow that hooks PUT events and execute a Fission or Knative function that
sends a notification.

#### Authorization

A user may want to call a workflow to check if a Zenko operation is authorized.

Examples of sub-use cases:

* Limited Access by IP Address: Allow unauthenticated access by specified IP addresses.
* Refuse PUTs by Time: For example, a financial service may forbid updating a system after 5pm.
* Refuse PUTs by File Size: This mode can enforce the 100-continue mechanism, e.g. to limit abuse of the
system, such as having to refuse a PUT after having ingesting 1TB of data.
* Refuse PUTs by Attribute: Read the PUT headers to enforce certain attributes.
* Bucket Object Prefix: Restricts users by prefixes.


#### AI Processing

Example: Auto-tagging.

A user wants that every time a picture is stored in the object storage, a function (e.g. Tensorflow based, or using Cloud cognition services like Azure Cognitive API or Amazon Rekognition) will analyze the content of the image and deduce a tag set describing the image.

Example: Auto-description of Videos

Same example as above but for describing what happen on the video.

Etc.

#### Cron-Based “Search” Workflows

These workflows are also asynchronous and triggered by a crontab and typically generate their initial queues by
searching objects in buckets, using basic listing or more complex Zenko Metadata search.


### Proposed Changes

#### Architecture


```
              +-----+ Graphical workflow edition
              |Orbit| Workflow Statistics
           +--------+ Workflow Dashboard
           |
           |
           |
    +------v---+        +---------------+  workflow +-------+
    |Management| RESTful|Workflow Engine+---------->+ Store |
    |Agent     +------->+Operator       |  storage  +---+---+
    +---+------+        +-+-----------+-+               
        |                 |           |             +-------+    
pause/  |                 |           |             |MongoDB|
resume  v                 v           |             +-------+
     +--+--+            +-+-------+   |restart          |Operati
     |Redis|            |k8s      |   |                 |log    
     +--+--+            |configmap|   +------------+    |
        |               +-+-------+   |            |    |
        |                 |           |            v    v
        |                 |           |          +-+----+--------+
        |                 v           v          |Queue populator|
        |              +--+-----------+-+        |(log reader)   |
        +------------->+ Workflow       |        +-+-------------+
                       | Engine         |          |
                       | Backbeat Ext   |          v
                       +----+           |        +------------+
                       |cron|           +<-------+Event queue |
                       +----+-----------+        +------------+
                         1|           |1
                          |           |
                         *v           v*
               +----------------+   +--------------+
               |Function        |   |Function      |
               |Engine (premise)|   |Engine (cloud)|
               +----------------+   +--------------+

```

| Component | Description |
| ------------- | ------------- |
| Graphical Workflow Edition  | One can define multiple workflows using this Orbit UI component. When saving workflows, it generates a JSON file describing the set of workflows the Workflow Manager can read.  |
| Workflow Statistics | The Workflow Manager maintains and reports statistics about objects traversing each workflow the steps of each workflow. |
| Workflow Dashboard | The system administrator can view Orbit UI statistics  graphically. |
| Management Agent | Agent that communicates with Orbit. |
| Workflow Engine Operator | RESTful agent that LIST, GET and PUT workflows in persistent storage. Can also start/stop workflows |
| Store | Used for workflow configuration storage |
| MongoDB| Used to trigger "new data" events. |
| Redis | Used for storing pause/resume information (different from start/stop) |
| K8s configmap | The workflow engine operator generates a configmap containing active workflows |
| Workflow Engine Backbeat extension | This Backbeat component executes the JSON-defined workflows. |
| Queue populator (log reader) | This container filters events from the log reader and send them in the event queue |
| Cron | The internal event generator for list/search |
| Event queue | The event persistent storage (Kafka) |
| Function Engine (premise) | 1 or many function engines running on premises (e.g. Fission or Knative) |
| Function Engine (cloud) | 1 or many function engines running on the cloud (e.g. Azure functions or AWS Lambda) |


#### Workflow Engine Operator

The workflow engine operator plays a central role in the workflow engine configuration.

It has special k8s privileges to create configmaps and restart the workflow engine containers.

The operator is responsible for transforming the workflow configs from the persistent storage
to the k8s configmaps in a way that is friendly to modifications and upgrades.

When a user creates a new workflow in the UI, the new workflow will be POSTed to the workflow engine operator
and stored in the Store. Also a new list of active workflow will be generated in the workflow configmap (which is an array of workflow files). The workflow will be applied to the new traffic only.

When a user modifieds a new workflow in the UI, the modified workflow will be PUT to the workfow engine operator and persisted in the Store. The old workflow(s) will be active until all old events are purged from the event queue. The new workflow will be active for new events only (for this a new consumer group will be created). A specific stopper message will be inserted in the event queue to force stop the processing of old events.

When a user stops (different from pause) an active workflow, a specific stopper message will be inserted in the event queue to force stop the processing after all old events are processed.

When a user starts (different from resume) a stopped workflow. A stopper is inserted for the old workflow. The processing will start for new events only, a specific consumer group will be created.

Pause and Resume are not managed by the workflow engine operator but directly through Redis. Pause does not stop the workflow but simply pauses (presumably temporarily) the workflow execution.


#### Other High-Level Requirements


* Multi-Language Support:
A workflow function could be written in any language (NodeJS, Python, Go,
C, C++, etc.).
* Single Framework Language:
The workflow manager is a Backbeat task written in NodeJS.


#### Examples of "external" on-premise functions

* file-type: Guess the MIME file type of an object by reading a minimal preamble of data and create a new tag called "mime".
* exif-extract: Extract EXIF information from an image and generate tags.
* auto-tag: call a Tensorflow model to auto-tag a picture.

#### Example of "external" cloud functions

* auto-tag: call Azure function + cognitive service to describe a picture (In this case the object content is POSTed to Azure).


#### Possible Implementation

For the function framework we are going to reuse existing
serverless frameworkw:
* [Kubeless](https://forum.zenko.io/t/deploying-cloudserver-along-with-kubeless-to-do-event-based-processing/478)
* [Fission](https://fission.io/)
* [Knative](https://github.com/knative/).

We want to be independent from the engine.

This type of serverless framework generally come with a set of
cli-tools to create/destroy functions. And also supports some API (see [Fission doc](https://docs.fission.io/) or 
[Kubeless through Kubernetes
API](https://kubeless.io/docs/advanced-function-deployment/)) to
define functions programmatically.

Generally, the functions can be implemented in the following languages:

* Python
* Node.js
* Ruby
* PHP
* Golang
* .NET
* Ballerina
* custom runtimes

Which matches our requirements.

Generally serverless frameworks support multiple ways of interacting with functions which are of interest for us:

* CLI-based: Useful for debugging and troubleshooting
* HTTP-based: Useful for synchronous function calls.
* Kafka Topic based: Useful for asynchronous function calls (e.g. fission returns the error code
in an error topic (see [here](https://docs.fission.io/usage/kafka-trigger-tutorial/)).

They also generally supports [auto-scaling](https://kubeless.io/docs/autoscaling/).

They also can generally report metrics through Prometheus.

#### Software stack


```
+---------------------------------------------------+
|                       Orbit                       |
+----------------------------------+----------------+
|           Existing panels        |  Workflow UI   |
+---------------------------------------------------+
|Replication|Lifecycle MD|Ingestion|Execution Engine|
+---------------------------------------------------+
|                       Backbeat                    |
+---------------------------------------------------+
|                        Zenko                      |
+---------------------------------------------------+
```


#### Metrics

TBD
