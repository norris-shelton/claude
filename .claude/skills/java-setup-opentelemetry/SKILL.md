---
description: Set up OpenTelemetry observability for a Spring Boot 3 Java service (Micrometer Tracing + OTLP → DataDog Agent or any OTel collector). Auto-derives base package and service name; Maven/Gradle. Triggers: set up OpenTelemetry, enable distributed tracing, add observability, configure tracing, send traces to DataDog, OTLP exporter, wire up OTel collector, Micrometer tracing, DataDog APM, propagate trace headers, no traces in DataDog, missing traceId in logs, trace context lost across services, spans not exported, baggage not propagated, micrometer-tracing-bridge-otel, management.otlp.tracing.endpoint, ObservationRegistry, @Observed annotation, X-Amzn-Trace-Id, X-Amz-Cf-Id, X-TS-AK-GRN, akamai global identifier, add Akamai GRN, add Akamai tracing, CloudFront trace.
argument-hint: [project-path]
allowed-tools: [Bash, Read, Edit, Write, Glob, Grep]
---

# Java: Set Up OpenTelemetry

Configure a Spring Boot 3.x Java service for distributed tracing and metrics via Micrometer Tracing, OpenTelemetry, and OTLP export to a DataDog Agent (or any OTel collector).

## 0. Auto-derive service identity

Before editing anything, derive these from the project (do not ask the user):

- **`<service>`** — the application's Spring `spring.application.name`. If unset, default to the artifactId of the web/app module.
- **`<web-module>`** — the Maven/Gradle module that contains the `@SpringBootApplication` class.
- **`<base.package>`** — the package of that `@SpringBootApplication` class (deepest common package under `src/main/java`).

Detection:

```bash
# Find the application class and its package
rg -l --type java '@SpringBootApplication' src/main/java
# → derive <base.package> from its package declaration
# → <web-module> is the gradle/maven module containing that file
```

All placeholders below (`<service>`, `<base.package>`) refer to those values.

## 1. Add dependencies

### Maven (`<web-module>/pom.xml`)

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
```

### Gradle (`<web-module>/build.gradle[.kts]`)

```groovy
implementation 'io.micrometer:micrometer-tracing'
implementation 'io.micrometer:micrometer-tracing-bridge-otel'
implementation 'io.opentelemetry:opentelemetry-exporter-otlp'
```

`opentelemetry-api` and `opentelemetry-context` are pulled in transitively — do not declare them explicitly. Versions are managed by `spring-boot-dependencies` BOM; do not pin.

## 2. Configure `application.properties`

Add to `src/main/resources/application.properties`. Substitute the auto-derived `<service>`.

```properties
# Application identification
spring.application.name=<service>

# Enable context propagation (recommended by Spring Boot 3)
spring.threads.virtual.enabled=false

# Tracing & observability
management.tracing.enabled=true
# Sampling — 1.0 in dev, lower in high-traffic prod. Override via env var per environment.
management.tracing.sampling.probability=${OTEL_TRACES_SAMPLER_ARG:0.1}

# Observation annotations (@Observed support)
management.observations.annotations.enabled=true

# Common observation key-values for dimensional drill-down
management.observations.key-values.service=<service>
management.observations.key-values.env=${spring.profiles.active:production}
management.observations.key-values.version=@project.version@

