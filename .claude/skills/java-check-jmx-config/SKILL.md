---
description: Check, audit, or fix JMX configuration and DataSource MBean setup in a Spring Boot Java service. Auto-derives base package; fixes default-domain, @ManagedResource, package placement, duplicate beans, missing properties, and the JmxTest. Triggers: check/audit/configure JMX, fix JMX domain, register DataSource as MBean, DataSource not showing in JConsole, wrong JMX domain, MBean under sub-package, duplicate DataSource in JConsole, pool metrics missing, @ManagedResource annotation, spring.jmx.default-domain, server.tomcat.mbeanregistry.enabled, tomcat-jdbc pool, JmxTest.
argument-hint: [project-path]
allowed-tools: [Bash, Read, Edit, Write, Glob, Grep]
---

# Java: Check JMX Configuration

Verify this project's JMX configuration and DataSource wrapper class naming convention. Auto-fix missing JMX properties.

## 0. Auto-derive base package

Find the base Java package (the package of the `@SpringBootApplication` class) and use it as `<base.package>` everywhere below. Do **not** prompt the user.

```bash
rg -l --type java '@SpringBootApplication' src/main/java
# read the file's `package` declaration → <base.package>
# the JMX default-domain is identical to <base.package>
```

All `<base.package>` references below refer to that derived value.

## 1. DataSource wrapper class

### 1a. Find it

Search `src/main/java` for any class extending `org.apache.tomcat.jdbc.pool.DataSource`. Report class name and path.

- **PASS** if named exactly `DataSource`.
- **FAIL** otherwise (e.g., `JmxDataSource`, `TrDataSource`) — rename to `DataSource`.

### 1b. `@ManagedResource` annotation

Must **not** include `objectName`. Spring derives the JMX domain from the class's package; hardcoding `objectName` collides when multiple beans exist.

- **PASS** if no `objectName` attribute.
- **FAIL** if present — remove it; keep only `description`.

### 1c. Package placement

The `DataSource` class must live in `<base.package>` exactly, not a sub-package. If it's in a sub-package (e.g., `<base.package>.spring.jmx`), the DataSource appears under that sub-package in JConsole instead of the correct top-level domain.

- **PASS** if package equals `<base.package>`.
- **FAIL** otherwise — use `git mv` (preserves history), update the `package` declaration, and update all imports.

### 1d. Duplicate `@Bean` registrations

