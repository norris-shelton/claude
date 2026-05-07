---
description: Audit Spring/Java outbound HTTP clients and disable cookie storage to prevent load-balancer stickiness cookies from pinning server-to-server traffic
argument-hint: [project-path]
allowed-tools: [Bash, Read, Edit, Write, Glob, Grep]
---

# Stop Sticky Sessions

Use this command when a service hosted behind a sticky load balancer is also a *client* of other services behind sticky
load balancers, and outbound calls are unintentionally being pinned to a single backend instance.

## The problem in one paragraph

Sticky-session load balancers (AWS ALB, classic ELB with stickiness, F5, NGINX `ip_hash`/cookie, etc.) work by setting a
`Set-Cookie` response header (`AWSALB`, `AWSALBCORS`, `BIGipServer<pool>`, `JSESSIONID`, etc.). Any HTTP client that *
*stores cookies and replays them** will, after its first request to a downstream service, keep sending the same cookie
on every subsequent request — and the LB will dutifully route every one of them to the same backend instance. Because
Spring HTTP-client beans are usually singletons, a single shared cookie store pins **all** outbound traffic from this
JVM, regardless of which inbound request or which user triggered it.

The fix is to disable cookie storage on every outbound HTTP client. Inbound stickiness (browser → this server) is
unaffected — that is enforced by the LB sitting in front of this server, not by anything inside the JVM.

## Step 1 — Detect

Run these checks. Report findings before changing anything.

```bash
# Identify HTTP-client libraries on the classpath (the underlying transport).
# Look for any of: apache httpclient 4, apache httpclient 5, okhttp, reactor-netty, jdk java.net.http.
mvn -q dependency:tree 2>/dev/null | grep -iE 'httpclient|httpcore|okhttp|reactor-netty|java\.net\.http' || \
  ./gradlew -q dependencies 2>/dev/null | grep -iE 'httpclient|httpcore|okhttp|reactor-netty'

# Identify Spring HTTP-client usage in source code.
rg -n --type java --type kotlin -e 'RestTemplate|RestClient|WebClient|FeignClient|@FeignClient'

# Identify @Bean definitions that create those clients (these are where the fix goes).
rg -n --type java --type kotlin -B1 -A2 -e '@Bean' | rg -B2 -A2 -e 'RestTemplate|RestClient|WebClient|HttpClient'

# Look for any explicit cookie handling already in place — these may need removal or auditing.
rg -n --type java --type kotlin -e 'CookieStore|BasicCookieStore|CookieJar|CookieManager|CookieSpec|cookieCodec|cookie\(|setCookie'
```

## Step 2 — Understand the defaults (this is the trap)

| Stack | Default cookie behavior | Sticky risk? |
|---|---|---|
| `RestTemplate` + Apache HttpClient 4 on classpath | `HttpClientBuilder.create()` installs a `BasicCookieStore` and processes `Set-Cookie` | **YES** |
| `RestTemplate` + Apache HttpClient 5 on classpath | `HttpClients.custom()`/`createDefault()` installs a default cookie store | **YES** |
| `RestTemplate` with no HttpClient (uses `SimpleClientHttpRequestFactory` / JDK `HttpURLConnection`) | No cookie store | No |
| `RestClient` (Spring 6.1+) with Apache HttpClient on classpath | Same as RestTemplate — uses `HttpComponentsClientHttpRequestFactory` | **YES** |
| `RestClient` with JDK `HttpClient` (`JdkClientHttpRequestFactory`) | JDK `HttpClient` defaults to `CookieHandler.getDefault()` — usually `null`, but if anything in the JVM ever called `CookieHandler.setDefault(...)`, it leaks process-wide | Possible |
| `WebClient` + Reactor Netty (default) | No cookie store; cookies only sent if the app calls `.cookie(...)` per request | No, unless `.cookie(...)` is called or a custom `HttpClient` configured `cookies` |
| `WebClient` + Jetty `HttpClient` connector | Jetty `HttpClient` has a `CookieStore` by default (`InMemoryHttpCookieStore`) | **YES** |
| `WebClient` + JDK `HttpClient` connector | Same caveat as RestClient on JDK | Possible |
| OkHttp | `CookieJar.NO_COOKIES` by default | No, unless a `CookieJar` was configured |
| `@FeignClient` (default) | Depends on underlying client; with Apache HttpClient: **YES** |

