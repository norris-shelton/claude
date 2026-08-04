---
name: java-setup-opentelemetry
model: sonnet
effort: medium
description: This skill should be used when a user wants to set up OpenTelemetry observability for a Spring Boot 3 Java service (Micrometer Tracing + OTLP → DataDog Agent or any OTel collector). Triggers include "set up OpenTelemetry", "configure tracing", "OTLP exporter", "Micrometer tracing", "DataDog APM", "no traces in DataDog", "missing traceId in logs", "trace context lost across services", "propagate trace headers", "add Akamai GRN", "X-Amzn-Trace-Id", "X-Amz-Cf-Id", "X-TS-AK-GRN", "virtual thread metrics", "virtual threads DataDog", "thread metrics flatlined", "VirtualThreadMetrics", "virtual thread pinning".
version: 0.9.0
metadata:
  author: Twinspires Engineering
  tags: workflow,java,observability,opentelemetry,micrometer,datadog,tracing
  alwaysApply: "false"
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

# Virtual threads (optional). Only enable after validating library compatibility and context
# propagation — and see section 9 for the DataDog metric impact and VirtualThreadMetrics setup.
spring.threads.virtual.enabled=${SPRING_THREADS_VIRTUAL_ENABLED:false}

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
    filesMatching(['application*.properties', 'application*.yml']) {
        // Replace Maven-style @...@ tokens in resources
        filter(org.apache.tools.ant.filters.ReplaceTokens, tokens: [
                'project.version': project.version.toString()
        ])
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

> **Propagation note.** The interceptor starts a *new* observation and explicitly opens a scope on the request thread (`observation.openScope()`), making it the *current* context. Spring's auto-instrumented `RestClient` / `WebClient` / `RestTemplate` only see this observation as their parent — and propagate its baggage downstream — when it's the current context. Without `openScope()`, downstream calls inherit the controller's observation (or none) and the telemetry key-values are lost. The scope is closed in `afterCompletion` before the observation is stopped. For hand-rolled `HttpClient` calls outside the request thread, wrap them in `observation.scoped(() -> ...)`.

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
    private static final String TELEMETRY_OBSERVATION_SCOPE_KEY = "telemetry.observation.scope";

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
            // Open a scope so this observation becomes the *current* context on the request thread.
            // Without openScope(), Spring's auto-instrumented RestClient/WebClient/RestTemplate will
            // not see this observation as the parent and the key-values won't propagate downstream.
            Observation.Scope scope = observation.openScope();
            request.setAttribute(TELEMETRY_OBSERVATION_KEY, observation);
            request.setAttribute(TELEMETRY_OBSERVATION_SCOPE_KEY, scope);
        } catch (Exception e) {
            logger.warn("Failed to process telemetry headers, continuing with default tracing", e);
        }
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                                Object handler, @Nullable Exception ex) {
        try {
            Observation.Scope scope = (Observation.Scope) request.getAttribute(TELEMETRY_OBSERVATION_SCOPE_KEY);
            if (scope != null) {
                scope.close();
            }
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

## 9. Virtual threads: keep DataDog telemetry meaningful

**JDK gate — check before applying.** This section requires **JDK 21+**. Determine the project's JDK from the build file (`maven.compiler.release` / `<java.version>` in `pom.xml`, or the Gradle toolchain / `sourceCompatibility`); if it is below 21, **skip this entire section** — do not add `micrometer-java21` or the binder (the app would fail at runtime with unsupported-class-version errors), and note that `spring.threads.virtual.enabled` is inert below JDK 21 anyway.

Apply this section whenever the service runs on JDK 21+ with `spring.threads.virtual.enabled=true` (the section 2 property). Trace reporting itself is unaffected — spans start on the request's virtual thread and export normally — but the standard thread metrics silently stop representing load:

- **`jvm.threads.*` goes quiet and misleads.** Micrometer's `JvmThreadMetrics` (`jvm.threads.live`, `jvm.threads.states`, `jvm.threads.peak`) is built on `ThreadMXBean`, which by spec does **not** count virtual threads. With platform threads those graphs track request concurrency; with virtual threads they flatline at a couple dozen carrier/housekeeping threads no matter how loaded the service is. Any DataDog monitor or dashboard keyed on thread count stops measuring anything.
- **`tomcat.threads.*` disappears.** `tomcat.threads.busy` / `tomcat.threads.current` / `tomcat.threads.config.max` reflect the connector's platform-thread pool, which Boot's virtual-thread customizer replaces with a virtual-thread-per-task executor.
- **Replacement load signals** to move dashboards/monitors to: `http.server.requests` active/throughput (unaffected, still the best request-level signal), `tomcat.connections.current` vs `server.tomcat.max-connections` (the new front-door cap), and the DataSource pool's `numActive` (the new throttle — see [[java-check-jmx-config]]).

### Add `VirtualThreadMetrics`

Micrometer's `micrometer-java21` module ships `VirtualThreadMetrics` — pinning duration (`jvm.threads.virtual.pinned`) and scheduler submit failures (`jvm.threads.virtual.submit.failed`) — the post-migration health signals to watch in DataDog. Version is managed by the Spring Boot BOM; do not pin. (The `java21` in the artifactId is Micrometer's naming convention for the *minimum* JDK whose APIs the module needs — not the JDK you must run. There is no `micrometer-java25`; this is the correct artifact for any JDK ≥ 21, including 25.)

Maven (`<web-module>/pom.xml`):

```xml
<!-- Virtual-thread metrics (pinning, submit failures) — only with spring.threads.virtual.enabled=true -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-java21</artifactId>
</dependency>
```

Gradle (`<web-module>/build.gradle[.kts]`):

```groovy
implementation 'io.micrometer:micrometer-java21'
```

On Spring Boot **3.5+** the dependency is all that's needed — Boot auto-configures the binder. On earlier 3.x, register it explicitly (Spring closes the binder's JFR stream on shutdown since it's `AutoCloseable`):

```java
import io.micrometer.java21.instrument.binder.jdk.VirtualThreadMetrics;

@Bean
public VirtualThreadMetrics virtualThreadMetrics() {
    return new VirtualThreadMetrics();
}
```

Verify: `curl -sf http://localhost:8080/actuator/metrics/jvm.threads.virtual.pinned` returns 200 once virtual threads are enabled and the binder is registered.

> **Pinning context.** On JDK 24+ (JEP 491), `synchronized` no longer pins carrier threads, so `jvm.threads.virtual.pinned` should stay near zero — only JNI/native frames still pin. A sustained non-zero pinned duration on JDK ≤ 23 usually points at `synchronized` I/O paths (e.g. older JDBC drivers).

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
- [ ] If on JDK 21+ with `spring.threads.virtual.enabled=true`: `micrometer-java21` added, `jvm.threads.virtual.pinned` present in `/actuator/metrics`, and dashboards/monitors moved off `jvm.threads.*` / `tomcat.threads.*` (section 9); if below JDK 21, section 9 skipped entirely

## Related skills

- [[java-pooled-restclient]] — outbound HTTP clients configured through that skill are auto-traced by Micrometer/OTel without extra wiring.
