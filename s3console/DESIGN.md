# S3 Console

## Concepts

### Master Key

- are a unique set of keys
- allows to CRUD Accounts
- retrieve consumption metrics on UTAPI

### Root User Access Keys and Secret Key

- are a set of keys associate with a single Account
- allows performing all the actions on IAM and S3 entities within an Account.
- allows retrieving consumption metrics on UTAPI

### Storage Administrator

The storage Administrator is responsible for managing and monitoring the services
provided by Scality products.

### Storage Manager

The Storage Manager is responsible for creating and selling Accounts to Storage
Account Owners.

### Storage Account Owner

The Storage Account Owner is responsible for managing an securing a Storage Account.

## Context & Problems to solve

### Security Issue

The S3 console is a stateless component using a set of root AK/SK to manage IAM
entities. Each S3 console instance has its own set root AK/SK on each account.  

When an Account Owner is performing a s3:listAccessKeys on his account, he saws
all the root AK/SK created by the S3 console containers, and the keys he created
for his usages. Storage Account Owner doesn't want to see root AK/SK he has
not created for security reasons.

### Desynchronization Issue

The S3 console is a stateless component deployed on every stateless server.

After a few months of productions where accounts are created and deleted on the
S3 console, all S3 console contrainers doesn't have the same number of accounts.  

Scality support have to manually resynchronize the SQLite database located inside
the S3 console containers.

### Scaling Issue

The S3 console is a stateless component deployed on every stateless server. When
requests on the S3/IAM services increases, Storage Administrator are adding more 
stateless components to handle the workload.

The new S3 console containers are empty. They do not have any references to 
the accounts previously created.

Scality support has to manually resynchronize 
the SQLite database located inside the S3 console container.

## Design

### API

### UI

### Upgrade/Downgrade

### Security

### Tests

### Documentation

## Open question