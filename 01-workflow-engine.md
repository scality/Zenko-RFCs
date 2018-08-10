
# ZIP#1 (Zenko Improvement Proposal #1) - Workflow Engine

## Overview

### Problem Description

### Use Cases

Which use cases does this address? What impact does this change have? On which
actors?

Be clear about the model actor in each use case: developer, user, deployer, etc.

#### Notification SNS (Simple Notification Service) like-API
We may support the PutBucketNotification API and present the option of
triggering a workflow in case of an event.

#### Authorization API
We may support an API similar to SNS that can call a workflow before actually
sending any data to CloudServer (e.g. avoiding sending unauthorized 1 TB files).

##### Limited Access by IP Address
Allow unauthenticated access by specified IP addresses.
##### Refuse PUTs by Time
For example, a financial service may forbid updating a system after 5pm.
##### Refuse PUTs by File Size
This mode can enforce the 100-continue mechanism.
##### Refuse PUTs by Attribute
Read the PUT headers to enforce certain attributes.
##### Bucket Object Prefix
Restricts users by prefixes.

#### Cron-Based “Search” Workflows
These workflows are triggered by a crontab and typically generate their initial queues by searching objects in locations; aws:foo/\*.jpg , for example.


### Proposed Changes

[Here is how we propose to solve the problem.]

[Scope of effort.]

#### Architecture Diagram

```

+-----------------+
|                 |
|                 |
|    Orbit UI     | Graphical Workflow Edition
|                 |
|                 | Statistics, Dashboard
+---------------+-+
                |
                |
                |              +-------------------------+
                +              |                         |
          JSON workflow        |                         |
                +              |   Workflow Engine       |
                |              |                         |
             +--v--------+     |                         |
             |Management +----->                         |
             |Agent      |     +-^----------------^------+
             +-----------+       |                |
                                 |                |
                                 |                |
                                 |                |
+----------------+            +--+----------  +---+-------+
|                |   DATA     |            |  |           |
|  Cloud Server  |            |            |  |           |
|                <------------+ plugin1    |  | plugin2   |
|                |            |            |  |           |
+----------------+            +------------+  +-----------+


```

#### Components

| Component | Description |
| ------------- | ------------- |
| Graphical Workflow Editing  | One can define multiple workflows using this Orbit UI component. When saving workflows, it generates a JSON file describing the set of workflows the Workflow Manager can read. **[This is the first mention of the "Workflow Manager" entity. Define it sooner.]** |
| Workflow Engine  | This Backbeat component executes the JSON-defined workflows. |
| Plugins | Plugins register to the Workflow Engine and perform single tasks in the workflow, such as compression or Lambda triggering. |
| Workflow Statistics | The Workflow Manager maintains and reports statistics about objects traversing each workflow the steps of each workflow. |
| Workflow Dashboard | The system administrator can view Orbit UI statistics  graphically. |
| Management Agent | Agent that communicates with Orbit. |
| CloudServer | Needed when a plugin requires reading or writing data. |

#### Requirements


##### Workflow
Orbit will send the new JSON file to the Workflow Manager, which will be
responsible for executing it.

At every step, the Workflow Manager will check for matches in object names or
other properties to trigger an action, which can be a built-in function or an
external module (e.g. encryption, compression, data movement, etc.)

##### Non-Disruption of Existing Traffic
Currently, modifying a workflow in place is too disruptive for traffic, so a
new workflow is always created. We propose applying new workflows to a
portion (5%, for example) of traffic.

##### Multiple Parallel Workflows
The Workflow Manager can simultaneously execute many workflows.

#####  Multi-Step Workflows
A workflow can consist of many steps, with one or many inputs.

##### Workflow Source
At the beginning of every workflow there is a metadata source (events generated
by actual traffic or by a cloud trace, such as Admin API or other cloud-specific
event logs like AWS Lambda or Azure functions). A workflow’s input can also
result from metadata searches executing at specified times (for example, searchObject (“bar”, “\*.jpg” )[**is this search syntax? If so, set as
``code``. If not, get back to me.**] bound to a crontab).

##### Workflow Termination
At the end of every workflow is a terminator that stops the workflow. The
terminator can, for example, be a copy to a cloud (putObjectToLocation()),
or the triggering of a Lambda function (executeLambda()).

