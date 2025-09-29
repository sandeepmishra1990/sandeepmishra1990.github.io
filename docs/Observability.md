# documentation: https://opentelemetry.io/docs/collector/installation/
-------------------------------------------------------------------

## set docker network
-----------------------
```
docker network create monitoring
```

//this is to run trace generator client
----------------------
```
export GOBIN=${GOBIN:-$(go env GOPATH)/bin}
```

## Otel Collector (individual) or below
------------------------
```
docker pull otel/opentelemetry-collector-contrib:0.132.0
```

## simulate client to generate traces,logs,metrics
-------------------------
```
go install github.com/open-telemetry/opentelemetry-collector-contrib/cmd/telemetrygen@latest
```

## otel collector run--> collector with and without config
--------------------------

//run the collector without config file
//this will run the collector with default configuration
//it will listen on port 4317 for OTLP traces and metrics
//and on port 4318 for OTLP logs
//and on port 55679 for Jaeger traces
//you can change the ports if needed
//you can also add a config file to customize the collector
//for example, you can create a config file named otel-collector-config.yaml
//and run the collector with it using the -c flag
//docker run -p 4317:4317 -p 4318:4318 -p 55679:55679 \
//  -v $(pwd)/otel-collector-config.yaml:/otel-collector-config.yaml \
//  otel/opentelemetry-collector-contrib:0.132.0 \
//  --config /otel-collector-config.yaml

## run the collector with default configuration

```
docker run -d --name=otel-collector \
  --network monitoring \
  -p 4317:4317 \
  -p 4318:4318 \
  -p 55679:55679 \
  otel/opentelemetry-collector-contrib:0.132.0 \
  2>&1 | tee collector-output.txt # Optionally tee output for easier search later
```

## otel collecotr run with config
-----------------------------
```
docker run -d --name=otel-collector \
  --network monitoring \
  -p 4317:4317 \
  -p 4318:4318 \
  -p 55679:55679 \
  -p 8889:8889 \
  -v $(pwd)/otel-config.yaml:/etc/otel/config.yaml \
  otel/opentelemetry-collector-contrib:0.132.0 \
  --config /etc/otel/config.yaml \
  2>&1 | tee collector-output.txt

```

## For an easier time seeing relevant output you can filter it:
------------------------------
```
$GOBIN/telemetrygen traces --otlp-insecure \
  --traces 3 2>&1 | grep -E 'start|traces|stop'

//to check collector logs if traces are generated or not
---------------------------
$ grep -E '^Span|(ID|Name|Kind|time|Status \w+)\s+:' ./collector-output.txt
```

## visualization
-------------------------
//whilesetting the datasource use url as http://host.docker.internal:9090 not localhost because one container willl call other container inside docker and localhost will not work in that case.

```
docker run -d --name=grafana  --network monitoring -p 3000:3000 grafana/grafana
```

 ## setup prometheus execute from Otel-collec-jaeger folder all the commands
----------------------
```
docker run -d --name prometheus \
  --network monitoring \
  --name prometheus \
  -p 9090:9090 \
  -v $(pwd)/prometheus:/etc/prometheus \
  prom/prometheus
```


  ## tracing -- run jaeger UI
  -------------------------
```
docker run -d --name jaeger-ui \
  --network monitoring \
  -p 16686:16686 \
  -p 9411:9411 \
  -e COLLECTOR_ZIPKIN_HOST_PORT=:9411 \
  jaegertracing/all-in-one:latest
```


## logs loki
-------------------------

```
docker run -d --name loki \
  --network monitoring \
  -p 3100:3100 \
  grafana/loki:latest \
  -config.file=/etc/loki/local-config.yaml
```



 ## Configuration file for the otel otel-config.yaml
--------------------
```
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

exporters:
  prometheus:
    endpoint: "0.0.0.0:8889"
  debug:
    verbosity: detailed
  zipkin:
    endpoint: "http://jaeger-ui:9411/api/v2/spans"
  otlphttp/logs:
    endpoint: "http://loki:3100/otlp"
    tls:
     insecure: true


service:
  pipelines:
    metrics:
      receivers: [otlp]
      processors: []
      exporters: [prometheus]
    traces:
      receivers: [otlp]
      processors: []
      exporters: [debug, zipkin]
    logs:
      receivers: [otlp]
      exporters: [otlphttp/logs]

```

## docker-compose file for spinning the jaeger UI
---------------------------
```
version: '3.7'
services:
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686" # the jaeger UI
      - "4317:4317" # the OpenTelemetry collector grpc
      - "4318:4318"
    environment:
      - COLLECTOR_OTLP_ENABLED=true

```

[Architecture Image](docs/otel-arch.png)