Key insight: the *only* way Apache HttpClient (4 or 5) appears on a Spring Boot classpath silently is via a transitive
dependency. If it is present, `RestTemplateBuilder` and `RestClient.Builder` upgrade to
`HttpComponentsClientHttpRequestFactory` automatically — even if no code in this repo asked for it. **Presence on the
classpath is enough to enable sticky-cookie behavior.**

## Step 3 — Establish a test baseline

Before editing any code, run the project's full test suite and capture the result. This is the reference you will diff
against after the fix.

```bash
./mvnw clean test 2>&1 | tee /tmp/sticky-baseline.txt
# or: ./gradlew clean test 2>&1 | tee /tmp/sticky-baseline.txt
```

Record: total tests run, failures, errors, skips. If tests fail before the fix, capture *which* tests fail — those are
pre-existing and must remain the only failures after the fix.

## Step 4 — Apply the fix

Edit every `@Bean`-producing method (or builder call site) that creates a `RestTemplate` / `RestClient` / `WebClient` /
Feign `Client` so the underlying HTTP client cannot store cookies. Patterns below.

### A. RestTemplate + Apache HttpClient 4 (Spring 5.x and earlier only)

> **Spring 6.x / Spring Boot 3.x warning:** Spring Framework 6.0 rewrote
`org.springframework.http.client.HttpComponentsClientHttpRequestFactory` to be HC5-only. On Spring 6+, this pattern will
not compile because the factory no longer accepts an HC4 `CloseableHttpClient`. If the project is on Spring 6+ and only
HC4 is on the classpath, add `org.apache.httpcomponents.client5:httpclient5` as a direct dependency and use Pattern B
instead.

```java
import org.apache.http.impl.client.HttpClientBuilder;
import org.springframework.boot.web.client.RestTemplateBuilder;
import org.springframework.http.client.HttpComponentsClientHttpRequestFactory;
import org.springframework.web.client.RestTemplate;

@Bean
public RestTemplate restTemplate(RestTemplateBuilder builder) {
    HttpComponentsClientHttpRequestFactory factory =
            new HttpComponentsClientHttpRequestFactory(HttpClientBuilder.create().disableCookieManagement().build());
    return builder.requestFactory(() -> factory).build();
}
```

### B. RestTemplate + Apache HttpClient 5

```java
import org.apache.hc.client5.http.impl.classic.HttpClients;
import org.springframework.boot.web.client.RestTemplateBuilder;
import org.springframework.http.client.HttpComponentsClientHttpRequestFactory;
import org.springframework.web.client.RestTemplate;

@Bean
public RestTemplate restTemplate(RestTemplateBuilder builder) {
    HttpComponentsClientHttpRequestFactory factory =
            new HttpComponentsClientHttpRequestFactory(HttpClients.custom().disableCookieManagement().build());
    return builder.requestFactory(() -> factory).build();
}
```

### C. RestClient (Spring 6.1+) + Apache HttpClient

```java

@Bean
public RestClient restClient() {
    return RestClient.builder()
                     .requestFactory(new HttpComponentsClientHttpRequestFactory(HttpClients.custom()
                                                                                           .disableCookieManagement()
                                                                                           .build()))
                     .build();
}
```

If the codebase uses `RestClient.Builder` injection rather than building from scratch, set `.requestFactory(...)` on the
builder before calling `.build()`.

### D. WebClient + Reactor Netty (defensive — only needed if cookies were explicitly configured)

Reactor Netty does not store cookies by default. The *only* fix needed is to make sure no code path is enabling them:

```java
import reactor.netty.http.client.HttpClient;
import org.springframework.http.client.reactive.ReactorClientHttpConnector;

@Bean
public WebClient webClient() {
    HttpClient httpClient = HttpClient.create();   // do NOT call .cookie(...) or attach a cookie codec
    return WebClient.builder().clientConnector(new ReactorClientHttpConnector(httpClient)).build();
}
```

Audit existing code for `.cookie(...)`, `cookieCodec`, or any custom `cookieJar`/`cookieStore` and remove them unless
they are intentional per-request cookies (not LB cookies).

### E. WebClient + Jetty connector

