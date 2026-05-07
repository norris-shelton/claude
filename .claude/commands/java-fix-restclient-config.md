---
description: Fix outbound HTTP client configuration by adding a managed Apache HttpClient 5 connection pool
argument-hint: [ project-path ]
allowed-tools: [ Bash, Read, Edit, Write, Glob, Grep ]
---

# Fix REST Client Configuration

Fix the outbound HTTP client configuration for this Spring Boot project by introducing a properly managed Apache
HttpClient 5 connection pool. Work through each step below in order.

## Step 0 — Baseline test run

Run `mvn clean test` and record the number of tests that pass and fail. This is the baseline. The verification step at
the end must pass at least as many tests as this baseline — do no harm.

## Step 1 — Add the httpclient5 dependency

Locate the `pom.xml` that manages the web/application module dependencies. Add the following dependency (no version —
managed by the Spring Boot BOM) if it is not already present:

```xml

<dependency>
    <groupId>org.apache.httpcomponents.client5</groupId>
    <artifactId>httpclient5</artifactId>
</dependency>
```

## Step 2 — Add connection pool properties

Locate the main `application.properties` or `application.yml` for the project. Add the following properties if they are
not already present. The TTL must stay below the AWS ALB idle timeout (60 s default) to prevent reusing half-closed
connections.

```properties
# Outbound HTTP connection pool (Apache HttpClient 5)
# TTL must be below AWS ALB idle timeout (60s default) to prevent reusing half-closed connections
http.client.pool.ttl-seconds=50
http.client.pool.max-total=200
http.client.pool.max-per-route=50
# Evict connections that have been idle longer than this threshold (typically aligned with the TTL)
http.client.pool.idle-evict-seconds=50
http.client.connect-timeout-seconds=5
http.client.response-timeout-seconds=30
```

## Step 3 — Rewrite the RestClient configuration class

Search `src/main/java` for the Spring `@Configuration` class that declares the `RestClient` bean (search for
`RestClient` combined with `@Bean` or `@Configuration`). Rewrite it so that it:

- Declares a `HttpClientConnectionManager` `@Bean` (so Spring calls `close()` on it at context shutdown) built with
  `PoolingHttpClientConnectionManagerBuilder` using `@Value`-injected properties from Step 2.
- Declares a `RestClient` `@Bean` that wires a `CloseableHttpClient` (configured with the connection manager,
  `evictExpiredConnections()`, `evictIdleConnections`, and a `RequestConfig` with connect/response timeouts) into an
  `HttpComponentsClientHttpRequestFactory`, then passes it to `RestClient.builder()` along with default JSON
  `Content-Type` and `Accept` headers.

## Step 4 — Audit every RestClient.create() call

Search the entire `src/main/java` source tree for `RestClient.create()`. For each occurrence:

1. Read the surrounding method to understand what HTTP call is being made.
2. Determine what `Content-Type` the upstream endpoint expects:
    - If the body is a POJO/DTO → JSON → the global `restClient` bean (which defaults to `application/json`) is a direct
      replacement; remove the local declaration and use the injected field.
    - If the body is a `MultiValueMap` or the endpoint requires `application/x-www-form-urlencoded` → the injected
      `restClient` defaults to `application/json` and **will break the call** unless
      `.contentType(MediaType.APPLICATION_FORM_URLENCODED)` is added to that specific request chain. Make the
      substitution and add the explicit content type.
    - If the call sets its own `Content-Type` explicitly (via `.contentType(...)` or `.headers(...)`) → the global bean
      is safe to use; the per-request header overrides the default.
    - If the content type or endpoint behavior is ambiguous → **do not change the call**; instead leave a
      `// TODO: review — RestClient.create() not converted; verify Content-Type before switching to injected restClient`
      comment and report the location to the developer.
3. Replace the local `RestClient` variable declaration and all usages with the injected `restClient` field, applying any
   content-type fix from point 2.

## Step 5 — Build verification

Run `mvn clean test` to confirm the project compiles cleanly and all tests pass. Compare the results against the Step 0
baseline — the number of passing tests must be greater than or equal to the baseline. Report any errors, test failures,
or regressions.

## Step 6 — Summary

Report:

- Which `RestClient.create()` calls were converted automatically.
- Which calls required a `.contentType(...)` fix.
- Which calls (if any) were left unconverted with a TODO comment and why.
