# UTAPI metrics protection

## Context

UTAPI v1 And v2 consumption metrics can be retrieved using root user access keys
or IAM users access keys.

IAM user can have restricted access to specific metrics using IAM policies
mechanism. However, root users have unlimited access to consumption metrics. For
instance, `accountA` root user can access `accountB` metrics.

Service provider customers are complaining about the current implementation. They
would like metrics to be protected at the account level so that they can expose
UTAPI to their customers.

In the meantime, service providers have used this behavior to implement their
billing system. So, if we change the current behavior, we need to provide service
providers another to bill their customers.

## Requirements

### Billing Story

**As a** Storage Administrator

**I want** to be able to list all storage accounts consumption metrics

**In order** to bill storage account owners

Acceptance criteria

* A single set of keys must allow retrieving all storage accounts consumption
  metrics
* Credentials must not allow to create, update or delete storage accounts, IAM,
  S3, or STS resources (e.g: buckets, objects, users, policies, roles…)

### Protection Story

**As a** Storage Account Owner(a.k.a root user),

**I want to** prevent my IAM Account consumption metrics to be accessed by
another Storage Account Owner

**In order to** keep account consumption metrics private

Acceptance criteria:

* Account A can't access Account B consumption metrics using root or non-root
  user credentials
