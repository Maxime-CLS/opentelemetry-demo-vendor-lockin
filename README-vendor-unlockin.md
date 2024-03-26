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
docker compose -f docker-compose-vendor-lockin.yml up -d 
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
    endpoint: "fleet-server:8200"
    headers:
      # Elastic APM Server secret token
      Authorization: "Bearer supersecrettoken"
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

## https://www.elastic.co/guide/en/observability/8.12/open-telemetry-resource-attributes.html#open-telemetry-resource-attributes
processors:
  resource:
    attributes:
      - key: deployment.environment
        action: insert
        value: demo


service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch, resource]
      exporters: [spanmetrics, otlp/elastic]
    metrics:
      receivers: [otlp, spanmetrics]
      processors: [memory_limiter, cumulativetodelta, batch]
      exporters: [otlp/elastic]
    logs:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlp/elastic]
```

#### Configure Fleet Server with APM Integration

First, we’ll need to pull the CA certificate from the cluster :

```
docker cp es01:/usr/share/elasticsearch/config/certs/ca/ca.crt ./
```

Next, we will need to get the fingerprint of the certificate. For this, we can use an OpenSSL command:

```
openssl x509 -fingerprint -sha256 -noout -in ca.crt | awk -F"=" {' print $2 '} | sed s/://g
```

This will produce a value similar to: 

`5A7464CEABC54FA60CAD3BDF16395E69243B827898F5CCC93E5A38B8F78D5E72`

Finally, we need to get the whole cert into a yml format. We can do this with a `cat` command or just by opening the cert in a text editor:

```
cat ca.crt
```

```
-----BEGIN CERTIFICATE-----
MIIDSTCCAjGgAwIBAgIUT5DmPOHDM2WXcnC7EeNiOOMa9rkwDQYJKoZIhvcNAQEL
BQAwNDEyMDAGA1UEAxMpRWxhc3RpYyBDZXJ0aWZpY2F0ZSBUb29sIEF1dG9nZW5l
cmF0ZWQgQ0EwHhcNMjQwMzI5MTAzNjQ0WhcNMjcwMzI5MTAzNjQ0WjA0MTIwMAYD
VQQDEylFbGFzdGljIENlcnRpZmljYXRlIFRvb2wgQXV0b2dlbmVyYXRlZCBDQTCC
ASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAKwnD+31VOVo1Njdqg+EFED5
ONF3E1RUaIbBdd5Y9n+D1dz95TTJY9rIKxFu122NpS9PTzpeM9Opt8EWMWeH35YS
iFp9PJ0VhmS6TbSsD3ZPuMvFMgGhDlERxvzWO1U9H3W816zA/PkbJoEmmahrlBYD
gGDMRsGOpA5FJxBNkyNL9c3148v4x2usNCh/rPP+/WxZpMf19TMKCHfh6BWDvk31
vfx+KS2dWRcZ6aDkGY9+ANNYVLUN4L/mb2fapLVJFN7Yz6IgGBK2qiU0CgeecKO/
KSHB2Jb0YZVfw1Nw01VeFpvhHcaIE0kBRx6NG6i1rai233AynsZ7Boqhbfb1cFsC
AwEAAaNTMFEwHQYDVR0OBBYEFFiG0P3UKIz0yR33osboCznpqZBVMB8GA1UdIwQY
MBaAFFiG0P3UKIz0yR33osboCznpqZBVMA8GA1UdEwEB/wQFMAMBAf8wDQYJKoZI
hvcNAQELBQADggEBAAj+cJCUaVWve0D0qsGiri4V5EQiJauEab0IyXQnZg85bKym
n1HnKLAob6sLuqjQXoHd05bWrLsm2O5qYkqHxQcCj0uzEiBSr0QfFfCk4BvJDd6N
duS6wJhnSPjjV5ib07BsRS2CM+l6BDWwHfszvhM3zFnKzWPrDYmVFPindqAV4skc
XlspSJlJmL7bSAqpEIcUH8H7tVBhkNRxA2NXv8/eCWXqJILNxxqS3dHFaFi91p25
XHvvijORGdZhXhvDumG1mX28Cv2Guluoq++wRK9lxGrDVSkO6gShcIE/po+kOV9Q
/UnuYFusDIAKAB7Q29Patef6852Q5A0vGSSSL+I=
-----END CERTIFICATE-----
```

Once you have the certificate text, we will add it to a yml format and input all this information into the Fleet Settings screen from earlier.

For “Hosts,” we will want to use “https://es01:9200”. This is because the container that hosts the Fleet server understands how to communicate with the es01 container to send data.

Input the fingerprint that was produced for the field “Elasticsearch CA trusted fingerprint.”

Finally, add the certificate text to the “Advanced YAML configuration.” Since this is a yml configuration, it will throw an error if not spaced correctly. 

Start with:

```
ssl:
  certificate_authorities:
    - |
