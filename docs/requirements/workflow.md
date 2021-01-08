# Workflow requirements

## Concepts Definition

## Workflow Overview Tab Requirements

### Replication

#### Shortlisted

Volume:

- ZENKO-3014 Ability to monitor the number of object in the source
- ZENKO-3015 Ability to monitor the number of object in the destination
- ZENKO-3016 Ability to monitor the number of object with a being/ongoing status
- ZENKO-3017 Ability to monitor the number of object with a failed replication status
- TBD-0001 Ability to monitor the number of object with a pending replication status
- TBD-0002 Ability to monitor the number of object replicated

Velocity:

- TBD-0003 Ability to monitor the number of op/s processed
- TBD-0004 Ability to monitor the currently bandwidth used

Operations:

- TBD-0005 Ability to monitor the number of operation failed per second
- TBD-0006 Ability to monitor the number of operation being/ongoing per second
- TBD-0007 Ability to monitor the number of operation success per second

SLA:

- S3C-3618 Ability to monitor the actual replication RPO

#### Studied

Will be used included Storage Administrator pages:

- S3C-3635 Ability to indicate the number of objects per partition in Kafka
- S3C-3636 Ability to monitor the state of the queue processor
- TBD-0008 Ability to monitor the state of the queue populator
- S3C-3631 Ability to indicate the bandwidth used by CRR services vs S3 services
- S3C-3632 Ability to indicate the number operation per second used by CRR services vs S3 services
- S3C-3633 Ability to indicate the latency per operation used by CRR service vs S3 frontend service

#### To studied

- S3C-3634 Ability to indicate how much time is spent accessing the local data
  vs time spent sending the data for CRR