```java
import org.eclipse.jetty.client.HttpClient;
import org.eclipse.jetty.http.HttpCookieStore;
import org.springframework.http.client.reactive.JettyClientHttpConnector;

@Bean
public WebClient webClient() {
    HttpClient jetty = new HttpClient();
    jetty.setHttpCookieStore(new HttpCookieStore.Empty());   // disables cookie storage
    return WebClient.builder().clientConnector(new JettyClientHttpConnector(jetty)).build();
}
```

### F. WebClient + JDK HttpClient connector

```java
import java.net.http.HttpClient;

import org.springframework.http.client.reactive.JdkClientHttpConnector;

@Bean
public WebClient webClient() {
    HttpClient jdk = HttpClient.newBuilder()
                               .cookieHandler(null)   // explicit; ensures process-wide CookieHandler.getDefault() is not used
                               .build();
    return WebClient.builder().clientConnector(new JdkClientHttpConnector(jdk)).build();
}
```

### G. OkHttp (used directly, or as Feign/Retrofit transport)

OkHttp defaults to `CookieJar.NO_COOKIES`. Only fix needed is to ensure nothing installs a different `CookieJar`:

```java
OkHttpClient client =
        new OkHttpClient.Builder().cookieJar(okhttp3.CookieJar.NO_COOKIES)   // explicit; defends against future changes
                                  .build();
```

### H. Feign

If `feign.httpclient.enabled=true` (Apache HttpClient 4) or `feign.httpclient.hc5.enabled=true` (HttpClient 5) — same
trap as RestTemplate. Provide a `feign.Client` bean built from a cookie-disabled HttpClient:

```java

@Bean
public feign.Client feignClient() {
    return new feign.httpclient.ApacheHttpClient(HttpClientBuilder.create().disableCookieManagement().build());
}
```

Or for HC5:

```java

@Bean
public feign.Client feignClient() {
    return new feign.hc5.ApacheHttp5Client(HttpClients.custom().disableCookieManagement().build());
}
```

## Step 5 — Things NOT to do

- **Do not** try to filter cookies with a `ClientHttpRequestInterceptor` or `ExchangeFilterFunction`. Interceptors run
  *after* the cookie store has already attached cookies to the request, and *before* the response cookie store has
  captured `Set-Cookie`. Stripping the `Cookie` header in an interceptor partially works on outbound but does not stop
  the store from filling up; `Set-Cookie` will still be captured and resent on any code path that bypasses the
  interceptor.
- **Do not** rely on `Connection: close` or short-lived connections. Cookies are stored at the `HttpClient` level, not
  the connection level — pooled or not, they get replayed.
- **Do not** add a per-call header like `Cookie: AWSALB=` to overwrite the cookie. The shared cookie store still leaks
  across services if the bean is shared.
- **Do not** create a fresh `RestTemplate` per call as a workaround. That solves stickiness by accident at the cost of
  dropping connection pooling.

## Step 6 — Verify

1. Re-run the full test suite with the same command used for the baseline in Step 3:

   ```bash
   ./mvnw clean test 2>&1 | tee /tmp/sticky-after.txt
   # or: ./gradlew clean test 2>&1 | tee /tmp/sticky-after.txt
   ```

   Compare against `/tmp/sticky-baseline.txt`. The post-fix totals (tests run, failures, errors, skips) **must match the
   baseline exactly**. Any new failure is regression caused by the fix and must be resolved before continuing — do not
   accept "looks similar". If a referenced class (e.g. `org.apache.hc.client5.http.impl.classic.HttpClients`) is
   reachable only as a transitive dependency, the build will fail; declare the artifact as a direct `<dependency>` in
   the affected module's `pom.xml` (or `build.gradle`) and re-run. Iterate until results match the baseline.
2. Start the app locally, call an endpoint that triggers an outbound HTTP request, and confirm via `tcpdump`, mitmproxy,
   or a debug interceptor that **no `Cookie:` header** is sent on the second outbound request after the first response
   sets `AWSALB` (or whatever cookie the LB uses).
3. In a deployed environment, confirm load is distributed across downstream pods (downstream metrics / access logs
   should show traffic spread across instances rather than one).

## Step 7 — Suggested commit message

```
disable cookie management on outbound HTTP clients

AWS ALB stickiness cookies (AWSALB/AWSALBCORS) returned by downstream
services were being captured and replayed by the shared HTTP client,
pinning all server-to-server traffic to a single backend instance.
Disable cookie storage on every outbound client bean so each request is
load-balanced independently.
```