```
**And then paste the certificate text, making sure the indentation is correct.**

Example:

![Settings Fleet Server](https://static-www.elastic.co/v3/assets/bltefdd0b53724fa2ce/blt0fd5e8f5c53241e8/65256f40fbc4f6c0df9ae3af/Screenshot_2023-10-10_at_9.35.17_AM.png)

To access of [Settings Fleet Server](https://localhost:5601/app/fleet/settings). 


Sources : 
* [Getting started with the Elastic Stack and Docker Compose: Part 2](https://www.elastic.co/fr/blog/getting-started-with-the-elastic-stack-and-docker-compose-part-2)
* [Elastic - OpenTelemetry native support](https://www.elastic.co/guide/en/observability/8.13/apm-open-telemetry-direct.html#apm-connect-open-telemetry-collector)
* [Elastic - OpenTelemetry Resource attributes](https://www.elastic.co/guide/en/observability/8.13/apm-open-telemetry-resource-attributes.html)
* [How to run the official OpenTelemetry Demo with Elastic](https://discuss.elastic.co/t/dec-22th-2022-en-how-to-run-the-official-opentelemetry-demo-with-elastic/321229)


 ### Update the otel-collector configuration

```
docker restart otel-col 
```

#### Kibana APM Access
[Kibana APM Dashboard](https://localhost:5601/app/apm/services?comparisonEnabled=true&environment=ENVIRONMENT_ALL&rangeFrom=now-15m&rangeTo=now&offset=1d)



### Down demo platform

```
docker compose -f docker-compose-elastic.yml down 
```

## Elastic Stack & Grafana Labs

### Edit a otelcol-config-vendor-lockin-extras.yml file

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
    endpoint: "fleet-server:8200"
    headers:
      # Elastic APM Server secret token
      Authorization: "Bearer supersecrettoken"
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
  otlp/tempo:
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

## https://www.elastic.co/guide/en/observability/8.12/open-telemetry-resource-attributes.html#open-telemetry-resource-attributes
processors:
  resource:
    attributes:
      - key: deployment.environment
        action: insert
        value: demo

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch, resource]
      exporters: [spanmetrics, otlp/elastic, otlp/tempo]
    metrics:
      receivers: [otlp, spanmetrics]
      processors: [memory_limiter, cumulativetodelta, batch]
      exporters: [otlp/elastic, otlphttp/mimir]
    logs:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlp/elastic, loki]
```

### Running Elastic Stack & Grafana Labs

```shell
docker restart otel-col 
```

