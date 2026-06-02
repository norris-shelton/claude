---
description: Configure outbound HTTP for a Spring service — managed Apache HttpClient 5 connection pool with cookie storage disabled (prevents LB stickiness pinning). Triggers: "fix RestClient", "outbound HTTP pool", "stop sticky sessions", "AWSALB cookie", "disable cookie management".
argument-hint: [project-path]
allowed-tools: [Bash, Read, Edit, Write, Glob, Grep]
---

# Java: HttpClient 5 Outbound Configuration

Replaces both `java-fix-restclient-config` and `java-stop-sticky-sessions`. Establishes one canonical outbound HTTP client bean that is **pooled, properly timed-out, and cookie-disabled** — so:

- Connections are reused (no per-call socket setup) but TTL stays below the AWS ALB 60 s idle timeout to avoid half-closed connection reuse.
- LB stickiness cookies (`AWSALB`, `AWSALBCORS`, `BIGipServer*`, `JSESSIONID`) returned by downstream services are **not** stored or replayed, so server-to-server traffic is load-balanced per request.

Apply this once per service. If both `RestClient` and `RestTemplate` are used, share a single `CloseableHttpClient` between them.

## Step 0 — Baseline

Capture a test baseline so the verification step can prove no regression.

```bash
# Maven
./mvnw clean test 2>&1 | tee /tmp/hc5-baseline.txt
# Gradle
./gradlew clean test 2>&1 | tee /tmp/hc5-baseline.txt
```

Record total / passed / failed / skipped. Pre-existing failures must remain the only failures after the fix.

## Step 1 — Detect

Report findings before changing anything.

```bash
# Maven dependency check
./mvnw -q dependency:tree 2>/dev/null \
  | grep -iE 'httpclient|httpcore|okhttp|reactor-netty'
# Gradle dependency check
./gradlew -q dependencies --configuration runtimeClasspath 2>/dev/null \
  | grep -iE 'httpclient|httpcore|okhttp|reactor-netty'

# Where outbound clients are used
rg -n --type java --type kotlin -e 'RestTemplate|RestClient|WebClient|FeignClient|@FeignClient'

# Bean definitions (the fix goes here)
rg -n --type java --type kotlin -B1 -A2 -e '@Bean' \
  | rg -B2 -A2 -e 'RestTemplate|RestClient|WebClient|HttpClient|HttpComponents'

# Direct create() calls that bypass the shared bean
rg -n --type java --type kotlin -e 'RestClient\.create\(\)|new RestTemplate\('

# Any existing cookie handling (delete unless intentional per-request cookies)
rg -n --type java --type kotlin -e 'CookieStore|BasicCookieStore|CookieJar|CookieManager|CookieSpec|cookieCodec|\.cookie\(|setCookie'
```

### Why presence of httpclient on classpath matters

If Apache HttpClient 5 is on the classpath at all (even transitively), `RestTemplateBuilder` and `RestClient.Builder` auto-upgrade to `HttpComponentsClientHttpRequestFactory`, which installs a **default cookie store**. Presence alone enables the sticky-cookie bug. The fix below addresses both pooling and cookie storage in one bean — do not rely on auto-configuration.

## Step 2 — Add the dependency

Make `httpclient5` a **direct** dependency (not transitive — otherwise stripping a parent module will silently break the fix at runtime).

### Maven (`pom.xml` of the web/app module)

```xml
<dependency>
    <groupId>org.apache.httpcomponents.client5</groupId>
    <artifactId>httpclient5</artifactId>
</dependency>
```

### Gradle (`build.gradle` / `build.gradle.kts`)

```groovy
implementation 'org.apache.httpcomponents.client5:httpclient5'
```

Version is managed by the Spring Boot BOM; do not pin.

## Step 3 — Add properties

### `application.properties`

```properties
# Outbound HTTP pool (Apache HttpClient 5)
# TTL must stay below AWS ALB idle timeout (60s) so half-closed connections aren't reused.
http.client.pool.ttl-seconds=50
http.client.pool.max-total=200
http.client.pool.max-per-route=50
# Evict idle connections at the same cadence as the TTL.
http.client.pool.idle-evict-seconds=50
# Wait for a free connection from the pool (pool exhaustion guard).
http.client.connection-request-timeout-seconds=5
# TCP connect handshake.
http.client.connect-timeout-seconds=5
# Per-socket inactivity (read/write between bytes).
http.client.socket-timeout-seconds=30
# Total per-request budget (HC5 response timeout).
http.client.response-timeout-seconds=30
```

