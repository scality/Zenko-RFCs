
# S3 Select

This document describes S3 Select requirements.

## Context

Our customers are requesting AWS S3 select API.

TODO: add use cases

## User Stories

### Storage Administrator Story

**As a** Storage Administrator

**I want to** a monitor S3 select components

**So that** I can identify configuration, performance, or network issues

### Storage Account Owner Story

**As a** Storage Account Owner

**I want to** select a subset of my object using S3 Select API

**In order to** reduce the network bandwidth consumption

## Architecture

S3 select components must be:

- Scalable horizontally
- Supported on 1-site, 2-site and 3-site stretch deployment

## Upgrade

S3 Select feature must be available during rolling upgrade and not interrupt
running queries.

## Logs

Audit logs must contains who perform which s3 select action on which object.

## Documentation

The documentation must contains:

- How to enable the feature
- How to monitor the S3 select components
- How to scale-up the S3 select components

## Metrics

### Monitoring

S3 select components must expose the following monitoring metrics:

- S3 Select request count
- S3 select request execution time
- S3 select request status
- Amount of record scanned
- Amount of record returned
- Amount of data scanned in bytes
- Amount of data returned in bytes

S3 select components monitoring metrics must be filtrable by:

- Host name
- S3 select component instance name (e.g: container name)

S3 select components metrics must be exposed through HTTP endpoint following
Prometheus standards.

### Billing

Utilization API(UTAPI) must expose the following S3 select consumption metrics:

- Select request count
- Select scanned bytes
- Select returned bytes

The consumption metrics must be filtrable by:

- Account
- S3 Bucket
- IAM User

## GUI

Grafana must contain S3 select components monitoring dashboard.

## Sizing

Sizing tool must be updated to take in consideration S3 select components
requirements. (e.g: 2 CPU threads + 10GB RAM + 50GB SSD per stateless server)

## Performances expected

none.

## ISV compatibility

none.

## S3 Select API

This section describe the Structured Query Language (SQL) elements that
must be supported.

### Object format compatibility

| Format                        | Supported |
| :---------------------------- | --------: |
| CSV                           |       yes |
| JSON                          |       yes |
| Apache Parquet                |       yes |
| GZIP(CSV or JSON)             |       yes |
| BZIP2(CSV or JSON)            |       yes |
| server-side encrypted(SSE-S3) |       yes |

### SELECT command

| Command      | Supported |
| :----------- | --------: |
| SELECT list  |       yes |
| FROM clause  |       yes |
| WHERE clause |       yes |
| LIMIT clause |       yes |

### FROM clause

| FROM clause                | Supported |
| :------------------------- | --------: |
| By name (in an object)     |       yes |
| By index (in an array)     |       yes |
| By wildcard (in an object) |       yes |
| By wildcard (in an array)  |       yes |

### Scalar expressions

| Scalar expression                                          | Supported |
| :--------------------------------------------------------- | --------: |
| literal                                                    |       yes |
| column reference                                           |       yes |
| unary_op *expression*                                      |       yes |
| *expression* binary_op *expression*                        |       yes |
| func_name                                                  |       yes |
| *expression* [ NOT ] BETWEEN *expression* AND *expression* |       yes |
| *expression* LIKE *expression* [ ESCAPE *expression* ]     |       yes |

- unary_op = SQL unary operator
- binary_op = SQL binary operator
- func_name = name of a scalar function to invoke

### Supported Data Types

| Data Type | Supported |
| :-------- | --------: |
| bool      |       yes |
| integer   |       yes |
| string    |       yes |
| float     |       yes |
| decimal   |       yes |
| timestamp |       yes |

### Operators

| Operator                               | Supported |
| :------------------------------------- | --------: |
| AND                                    |       yes |
| NOT                                    |       yes |
| OR                                     |       yes |
| <                                      |       yes |
| >                                      |       yes |
| <=                                     |       yes |
| >=                                     |       yes |
| =                                      |       yes |
| <>                                     |       yes |
| !=                                     |       yes |
| BETWEEN                                |       yes |
| IN                                     |       yes |
| LIKE                                   |       yes |
| _ (Matches any character)              |       yes |
| % (Matches any sequence of characters) |       yes |
| IS NULL                                |       yes |
| IS NOT NULL                            |       yes |
| + (Math Operator)                      |       yes |
| - (Math Operator)                      |       yes |
| * (Math Operator)                      |       yes |
| / (Math Operator)                      |       yes |
| % (Math Operator)                      |       yes |

### Aggregate Functions

| Aggregate Function | Supported |
| :----------------- | --------: |
| AVG(*expression*)  |       yes |
| COUNT              |       yes |
| MAX(*expression*)  |       yes |
| MIN(*expression*)  |       yes |
| SUM(*expression*)  |       yes |

### Conditional Functions

| Conditional Function | Supported |
| :------------------- | --------: |
| CASE                 |       yes |
| COALESCE             |       yes |
| NULLIF               |       yes |

### Conversion Functions

| Conversion Function | Supported |
| :------------------ | --------: |
| CASE                |       yes |

### Date Functions

| Date Function | Supported |
| :------------ | --------: |
| DATE_ADD      |       yes |
| DATE_DIFF     |       yes |
| EXTRACT       |       yes |
| TO_STRING     |    **no** |
| TO_TIMESTAMP  |    **no** |
| UTCNOW        |    **no** |

### String Functions

| String Function               | Supported |
| :---------------------------- | --------: |
| CHAR_LENGTH, CHARACTER_LENGTH |    **no** |
| LOWER                         |       yes |
| SUBSTRING                     |    **no** |
| TRIM                          |       yes |
| UPPER                         |       yes |

## Source

- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/s3api/select-object-content.html
- https://aws.amazon.com/about-aws/whats-new/2018/09/amazon-s3-announces-new-features-for-s3-select/
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/selecting-content-from-objects.html
- https://docs.aws.amazon.com/AmazonS3/latest/API/RESTSelectObjectAppendix.html
- https://aws.amazon.com/blogs/aws/s3-glacier-select/
