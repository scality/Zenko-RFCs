# Monitoring in S3 Connector and Zenko

Both S3 Connector and Zenko deploy a plethora of services that need to be healtchecked and monitored
and both products have a common architecture, Prometheus, Grafana and Alert Manager to establish a
monitoring pipeline.

## Monitoring toolkit

### Prometheus

A stack/toolkit containing a distributed time series databases that collects metrics from jobs or exporters and recompiles
them into rich timeseries metrics. These metrics are queriable via an API or used as a source for vizualization in tools like Grafana

### Grafana

Grafana is a visualization tool that can have many data sources such as Prometheus, custom data exporters, Elastic Search etc.
In the context of S3C and Zenko, Prometheus will be used as primary source for metrics.

### Alert Manager

Alert Manager can be used to push alerts by setting some alerting rules in Prometheus server. The Alert Manager can push alerts
to different destinations such as email, chat applications and on call notifcation systems.

## Metrics Collection

Each service built by Scality (Cloudserver, Vault, Metadata etc) as well as third-party services (Redis, Kafka, Zookeeper etc.) should be configured for metrics collection.
Metrics can be collected by Prometheus in either of the following ways

### Pushing metrics

Services that don't have a persistent storage for metrics can push metrics as they occur to Prometheus' push gateway.

**Note:** This has a potential of negative impact if it's implementented in the critical path of processing a request by a service when there are issues contacting the push gateway.
Other best practices are outlined here https://prometheus.io/docs/practices/pushing/

### Exporters

Services that have metrics already stored in some form can expose the metrics via a `/_/metrics` route. The following doc https://prometheus.io/docs/instrumenting/writing_exporters/ can be used as for guidelines and best practices for writing exporters.


## Shipping

The artifacts needed such as Grafana dashboards are shipped with installers. Given the nature of metrics, it's likely that the same chart can be used for both S3C and Zenko products.
