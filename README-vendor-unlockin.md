# Opentelemetry Collector Vendor Unlocking Guide

## Elastic Stack

### Create a otelcol-config-elasticsearch-extras.yml file

#### Elasticsearch Exporter

[Documentation](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/elasticsearchexporter)

```yaml
# Copyright The OpenTelemetry Authors
# SPDX-License-Identifier: Apache-2.0

# extra settings to be merged into OpenTelemetry Collector configuration
# do not delete this file
exporters:
  logging:
    loglevel: debug
  elasticsearch/log:
    endpoints: [https://es01:9200]
    tls:
      insecure: true
      ca_file: /etc/certs/ca/ca.crt
    user: "elastic"
    password: "elastic"
    logs_index: logs-opentelemetry-index
    sending_queue:
      enabled: true
      num_consumers: 20
      queue_size: 1000
    timeout: 10s
  elasticsearch/trace:
    endpoints: [https://es01:9200]
    traces_index: apm-opentelemetry-index
    tls:
      insecure: true
      ca_file: /etc/certs/ca/ca.crt
    user: "elastic"
    password: "elastic"


service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [elasticsearch/trace]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [elasticsearch/log]
```

### Running Elastic Stack

```shell
docker compose -f docker-compose-elastic.yml up -d 
```

#### Kibana Access
[Kibana Dashboard](https://localhost:5601)

#### Exporter OTLP

* [Documentation OTLP HTTP](https://github.com/open-telemetry/opentelemetry-collector/blob/v0.96.0/exporter/otlphttpexporter/README.md)
* [Documentation OTLP GRPC](https://github.com/open-telemetry/opentelemetry-collector/blob/v0.96.0/exporter/otlpexporter/README.md)


```yaml
# Copyright The OpenTelemetry Authors
# SPDX-License-Identifier: Apache-2.0

# extra settings to be merged into OpenTelemetry Collector configuration
# do not delete this file
exporters:
  logging:
    loglevel: debug
  otlp/elastic:
    # Elastic APM server https endpoint without the "https://" prefix
    endpoint: "apm-server:8200"
    tls:
      insecure: true
    timeout: 10s
    sending_queue:
      enabled: true
      num_consumers: 20
      queue_size: 1000
    retry_on_failure:
      enabled: true
      initial_interval: 10s
      randomization_factor: 0.7
      multiplier: 1.3
      max_interval: 60s
      max_elapsed_time: 10m
    compression: gzip


service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp/elastic]
    metrics:
      receivers: [otlp]
      exporters: [otlp/elastic]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp/elastic]
```


 ### Update the otel-collector configuration

```
docker stop otel-col && docker rm otel-col && docker compose -f docker-compose-elastic.yml up -d 
```

### Down demo platform

```
docker compose -f docker-compose-elastic.yml down 
```

## Elastic Stack & Grafana Labs

### Create a otelcol-config-elasticsearch-grafana-extras.yml file

#### Grafana Exporter

```yaml
# Copyright The OpenTelemetry Authors
# SPDX-License-Identifier: Apache-2.0

# extra settings to be merged into OpenTelemetry Collector configuration
# do not delete this file
exporters:
  logging:
    loglevel: debug
  otlp/elastic:
    # Elastic APM server https endpoint without the "https://" prefix
    endpoint: "apm-server:8200"
    tls:
      insecure: true
    timeout: 10s
    sending_queue:
      enabled: true
      num_consumers: 20
      queue_size: 1000
    retry_on_failure:
      enabled: true
      initial_interval: 10s
      randomization_factor: 0.7
      multiplier: 1.3
      max_interval: 60s
      max_elapsed_time: 10m
    compression: gzip
  # Exporter for sending trace data to Tempo.
  otlp/grafana:
    # Send to the locally running Tempo service.
    endpoint: tempo:4317
    # TLS is not enabled for the instance.
    tls:
      insecure: true
  # Exporter for sending Prometheus data to Mimir.
  otlphttp/mimir:
    # Send to the locally running Mimir service.
    endpoint: http://mimir:9009/otlp
    # TLS is not enabled for the instance.
    tls:
      insecure: true
    timeout: 10s
  loki:
    endpoint: "http://loki:3100/loki/api/v1/push"
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp/elastic, otlp/grafana]
    metrics:
      receivers: [otlp]
      exporters: [otlp/elastic, otlphttp/mimir]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp/elastic, loki]
```

### Running Elastic Stack & Grafana Labs

```shell
docker compose -f docker-compose-elastic-grafana.yml up -d 
```

#### Kibana & Grafana Access
* [Kibana Dashboard](https://localhost:5601)
* [Grafana Dashboard](http://localhost:8080/grafana/)



```
docker compose -f docker-compose-elastic-grafana.yml up -d 
```
