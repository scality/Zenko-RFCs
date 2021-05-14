# CRR Metrics

## Queue Populator Flow
```mermaid
sequenceDiagram
    participant raft as Raft Log
    participant pop as Queue Populator
    participant Kafka
    participant Prometheus
    pop-->>raft: Scrape logs
    pop->>Kafka: Push message
    loop Store metrics
        pop->>pop: Track usage
    end
    Prometheus-->>pop: Scrape metrics
```

## Queue Processor Flow
```mermaid
sequenceDiagram
    participant Kafka
    participant proc as Queue Processor
    participant source as Source
    participant dest as Destination
    participant Prometheus
    proc-->>Kafka: Pull message
    proc-->>source: Pull object
    loop Store metrics
        proc->>proc: Track usage
    end
    Prometheus-->>proc: Scrape pull metrics
    proc->>dest: Push object to destination
    loop Store metrics
        proc->>proc: Track usage
    end
    Prometheus-->>proc: Scrape push metrics
```

## Status Processor Flow
```mermaid
sequenceDiagram
    participant Kafka
    participant proc as Status Processor
    participant md as S3 Metadata
    participant Prometheus
    proc-->>Kafka: Pull message
    proc->>md: Update replication status
    loop Store metrics
        proc->>proc: Track usage
    end
    Prometheus-->>proc: Scrape metrics
```

## Metrics
| Name | Help | Type |
| ---- | ---- | ---- |
replication_populator_messages | Total number of kakfa messages produced by the queue populator | Counter
replication_populator_objects | Total objects queued for replication | Counter
replication_populator_bytes | Total number of bytes queued for replication not including metadata | Counter
replication_processor_messages | Total number of kakfa messages consumed by the queue processor | Counter
replication_processor_objects | Number of objects replicated | Counter
replication_processor_bytes | Number of bytes replicated not including metadata | Counter
replication_processor_elapsed_seconds | Replication jobs elapsed time in seconds | Histogram
replication_status_processor_objects | Number of objects updated | Counter

The difference between processor objects and queued objects is the current number of waiting objects.
The difference between processor bytes and queued bytes is the current amount of bytes waiting to replicate.

## Labels
| Name | Description |
| ---- | ----------- |
origin | What method began the replication
raftId | Which raft logs the populator is connected to
partition | Which Kafka partition the event comes from or is going to
sourceLocation | Which location our data replication is coming from, part of configuration
sourceLocationType | Type of location our replication is coming from, such as AWS or Memory
destinationLocation | Which location our data replication is going to, part of configuration
destinationLocationType | Type of location our replication is going to, such as AWS or Memory
kafkaStatus | Kafka execution status
contentLengthRange | Range of sizes of content the request falls into, such as 10MB to 30MB
replicationStatus | Result of the replication process; success or failed