> **TTL vs idle-evict:** TTL is an absolute lifetime — once a connection is older than TTL, the pool will not lease it again. `idle-evict` runs a background sweeper that closes connections that have been idle longer than the threshold. With ALB-fronted upstreams both matter: TTL prevents a long-idle connection from being leased after the ALB silently dropped its half, and the evicter actively reclaims the FDs before then. They are normally equal; set `idle-evict < ttl` only if you want eager FD reclamation.

## Step 4 — Write the configuration class

Create (or rewrite) the single `@Configuration` class that owns the outbound HTTP wiring. Place it in the web/app module's base package (auto-derived from `src/main/java/<base>`).

```java
package <base.package>;

import java.time.Duration;
import java.util.concurrent.TimeUnit;

import org.apache.hc.client5.http.config.ConnectionConfig;
import org.apache.hc.client5.http.config.RequestConfig;
import org.apache.hc.client5.http.impl.classic.CloseableHttpClient;
import org.apache.hc.client5.http.impl.classic.HttpClients;
import org.apache.hc.client5.http.impl.io.PoolingHttpClientConnectionManager;
import org.apache.hc.client5.http.impl.io.PoolingHttpClientConnectionManagerBuilder;
import org.apache.hc.core5.util.TimeValue;
import org.apache.hc.core5.util.Timeout;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.MediaType;
import org.springframework.http.client.HttpComponentsClientHttpRequestFactory;
import org.springframework.web.client.RestClient;

@Configuration
public class HttpClientConfig {

    @Bean(destroyMethod = "close")
    public PoolingHttpClientConnectionManager httpClientConnectionManager(
            @Value("${http.client.pool.max-total}") int maxTotal,
            @Value("${http.client.pool.max-per-route}") int maxPerRoute,
            @Value("${http.client.pool.ttl-seconds}") long ttlSeconds,
            @Value("${http.client.socket-timeout-seconds}") long socketTimeoutSeconds,
            @Value("${http.client.connect-timeout-seconds}") long connectTimeoutSeconds) {

        ConnectionConfig connectionConfig = ConnectionConfig.custom()
                .setSocketTimeout(Timeout.ofSeconds(socketTimeoutSeconds))
                .setConnectTimeout(Timeout.ofSeconds(connectTimeoutSeconds))
                .setTimeToLive(TimeValue.ofSeconds(ttlSeconds))
                .build();

        return PoolingHttpClientConnectionManagerBuilder.create()
                .setMaxConnTotal(maxTotal)
                .setMaxConnPerRoute(maxPerRoute)
                .setDefaultConnectionConfig(connectionConfig)
                .build();
    }

    @Bean(destroyMethod = "close")
    public CloseableHttpClient httpClient(
            PoolingHttpClientConnectionManager connectionManager,
            @Value("${http.client.pool.idle-evict-seconds}") long idleEvictSeconds,
            @Value("${http.client.connection-request-timeout-seconds}") long connectionRequestTimeoutSeconds,
            @Value("${http.client.response-timeout-seconds}") long responseTimeoutSeconds) {

        RequestConfig requestConfig = RequestConfig.custom()
                .setConnectionRequestTimeout(Timeout.ofSeconds(connectionRequestTimeoutSeconds))
                .setResponseTimeout(Timeout.ofSeconds(responseTimeoutSeconds))
                .build();

        return HttpClients.custom()
                .setConnectionManager(connectionManager)
                .setDefaultRequestConfig(requestConfig)
                .evictExpiredConnections()
                .evictIdleConnections(TimeValue.ofSeconds(idleEvictSeconds))
                .disableCookieManagement()   // critical — prevents LB stickiness cookies from being replayed
                .build();
    }

    @Bean
    public RestClient restClient(CloseableHttpClient httpClient) {
        return RestClient.builder()
                .requestFactory(new HttpComponentsClientHttpRequestFactory(httpClient))
                .defaultHeader("Content-Type", MediaType.APPLICATION_JSON_VALUE)
                .defaultHeader("Accept", MediaType.APPLICATION_JSON_VALUE)
                .build();
    }
}
```

### If the project also uses `RestTemplate`

Share the same `CloseableHttpClient`:

```java
@Bean
public RestTemplate restTemplate(CloseableHttpClient httpClient, RestTemplateBuilder builder) {
    return builder.requestFactory(() -> new HttpComponentsClientHttpRequestFactory(httpClient)).build();
}
```

### Auto-deriving `<base.package>`