#### Kibana & Grafana Access
* [Kibana Dashboard](https://localhost:5601)
* [Grafana Dashboard](http://localhost:8080/grafana/)

## Elastic Stack & Grafana Labs & Dynatrace

### Set up your Dynatrace account and environment variables

Create a Dynatrace account. If you don’t have one, you can use a [trial account](https://www.dynatrace.com/signup/).

Next, create an access token that includes scopes for the following:

* Ingest OpenTelemetry traces (openTelemetryTrace.ingest)
* Ingest metrics (metrics.ingest)
* Ingest logs (logs.ingest)

### Edit a otelcol-config-vendor-lockin-extras.yml file

#### Dynatrace Exporter

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
    endpoint: "fleet-server:8200"
    headers:
      # Elastic APM Server secret token
      Authorization: "Bearer supersecrettoken"
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
  otlp/tempo:
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
  # otlp/http exporter to Dynatrace. 
  otlphttp/dynatrace: 
   endpoint: "https://isi13930.live.dynatrace.com/api/v2/otlp" 
   headers: 
    Authorization: "Api-Token dt0c01.SSG2WEQB5TEQC4EOUZEY5EO7.BA2JPPHOGAD43GGGQPYMJGEZP6EQ55XFANHALUCWOANOL4NYALEUY6OAA6KTFQYK" 

## https://www.elastic.co/guide/en/observability/8.12/open-telemetry-resource-attributes.html#open-telemetry-resource-attributes
processors:
  resource:
    attributes:
      - key: deployment.environment
        action: insert
        value: demo

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch, resource]
      exporters: [spanmetrics, otlp/elastic, otlp/tempo, otlphttp/dynatrace]
    metrics:
      receivers: [otlp, spanmetrics]
      processors: [memory_limiter, cumulativetodelta, batch]
      exporters: [otlp/elastic, otlphttp/mimir, otlphttp/dynatrace]
    logs:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlp/elastic, loki, otlphttp/dynatrace]
```


### Use Dynatrace to observe traces, metrics, and logs

Open your Dynatrace environment in the browser (using the link from the account creation step). Using the navigation on the left, open the “Distributed traces” page. When the application is running, traces, metrics, and logs start appearing on Dynatrace.  

#### View traces 

In this view, you can filter traces. For example, to visualize traces for the checkout service, you can enter “checkout” in the Filter requests field, as shown in the following image.

![Dynatrace Distributed_traces](https://marvel-b1-cdn.bc0a.com/f00000000236551/dt-cdn.net/wp-content/uploads/2022/10/Distributed_traces_Screenshot.png)

If you select one of the traces, you can see the different spans going through multiple services, as shown in the following image.


![Dynatrace HTTP_post_trace](https://marvel-b1-cdn.bc0a.com/f00000000236551/dt-cdn.net/wp-content/uploads/2022/10/HTTP_post_trace_Screenshot.png)

#### View metrics

Next, take a look at the received metrics by opening the “Metrics” view in the left navigation. You should see a lot of different metrics. Filter using calls.count to see the metric created by the span metrics connector.

![Dynatrace Metrics_create_chart](https://marvel-b1-cdn.bc0a.com/f00000000236551/dt-cdn.net/wp-content/uploads/2023/01/Metrics_create_chart.png)

The span metrics connector shows the number of spans each demo app service creates. In the Data Explorer, use the “Split by” field with the value service.name to separate the number of spans per service. You can see that it’s the front-end service that creates the most spans.

![Dynatrace Metrics_data_explorer](https://marvel-b1-cdn.bc0a.com/f00000000236551/dt-cdn.net/wp-content/uploads/2023/01/Metrics_data_explorer.png)

#### View Logs

One of the latest additions in the demo app is logs. To access logs in Dynatrace, navigate to the Log Viewer page. Select one of the log lines, and a sidebar appears with the available attributes. Select “View trace” to connect your log lines with traces and see in detail which request (distributed trace) led to this log line. The magic in the log correlation is the trace_id and span_id attributes OpenTelemetry libraries attach to each log message if available.


![Dynatrace Log-viewer](https://marvel-b1-cdn.bc0a.com/f00000000236551/dt-cdn.net/wp-content/uploads/2023/01/Log-viewer.png)

Sources : 
* [Running the OpenTelemetry demo application with Dynatrace](https://www.dynatrace.com/news/blog/opentelemetry-demo-application-with-dynatrace/)
