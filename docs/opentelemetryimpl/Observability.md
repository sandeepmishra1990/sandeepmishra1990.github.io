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


  ## tracing run jaeger UI
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


## Spring Boot Dependencies to enable OpenTelemetry
---------------------

```
experimental tracing,log exports libs
----------------
implementation 'io.micrometer:micrometer-tracing-bridge-otel'
implementation 'io.micrometer:micrometer-tracing-bridge-brave'
runtimeOnly 'io.micrometer:micrometer-registry-otlp'

for sending traces to otel collector
-----------------
implementation 'io.opentelemetry:opentelemetry-exporter-otlp'

```

## Application.yaml props
---------------------

```
logging.pattern.correlation=[${spring.application.name:},%X{traceId:-},%X{spanId:-}] 
logging.include-application-name=false
logging.level.root=DEBUG
logging.level.com.example.demo=DEBUG
logging.level.org.springframework.beans.factory=DEBUG

logging.level.io.micrometer.tracing=DEBUG
logging.level.io.opentelemetry=DEBUG


management.security.enabled=false
management.endpoints.web.exposure.include=metrics,health

management.observations.key-values.env=prod
management.observations.enable.all=true
management.observations.http.client.requests.name=demo-client-observation
management.observations.http.server.requests.name=demo-server-observation
management.observations.long-task-timer.enabled=false


management.tracing.enabled=true
management.tracing.sampling.probability=1.0

management.otlp.tracing.endpoint=http://localhost:4317
management.otlp.tracing.transport=GRPC

```

## Build file used for demo
---------------------------
```
plugins {
	id 'java'
	id 'org.springframework.boot' version '3.5.4'
	id 'io.spring.dependency-management' version '1.1.7'
}

group = 'com.example'
version = '0.0.1-SNAPSHOT'
description = 'Demo project for Spring Boot'

java {
	toolchain {
		languageVersion = JavaLanguageVersion.of(21)
	}
}

repositories {
	mavenCentral()
}

dependencies {
	//general app libs
	implementation 'org.springframework.boot:spring-boot-starter-web'


	//operational libs.
	implementation 'org.springframework.boot:spring-boot-starter-aop'
	developmentOnly 'org.springframework.boot:spring-boot-devtools'
	implementation 'org.springframework.boot:spring-boot-starter-actuator'


	// experimental tracing,log exports libs
	implementation 'io.micrometer:micrometer-tracing-bridge-otel'
	//implementation 'io.micrometer:micrometer-tracing-bridge-brave'
	runtimeOnly 'io.micrometer:micrometer-registry-otlp'
    //for sending traces to otel collector
	implementation 'io.opentelemetry:opentelemetry-exporter-otlp'



	//test libs
	testImplementation 'org.springframework.boot:spring-boot-starter-test'
	testRuntimeOnly 'org.junit.platform:junit-platform-launcher'


	//logging libs
	implementation 'org.slf4j:slf4j-api' // SLF4J API
	runtimeOnly 'ch.qos.logback:logback-classic' // Logback implementation

}

tasks.named('test') {
	useJUnitPlatform()
}
```


## Spring BEan Required to overwrite the spanExporting behaviour, example to exclude the actuator calls.

```
    @Bean
    public SpanExportingPredicate excludeActuatorPredicate() {
        return span -> {
            String url = span.getTags().get("http.url");
            String target = span.getTags().get("http.target");
            String route = span.getTags().get("http.route");
            // Drop if any tag matches actuator
            System.out.println("************"+url);
            if ((url != null && url.startsWith("/actuator")) ||
                    (target != null && target.startsWith("/actuator")) ||
                    (route != null && route.startsWith("/actuator"))) {
                return false;
            }
            return true;
        };
    }
```

## Spring Bean required to enable the AOP for observability
```
    @Bean
    public ObservedAspect observedAspect(ObservationRegistry registry) {
        return new ObservedAspect(registry);
    }
```

## Spring Bean to overwrite the text publsher to write the spans in logs 
```
   @Bean
    public ObservationHandler<?> observationTextPublisher() {
        return new ObservationTextPublisher(logger::info);
    }
```
## To provide custom name to endpoints 

```
@Observed(contextualName="DemoService.httptest.call", name="DemoService.httptest.call")
    @GetMapping("/httptest")
    public String call()
    {
        return service.testhttpservice();
    }
```


Architecture Image 
----------------------------
[Image](otel-arch.png)