Find the base package the same way the rest of these skills do: read `src/main/java/` and take the deepest directory that contains the application's `@SpringBootApplication` class.

## Step 5 — Migrate ad-hoc clients

### `RestClient.create()` / `new RestTemplate()`

For each occurrence in `src/main/java`:

1. Read the surrounding method to find the request `Content-Type`.
2. Replace the local declaration with the injected `restClient` / `restTemplate` field.
3. **If the call expects `application/x-www-form-urlencoded`** (e.g., body is `MultiValueMap`), add `.contentType(MediaType.APPLICATION_FORM_URLENCODED)` to the per-request chain — the shared bean defaults to JSON and would break the call.
4. **If the call sets its own `Content-Type` via `.contentType(...)` or `.headers(...)`**, the per-request header overrides the default — safe to migrate as-is.
5. **If the content type is ambiguous**, leave a `// TODO: review — verify Content-Type before switching to shared client` comment and report the location instead of changing the code.

### Other client types (defensive — only if used)

Audit and disable cookies where present:

| Client | Disable-cookies snippet |
|---|---|
| `WebClient` + Jetty | `jetty.setHttpCookieStore(new HttpCookieStore.Empty());` |
| `WebClient` + JDK | `HttpClient.newBuilder().cookieHandler(null).build()` |
| OkHttp | `.cookieJar(okhttp3.CookieJar.NO_COOKIES)` |
| Feign HC5 | `new feign.hc5.ApacheHttp5Client(HttpClients.custom().disableCookieManagement().build())` |

`WebClient` + Reactor Netty has no cookie store by default; just ensure nothing calls `.cookie(...)` or attaches a cookie codec.

## Step 6 — Things NOT to do

- **Do not** strip cookies with a `ClientHttpRequestInterceptor` / `ExchangeFilterFunction`. Interceptors run after the cookie store has already attached cookies and before `Set-Cookie` is captured — the store still fills up.
- **Do not** rely on `Connection: close`. Cookies are stored at the `HttpClient` level, not the connection level.
- **Do not** spray a per-call `Cookie: AWSALB=` header to overwrite. The shared store still leaks across services.
- **Do not** instantiate a fresh `RestTemplate`/`RestClient` per call to "fix" stickiness — it drops connection pooling.

## Step 7 — Verification

### 7a — Run tests

```bash
./mvnw clean test 2>&1 | tee /tmp/hc5-after.txt
# or: ./gradlew clean test 2>&1 | tee /tmp/hc5-after.txt
```

Diff totals against `/tmp/hc5-baseline.txt` — passed/failed/skipped must match exactly. Any new failure is regression and must be resolved before continuing.

### 7b — Prove cookies are not replayed (no `tcpdump` required)

Add a small integration test that stands up MockWebServer (OkHttp), returns a `Set-Cookie: AWSALB=...` header on the first response, then asserts the second outbound request has **no** `Cookie` header.

```java
@Test
void shouldNotReplayLoadBalancerCookies() throws Exception {
    try (MockWebServer server = new MockWebServer()) {
        server.enqueue(new MockResponse()
                .addHeader("Set-Cookie", "AWSALB=test-value; Path=/")
                .setBody("{}"));
        server.enqueue(new MockResponse().setBody("{}"));
        server.start();

        String url = server.url("/").toString();
        restClient.get().uri(url).retrieve().toBodilessEntity();
        restClient.get().uri(url).retrieve().toBodilessEntity();

        RecordedRequest first = server.takeRequest();
        RecordedRequest second = server.takeRequest();
        assertThat(first.getHeader("Cookie")).isNull();
        assertThat(second.getHeader("Cookie")).isNull();   // regression guard
    }
}
```

Include this test in `src/test/java` so the fix can't silently regress.

### 7c — Optional: in-deployment check

Confirm downstream metrics / access logs show traffic spread across pods rather than pinned to one. If outbound load remains concentrated on a single instance, the fix is not active in that environment.

## Step 8 — Suggested commit message

```
configure pooled, cookie-disabled outbound HttpClient 5

Consolidates outbound HTTP wiring into a single managed CloseableHttpClient
bean with a sized connection pool, explicit timeouts, and
disableCookieManagement(). The cookie change prevents AWS ALB stickiness
cookies (AWSALB/AWSALBCORS) returned by downstream services from being
captured and replayed by the shared client, which previously pinned all
server-to-server traffic to a single backend instance.
```

## Related skills

- [[java-setup-opentelemetry]] — once outbound is consolidated, the shared HttpClient gets traced automatically via Micrometer/OTel instrumentation.
