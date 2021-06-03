# S3 Lifecycle Expiration

This document describes S3 Lifecycle expiration requirements for S3 connector.

## Context

Scality customers wants to automate the deletion of objects and object versions. 
They want to have the ability to configure the expiration using a Graphical User 
Interface(GUI) or the standardized AWS S3 API `s3:PutBucketLifecycleConfiguration`,
`s3:DeleteBucketLifecycle`, and `s3:GetBucketLifecycleConfiguration`.

## Roles

Storage Administrator and the Storage Account Owner roles are impacted by the lifecycle
expiration feature.

### Storage Account Owner Role

Storage account owners are responsible for configuring lifecycle expiration for an
S3 client application.

### Storage Administrator Role

Storage Administrators are responsible for operating the Scality platform. They
monitor the status and performance of the components that deliver the lifecycle
expiration service.

## User Stories

### Storage Administrator Story

**As a** Storage Administrator

**I want to** a monitor the lifecycle expiration components

**So that** I can identify configuration, performance, or network issues

### Storage Account Owner Story

**As a** Storage Account Owner

**I want to**  delete objects or previous version of an object after a given amount of
days using a lifecycle expiration configuration

**In order to** store cost-effectively my objects

## Acceptance Criteria

### Deployment and Configuration

The expiration system must:

- work on 1-site, 2-site and 3-site stretch deployments
- have the ability to scale horizontally

### Performances

The expiration system must be able to handle:

- 20 000 buckets.
- 80 million buckets with few objects.
- 10 billion objects within a bucket.

### API

#### Lifecycle Expiration Configuration

Lifecycle expiration configuration must:

- be configurable from the endpoint that is used to create, read, update, and
  delete (CRUD) objects and buckets.
- expire object and object versions.
- allow filtering objects or object versions by key name prefix, object tags, or
  a combination of both.
- be able to delete incomplete multipart uploads
- respect object lock configuration
- work on objects replicated from another RING
- work on objects replicated with an object lock configuration from another RING

#### Monitoring API

Lifecycle expiration components must be exposed monitoring metrics through HTTP
endpoint following Prometheus standards.

### Identity and Access Management

IAM policies must support deny or allow `s3:PutBucketLifecycleConfiguration`,
`s3:DeleteBucketLifecycle`, and `s3:GetBucketLifecycleConfiguration` actions.

### UI

#### Lifecycle Expiration Configuration UI

Lifecycle expiration configuration must be configurable through the S3 browser.

#### Monitoring UI

Lifecycle expiration components must have a Grafana dashboard to troubleshoot
performance, configuration, and/or network issues.

### Sizing

The RING sizing tool must be updated to take into consideration the lifecycle
expirations components requirements(RAM, CPU,and storage consumption).