##### Impedance Tolerance Between Plugins
Because all plugins can’t process information at the same time, as well as for
resilience, every time a workflow step must execute a specific plugin, input
*and* output queuing (for example, in a specific topic) is performed, with the
output being the input of the next plugin, and so on.

##### Multi-Language Support
Because a workflow module could be written in any language (NodeJS, Python, Go,
C, C++, etc.) we will favor a REST API for interaction between modules.

##### Single Framework Language
The workflow manager is a Backbeat task written in NodeJS.

##### Workflow Types
Workflow Management may occur at different CloudServer process steps:

- Authorization step
- Notification step (event)
- Cron-based (e.g., search)

##### Nature of Queues
Workflow queues only contain metadata: the object name, the object ID, and the “location” to find the object (if possible in a transient source).

##### Types of Plugin
Plugins can be:
- Metadata-only for both input and output
- Data-plus-metadata for both input and output
- A hybrid of metadata input and data-plus-metadata output
- A hybrid of data-plus-metadata input and metadata output

##### Data Manipulation in Plugins
Plugins that manipulate data read it from CloudServer, with location:objectID
specified in queues metadata. The plugin’s output (if any) is streamed to an
object in CloudServer and the new location:objectID is stored in the queue for
the next plugin in line.

#### Generic Workflow Functions
Here is a non-exhaustive list of workflow functions:

- **filterByIPAddress()**

  Filter an S3 operation by IP address

- **putObjectToLocation(location)**

  Put a given object to a destination location.

- **newObjectEvent(location)**

  Triggered when a new object is created on a given
  location. This plugin will have to be fed with events
  coming from a location.

- **encryptObject()**

  Encrypt an object

- **compressObject()**

  Compress an object

- **executeLambda()**

  Execute a code snippet (bucket notification, for example)

- **searchObjects(location, regexp)**

  Search all objects in a given location that match regexp.
  Can be used as a metadata source.

- **erasureCode(n, k)**

  Generate an *n* data and *k* coding fragments (as new
  objects), to be dispatched on different locations.

- **tagObject()**

  Tag the specific object with an attribute

Third-party plugin example:

- **encodeVideo(format)**

  Encode a video in the given format. Generates a new object.

#### Order of Execution

1. **Plugin1 registers** The first plugin registers itself
   and its tuning parameters to the workflow engine.
2. **Plugin2 registers** The second plugin registers itself
   and its tuning parameters to the workflow engine.
3. **Workflow engine registers plugins** The workflow
   engine registers the new plugins and their tuning parameters to the configuration hub (Management agent).
4. **Management agent transmits plugins** The Management
   agent transmits the new plugins and their tuning parameters to Orbit.
5. **Edit workflow** An account edits the workflow in Orbit
   with a GUI. Because the two plugins are registered, the
   account can drag and drop them and tune their parameters.
6. **Push JSON workflow** When the account clicks
   **Deploy**, the new workflow is sent to the Management Agent.
7. **Store config** The workflow is stored in the
   configuration.
8. **Read config** The workflow engine producer (WEP) reads
   the configuration.
9. **Execute workflow** The WEP executes the workflow.
10. **Plugin1 queue** If, for example, the first step of the
    workflow were to feed the plugin queue with events
    coming from a specific location (as required by the
    plugins), with plugins only needing to be launched at
    specific times (per a crontab), the WMP would manage
    only the output queue.
11. **Produce plugin1** The events for the first plugin (for
    example, objects’ metadata) are queued in the dedicated  
    plugin1 queue.
12. **Consume plugin1** The Workflow Engine Consumer (WEC)
    reads the events for the first plugin.
13. **Call plugin1** For each event in the queue, the WEC
    calls plugin1 with a REST interface. By default, only
    the objects’ metadata is sent along. If the plugin
    requires data, it can use the location:objectId and ask
    CloudServer.
14. **Output plugin1** For each input event, an output event
    is returned to the WEC.
15. **Plugin2 queue** The WEC (acting as a WEP) feeds the
    second plugin input queue.
16. **Produce plugin2** Metadata is queued for plugin2
    consumption.
17. **Consume plugin2** The WEC consumes the queue.
18. **Call plugin2** Plugin2 is called on a REST interface.
    If plugin2 is a terminator plugin, the workflow ends
    here.

### Alternatives

What are other ways we could implement this, and why are we
not using them?