# OTLP export — defaults to local DataDog Agent. In containers/K8s, set
# OTEL_EXPORTER_OTLP_ENDPOINT=http://${DD_AGENT_HOST}:4318 (or your collector).
management.otlp.tracing.endpoint=${OTEL_EXPORTER_OTLP_ENDPOINT:http://localhost:4318}/v1/traces
management.otlp.tracing.timeout=30s

# OpenTelemetry resource attributes
management.opentelemetry.resource-attributes.service.name=<service>
management.opentelemetry.resource-attributes.service.version=@project.version@
management.opentelemetry.resource-attributes.deployment.environment=${spring.profiles.active:production}

# Baggage propagation (CDN / infra trace headers)
management.tracing.baggage.remote-fields=dd.service,dd.env,dd.version,akamai.global.identifier,amzn.trace.id,infosec.scanner,cloudfront.trace.id
management.tracing.baggage.correlation.fields=dd.service,dd.env,dd.version,akamai.global.identifier,amzn.trace.id,infosec.scanner,cloudfront.trace.id

# DataDog dimensional tags
management.metrics.tags.service=<service>
management.metrics.tags.env=${spring.profiles.active:production}
management.metrics.tags.version=@project.version@

# Actuator endpoints
management.endpoints.web.exposure.include=health,metrics,info,prometheus
```

> **Sampling guidance.** `0.1` (10%) suits steady production traffic. Use `1.0` in dev/staging to see every request, and consider `0.01` or lower if production is high-volume and you're paying per span. Override with the `OTEL_TRACES_SAMPLER_ARG` env var without changing the file.

> **Endpoint guidance.** `localhost:4318` is correct for a DataDog Agent running as a sidecar or host daemon. In Kubernetes set `OTEL_EXPORTER_OTLP_ENDPOINT=http://$(DD_AGENT_HOST):4318` via the downward API; in ECS use the service-discovery DNS name of your collector.

## 3. Verify Maven/Gradle resource filtering

`@project.version@` requires resource filtering. `spring-boot-starter-parent` enables this by default. If the project does **not** inherit it, add to `<web-module>/pom.xml`:

```xml
<build>
    <resources>
        <resource>
            <directory>src/main/resources</directory>
            <filtering>true</filtering>
            <includes>
                <include>application*.properties</include>
                <include>application*.yml</include>
            </includes>
        </resource>
    </resources>
</build>
```

For Gradle, add to `<web-module>/build.gradle`:

```groovy
processResources {
    filesMatching('application*.properties') {
        expand(project.properties)
    }
    filesMatching('application*.yml') {
        expand(project.properties)
    }
}
```

Verify by building and grepping the output: `@project.version@` should be replaced with the actual version.

## 4. Verify the DataDog Agent (or OTel collector)

Ensure the resolved endpoint is reachable from the app's runtime environment. Quick check:

```bash
curl -sf -X POST "${OTEL_EXPORTER_OTLP_ENDPOINT:-http://localhost:4318}/v1/traces" \
  -H 'Content-Type: application/json' -d '{"resourceSpans":[]}' \
  && echo "OTLP endpoint reachable"
```

## 5. Create the `TelemetryObservationInterceptor`

This `HandlerInterceptor` extracts CDN / infra telemetry headers from inbound requests, puts them in SLF4J MDC for log correlation, and starts a Micrometer `Observation` so the key-values become baggage propagated to downstream services.

> **Propagation note.** The interceptor starts a *new* observation rather than mutating the controller's existing observation. Baggage attached here propagates to downstream calls **only if** the outbound HTTP client runs within this observation's scope — Spring's auto-instrumented `RestClient` / `WebClient` / `RestTemplate` (when used with the Micrometer-instrumented `ObservationRegistry`) handle this automatically. If you have hand-rolled `HttpClient` calls, wrap them in `observation.scoped(() -> ...)` or they won't carry the headers.

Place in `<base.package>.web`.

```java
package <base.package>.web;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.lang.Nullable;
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.HandlerInterceptor;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import io.micrometer.observation.Observation;
import io.micrometer.observation.ObservationRegistry;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.slf4j.MDC;

@Component
@ConditionalOnProperty(name = "management.tracing.enabled", havingValue = "true")
public class TelemetryObservationInterceptor implements HandlerInterceptor {

    private static final Logger logger = LoggerFactory.getLogger(TelemetryObservationInterceptor.class);
    private static final String TELEMETRY_OBSERVATION_KEY = "telemetry.observation";

    private final ObservationRegistry observationRegistry;

    @Autowired
    public TelemetryObservationInterceptor(ObservationRegistry observationRegistry) {
        this.observationRegistry = observationRegistry;
    }

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        try {
            Observation observationBuilder =
                    Observation.createNotStarted("telemetry.correlation", observationRegistry)
                               .lowCardinalityKeyValue("http.method", request.getMethod())
                               .lowCardinalityKeyValue("component", "telemetry-interceptor")
                               .highCardinalityKeyValue("http.url", request.getRequestURL().toString());

            String amznTraceId = request.getHeader("X-Amzn-Trace-Id");
            if (amznTraceId != null && !amznTraceId.trim().isEmpty()) {
                addKey("AmznTraceId", amznTraceId);
                observationBuilder = observationBuilder.lowCardinalityKeyValue("amzn.trace.id", amznTraceId);
            }

            String scanner = request.getHeader("X-Scanner");
            if (scanner != null && !scanner.trim().isEmpty()) {
                addKey("Infosec", scanner);
                observationBuilder = observationBuilder.lowCardinalityKeyValue("infosec.scanner", scanner);
            }

            String cfTraceId = request.getHeader("X-Amz-Cf-Id");
            if (cfTraceId != null && !cfTraceId.trim().isEmpty()) {
                addKey("CfTraceId", cfTraceId);
                observationBuilder = observationBuilder.lowCardinalityKeyValue("cloudfront.trace.id", cfTraceId);
            }

            String akamaiIdentifier = request.getHeader("X-TS-AK-GRN");
            if (akamaiIdentifier != null && !akamaiIdentifier.trim().isEmpty()) {
                addKey("AkTraceId", akamaiIdentifier);
                observationBuilder =
                        observationBuilder.lowCardinalityKeyValue("akamai.global.identifier", akamaiIdentifier);
            }

            Observation observation = observationBuilder.start();
            request.setAttribute(TELEMETRY_OBSERVATION_KEY, observation);
        } catch (Exception e) {
            logger.warn("Failed to process telemetry headers, continuing with default tracing", e);
        }
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                                Object handler, @Nullable Exception ex) {
        try {
            Observation observation = (Observation) request.getAttribute(TELEMETRY_OBSERVATION_KEY);
            if (observation != null) {
                if (ex != null) {
                    observation.error(ex);
                }
                observation.stop();
            }
            MDC.remove("AmznTraceId");
            MDC.remove("Infosec");
            MDC.remove("AkTraceId");
            MDC.remove("CfTraceId");
        } catch (Exception e) {
            logger.warn("Error during telemetry observation cleanup", e);
        }
    }

    private void addKey(String key, String value) {
        if (value != null && !value.trim().isEmpty()) {
            MDC.put(key, value);
        }
    }
}
```

Register on a `WebMvcConfigurer`:

```java
@Override
public void addInterceptors(InterceptorRegistry registry) {
    registry.addInterceptor(telemetryObservationInterceptor);
}
```

Headers captured:

| Header            | MDC key       | Observation key            |
|-------------------|---------------|----------------------------|
| `X-Amzn-Trace-Id` | `AmznTraceId` | `amzn.trace.id`            |
| `X-Scanner`       | `Infosec`     | `infosec.scanner`          |
| `X-Amz-Cf-Id`     | `CfTraceId`   | `cloudfront.trace.id`      |
| `X-TS-AK-GRN`     | `AkTraceId`   | `akamai.global.identifier` |

## 6. Update `logback.xml` (or `logback-spring.xml`) log pattern

Append the MDC keys so they show in every log line. The `:--` default keeps the field present (as `-`) when absent.

```
|traceId-%X{traceId}|spanId-%X{spanId}|AmznTraceId-%X{AmznTraceId:--}|Infosec-%X{Infosec:--}|AkTraceId-%X{AkTraceId:--}|CfTraceId-%X{CfTraceId:--}| %xException{full} %n
```

## 7. Validate at runtime

```bash
curl -sf http://localhost:8080/actuator/health
curl -sf http://localhost:8080/actuator/metrics
```

Hit any application endpoint and confirm traces appear in DataDog APM under the `<service>` service name.

## 8. Optional: manual instrumentation

Use `io.micrometer.observation.annotation.@Observed` on methods, or inject `Tracer` / `OpenTelemetry` for custom spans.

## Acceptance checklist

- [ ] Dependencies added to `<web-module>` build file (Maven or Gradle)
- [ ] `spring.application.name` set to `<service>`
- [ ] `management.tracing.*` properties configured
- [ ] OTLP endpoint resolves to a reachable agent/collector (sampling/endpoint env vars documented for non-prod environments)
- [ ] `management.opentelemetry.resource-attributes.*` set
- [ ] Baggage `remote-fields` and `correlation.fields` include required propagation keys
- [ ] Resource filtering verified (`@project.version@` is replaced at build time)
- [ ] `TelemetryObservationInterceptor` created in `<base.package>.web` and registered via `WebMvcConfigurer`
- [ ] `logback.xml` pattern includes `AmznTraceId`, `Infosec`, `AkTraceId`, `CfTraceId` MDC keys
- [ ] `/actuator/health` and `/actuator/metrics` return 200
- [ ] Traces visible in DataDog APM

## Related skills

- [[java-pooled-restclient]] — outbound HTTP clients configured through that skill are auto-traced by Micrometer/OTel without extra wiring.
