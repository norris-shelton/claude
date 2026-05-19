---
description: Set up OpenTelemetry observability for a Spring Boot 3 Java service (Micrometer Tracing + OTLP → DataDog Agent)
---

# Java: Set Up OpenTelemetry

Configure a Spring Boot 3.x Java service for distributed tracing and metrics via Micrometer Tracing, OpenTelemetry, and
OTLP export to a local DataDog Agent.

> **Note:** Replace `traccount` / `traccount-web` placeholders below with this project's service name and web/app
module.

## 1. Add Maven dependencies

Add to the web/app module's `pom.xml` (e.g. `traccount-web/pom.xml`). Versions are managed by `spring-boot-dependencies`
BOM — do not pin them.

```xml
<!-- Micrometer Tracing for Spring Boot 3.x -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing</artifactId>
</dependency>

<!-- OpenTelemetry Bridge for Micrometer -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>

<!-- OpenTelemetry OTLP Exporter -->
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>

<!-- OpenTelemetry API for manual instrumentation -->
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-api</artifactId>
</dependency>

<!-- OpenTelemetry Context Propagation -->
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-context</artifactId>
</dependency>
```

## 2. Configure `application.properties`

Add to `src/main/resources/application.properties`. Replace `traccount` with this service's name.

```properties
# Application identification
spring.application.name=traccount

# Enable context propagation (recommended by Spring Boot 3)
spring.threads.virtual.enabled=false

# Enable tracing and observability
management.tracing.enabled=true
management.tracing.sampling.probability=0.1

# Enable observation annotations (Spring Boot 3 best practice)
management.observations.annotations.enabled=true

# Common observation key-values for dimensional drill-down
management.observations.key-values.service=traccount
management.observations.key-values.env=${spring.profiles.active:production}
management.observations.key-values.version=@project.version@

# OpenTelemetry OTLP configuration for DataDog Agent
management.otlp.tracing.endpoint=http://localhost:4318/v1/traces
management.otlp.tracing.timeout=30s

# OpenTelemetry resource attributes (Spring Boot 3 recommended approach)
management.opentelemetry.resource-attributes.service.name=traccount
management.opentelemetry.resource-attributes.service.version=@project.version@
management.opentelemetry.resource-attributes.deployment.environment=${spring.profiles.active:production}

# Baggage configuration for cross-service propagation
management.tracing.baggage.remote-fields=dd.service,dd.env,dd.version,akamai.global.identifier,amzn.trace.id,infosec.scanner,cloudfront.trace.id
management.tracing.baggage.correlation.fields=dd.service,dd.env,dd.version,akamai.global.identifier,amzn.trace.id,infosec.scanner,cloudfront.trace.id

# DataDog resource attributes
management.metrics.tags.service=traccount
management.metrics.tags.env=${spring.profiles.active:production}
management.metrics.tags.version=@project.version@

# Actuator endpoints
management.endpoints.web.exposure.include=health,metrics,info,prometheus
```

## 3. Verify Maven resource filtering

`@project.version@` requires Maven resource filtering. Confirm the build plugin is configured (Spring Boot's parent POM
enables this by default).

## 4. Verify the DataDog Agent (or OTel collector)

The OTLP endpoint defaults to `http://localhost:4318/v1/traces`. Ensure a local DataDog Agent (or OTel collector) is
listening on port 4318. Override `management.otlp.tracing.endpoint` per environment if needed.

## 5. Validate at runtime

After starting the app:

- `GET /actuator/health` — service up
- `GET /actuator/metrics` — Micrometer metrics exposed
- Hit any endpoint and confirm traces appear in DataDog APM under the `traccount` service.

## 6. Optional: manual instrumentation

Use `io.micrometer.observation.annotation.@Observed` on methods, or inject `Tracer` / `OpenTelemetry` for custom spans.

## Acceptance checklist

- [ ] Dependencies added to web/app module `pom.xml`
- [ ] `spring.application.name` set
- [ ] `management.tracing.*` properties configured
- [ ] `management.otlp.tracing.endpoint` points at the DataDog Agent / OTel collector
- [ ] `management.opentelemetry.resource-attributes.*` set
- [ ] Baggage `remote-fields` and `correlation.fields` include required propagation keys
- [ ] `/actuator/health` and `/actuator/metrics` return 200
- [ ] Traces visible in DataDog APM