# Zenko Specifications

This repository contains details about the designs of Zenko features.
[Zenko](https://www.zenko.io) is software that provides multi-cloud
data management capabilities across private and public clouds.

The Zenko project is driven by a rising chorus of customer
requirements to integrate data they manage on-premises storage
solutions with services available in public clouds. Most corporations
already utilize services from many cloud vendors mainly using the
Amazon Web Services (AWS) public cloud but also Microsoft Azure and
Google Cloud Services (GCS).

There is a clear need for enterprises to manage workflows across
clouds to use cloud-based services and to be able to select the
optimal service at any time for any data. Zenko is designed to give
"freedom and control" for users to select the best service for their
data and avoid the lock-in they have experienced with IT vendors.

## Existing specs

### Asynchronous mechanisms

| Spec     |     Description | Link     |
|----------|-----------------|:-------------:|
| CRR to AWS S3 | | [Link](https://github.com/scality/backbeat/blob/development/8.1/docs/crr-to-aws-s3.md)|
| Data mover | | [Link](https://github.com/scality/backbeat/blob/development/8.1/docs/data-mover.md)|
| Backbeat Halthchecks | | [Link](https://github.com/scality/backbeat/blob/development/8.1/docs/healthcheck.md) |
| Lifecycle | | [Link](https://github.com/scality/backbeat/blob/development/8.1/docs/lifecycle.md) |
| Metrics | | [Link](https://github.com/scality/backbeat/blob/development/8.1/docs/metrics.md) |
| Out-of-band updates from RING | | [Link](https://github.com/scality/backbeat/blob/development/8.1/docs/oob-s3-ring.md) |
| Out-of-band updates from FS | | [Link](https://github.com/scality/Zenko/blob/improvement/cosmos-design/cosmos/design.md) |
| Pause/Resume | | [Link](https://github.com/scality/backbeat/blob/development/8.1/docs/pause-resume.md) |
| Site level CRR | | [Link](https://github.com/scality/backbeat/blob/development/8.1/docs/site-level-crr.md)|
| Transient CRR source | | [Link](https://github.com/scality/backbeat/blob/development/8.1/docs/transient-crr-source.md) |

### Metadata Backends

| Spec     |     Description | Link     |
|----------|-----------------|:-------------:|
| Memory backend | | [Link](https://github.com/scality/Arsenal/blob/development/8.1/lib/storage/metadata/in_memory/bucket_mem.js.Design.md) |
| MongoDB backend | | [Link](https://github.com/scality/Arsenal/blob/development/8.1/lib/storage/metadata/mongoclient/Mongoclient.md)|

## Structure of this repository

This repository contains specifications for improvements to Zenko.
They follow a template:

- Clear title and lowercase-dashed name
- An overview that contains a description of the problem that we
  want to solve
- One or more use cases, with a clear mention of the actors in each
  use case
- Then a detailed description of the technical implementation
- A review of alternative approaches to solve the
  problem. This will show that the proposal has been analyzed at depth
  (think of this as bibliographic reference)

## How to add a spec

Anybody can add a spec: clone the repository,
create a copy of the file TEMPLATE.md and submit a pull request
with your proposal. The PR will be reviewed by Zenko team.