Find any `@Configuration` `@Bean` method that returns a `DataSource` by delegating to another existing `DataSource` bean (a wrapper created to satisfy a framework requirement like Spring Batch's `dataSource`). These create duplicate MBean entries.

Correct pattern — name array on the original `@Bean`, descriptive name first:

```java
// CORRECT — one MBean entry named "rwWinticketDataSource"; "dataSource" is an alias
@Bean({"rwWinticketDataSource", "dataSource"})
public DataSource rwWinticketDataSource() { /*...*/ }

// WRONG — creates a second MBean entry named "dataSource"
@Bean("dataSource")
public DataSource dataSource() { return rwWinticketDataSource; }
```

- **PASS** if no wrapper/delegate methods exist.
- **FAIL** otherwise — add the alias to the original `@Bean`'s name array; remove the wrapper method, its field injection, and any now-unused imports.

## 2. Properties

Search `src/main/resources` for `application.properties` files. Each must contain:

| Property | Required value |
|---|---|
| `spring.jmx.enabled` | `true` |
| `spring.jmx.unique-names` | `true` |
| `spring.jmx.default-domain` | `<base.package>` |
| `management.metrics.enable.jdbc` | `true` |
| `management.endpoints.jmx.exposure.include` | `health,metrics,info` |
| `server.tomcat.mbeanregistry.enabled` | `true` |

### Fix missing properties

For each file missing one or more, insert a `# JMX configuration` block containing only the missing properties immediately before the first `spring.datasource` or `spring.batch` line. If neither exists, append at end of file. Never duplicate existing properties.

## 3. JmxTest

**Prerequisite:** only run this check if step 1a found a `DataSource` wrapper class. If the project has no JDBC `DataSource`, report **N/A — no JDBC DataSource in project** and skip.

Look for `src/test/java/.../JmxTest.java`. The test must:

- Be annotated `@SpringBootTest(properties = "spring.jmx.enabled=true")`.
- Query MBeans with pattern `<base.package>:type=DataSource,*`.
- Assert the result set is not empty.
- For each matched MBean assert non-zero `MaxActive`, `MaxIdle`, `MaxWait`, `InitialSize`.

If missing, create `JmxTest.java` in `src/test/java/<base.package as path>/`:

```java
package <base.package>;

import javax.management.MalformedObjectNameException;
import javax.management.ObjectName;
import org.junit.jupiter.api.Test;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import java.util.Set;

import static org.junit.jupiter.api.Assertions.assertFalse;
import static org.junit.jupiter.api.Assertions.assertTrue;
import static org.junit.jupiter.api.Assertions.fail;

/**
 * Verifies that DataSource MBeans are registered under the correct JMX domain
 * and that pool configuration attributes are set to non-zero values.
 */
@SpringBootTest(properties = "spring.jmx.enabled=true")
class JmxTest {

    private final Logger logger = LoggerFactory.getLogger(getClass());

    @Autowired
    private javax.management.MBeanServer mBeanServer;

    @Test
    void testDataSourcesRegisteredUnderCorrectDomain() {
        try {
            ObjectName pattern = new ObjectName("<base.package>:type=DataSource,*");
            Set<ObjectName> names = mBeanServer.queryNames(pattern, null);
            assertFalse(names.isEmpty(),
                    "Expected at least one DataSource MBean under <base.package>:type=DataSource,*");

            for (ObjectName name : names) {
                // Pool implementations sometimes expose these as int, sometimes long.
                // Use Number to be resilient across Tomcat / HikariCP / DBCP2.
                long maxActive   = ((Number) mBeanServer.getAttribute(name, "MaxActive")).longValue();
                long maxIdle     = ((Number) mBeanServer.getAttribute(name, "MaxIdle")).longValue();
                long maxWait     = ((Number) mBeanServer.getAttribute(name, "MaxWait")).longValue();
                long initialSize = ((Number) mBeanServer.getAttribute(name, "InitialSize")).longValue();

                logger.info("{} - MaxActive={}, MaxIdle={}, MaxWait={}, InitialSize={}",
                        name, maxActive, maxIdle, maxWait, initialSize);

                assertTrue(maxActive > 0,   name + " MaxActive should be > 0, was "   + maxActive);
                assertTrue(maxIdle > 0,     name + " MaxIdle should be > 0, was "     + maxIdle);
                assertTrue(maxWait > 0,     name + " MaxWait should be > 0, was "     + maxWait);
                assertTrue(initialSize > 0, name + " InitialSize should be > 0, was " + initialSize);
            }
        } catch (MalformedObjectNameException e) {
            fail("Exception creating ObjectName:", e);
        } catch (Exception e) {
            fail("Exception reading MBean attributes:", e);
        }
    }
}
```

If the file exists but uses a wrong domain in its `ObjectName` pattern, update it to `<base.package>:type=DataSource,*`.

## 4. Verify

After applying all fixes:

```bash
# Maven
./mvnw -q test -Dtest=JmxTest
# Gradle
./gradlew test --tests JmxTest
```

If `JmxTest` is skipped because there's no DataSource, run a plain compile to confirm renames/moves didn't break anything:

```bash
./mvnw -q -DskipTests compile        # or: ./gradlew compileJava
```

## Report

End the run with:

- A summary of total PASS / FAIL checks (before fixes).
- A list of changes made (renames, moves, properties added, files created).
- The `JmxTest` result (pass / fail / skipped + reason).
