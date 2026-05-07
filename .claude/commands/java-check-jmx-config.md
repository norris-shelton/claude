---
description: Check JMX configuration and DataSource setup, auto-fixing any missing properties
argument-hint: [project-path]
allowed-tools: [Bash, Read, Edit, Write, Glob, Grep]
---

# Check JMX Configuration

Verify this project's JMX configuration and DataSource wrapper class naming convention. Automatically fix any missing
JMX properties.

## Steps

### 1. Find the DataSource wrapper class

Search the project's main Java source directory for any class that extends `org.apache.tomcat.jdbc.pool.DataSource`.
Report the class name and file path found.

The class must be named exactly `DataSource`. Any other name (e.g., `JmxDataSource`, `TrDataSource`,
`WrappedDataSource`) is a violation of the convention.

### 1b. Check the @ManagedResource annotation on the DataSource wrapper class

The `@ManagedResource` annotation must **not** include an `objectName` attribute. Spring JMX derives the JMX domain from
the class's Java package, so the class must be in the correct package (see step 1c). Hardcoding `objectName` causes all
instances of the class to collide under a single fixed name when multiple beans exist.

- PASS if `@ManagedResource` has **no** `objectName` attribute
- FAIL if `objectName` is present — remove it, leaving only the `description`

### 1c. Check the package of the DataSource wrapper class

The `DataSource` class must be in the **root application package** (`com.twinspires.<appname>`), not a sub-package.
Spring JMX derives the domain from the class's Java package. If the class is in a sub-package (e.g.,
`com.twinspires.endofday.spring.jmx`), the DataSource will appear under that sub-package in JConsole instead of the
correct top-level domain.

- PASS if the class package matches `com.twinspires.<appname>` exactly
- FAIL if the class is in any sub-package — use `git mv` to move the file to the root package (preserves git history),
  update the `package` declaration, and update all imports

### 1d. Check for duplicate DataSource bean registrations

Search all `@Configuration` classes for any `@Bean` method that returns a `DataSource` instance by delegating to another
existing DataSource bean (e.g., a wrapper method created solely to satisfy a framework naming requirement like Spring
Batch's `dataSource`). These cause the same object to appear twice in JConsole under different names.

The correct pattern is to add the required alias name directly to the original `@Bean` declaration using a name array,
with the descriptive name first:

```java
// CORRECT — one JMX entry, named "rwWinticketDataSource"; "dataSource" is an alias only
@Bean({"rwWinticketDataSource", "dataSource"})
public DataSource rwWinticketDataSource() { /*...*/ }

// WRONG — creates a second JMX entry named "dataSource"
@Bean("dataSource")
public DataSource dataSource() {return rwWinticketDataSource;}
```

- PASS if no such wrapper/delegate `@Bean` methods exist
- FAIL if found — add the alias name to the original `@Bean` array, then remove the wrapper method, its field injection,
  and any now-unused imports

### 2. Find and check application.properties for JMX properties

Search the project for `application.properties` files under `src/main/resources`. Check each one for the following
required JMX properties:

| Property | Required Value |
| --- | --- |
| `spring.jmx.enabled` | `true` |
| `spring.jmx.unique-names` | `true` |
| `spring.jmx.default-domain` | `com.twinspires.<appname>` (see below) |
| `management.metrics.enable.jdbc` | `true` |
| `management.endpoints.jmx.exposure.include` | `health,metrics,info` |
| `server.tomcat.mbeanregistry.enabled` | `true` |

**Determining `spring.jmx.default-domain`:** Derive the app name from the base Java package under
`src/main/java/com/twinspires/`. The next path segment after `twinspires` is the app name (e.g.,
`com/twinspires/endofday` → `endofday`). The required value is `com.twinspires.<appname>`.

### 3. Fix any missing JMX properties

For each `application.properties` file that is missing one or more required JMX properties, add a `# JMX configuration`
comment block containing all missing properties immediately before the first `spring.datasource` or `spring.batch`
property line. If neither exists, append the block at the end of the file.

Use these exact values when adding missing properties:

- `spring.jmx.enabled=true`
- `spring.jmx.unique-names=true`
- `spring.jmx.default-domain=com.twinspires.<appname>` (derived as described above)
- `management.metrics.enable.jdbc=true`
- `management.endpoints.jmx.exposure.include=health,metrics,info`
- `server.tomcat.mbeanregistry.enabled=true`

Only add properties that are missing; do not duplicate existing ones.

### 4. Check for JmxTest class

**Prerequisite:** Only perform this step if a DataSource wrapper class (extending
`org.apache.tomcat.jdbc.pool.DataSource`) was found in step 1. If no JDBC DataSource exists in the project, skip this
step entirely and report **N/A — no JDBC DataSource in project**.

Search `src/test/java` for a class named `JmxTest`. The domain used in its `ObjectName` pattern must match
`spring.jmx.default-domain` (i.e., `com.twinspires.<appname>`), derived the same way as in step 2.

The required test must:

- Be annotated `@SpringBootTest(properties = "spring.jmx.enabled=true")`
- Query MBeans using the pattern `com.twinspires.<appname>:type=DataSource,*`
- Assert the result set is not empty
- For each matched MBean, read and assert non-zero values for: `MaxActive`, `MaxIdle`, `MaxWait`, `InitialSize`

If missing, create `JmxTest.java` in the root test package (`src/test/java/com/twinspires/<appname>/`) with the
following template (substituting `<appname>` and the full package path):

```java
package com.twinspires.<appname>;

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
 * @author <author>
 */
@SpringBootTest(properties = "spring.jmx.enabled=true")
class JmxTest {

    private final Logger logger = LoggerFactory.getLogger(getClass());

    @Autowired
    private javax.management.MBeanServer mBeanServer;

    @Test
    void testDataSourcesRegisteredUnderCorrectDomain() {
        try {
            ObjectName pattern = new ObjectName("com.twinspires.<appname>:type=DataSource,*");
            Set<ObjectName> names = mBeanServer.queryNames(pattern, null);
            assertFalse(names.isEmpty(),
                "Expected at least one DataSource MBean under com.twinspires.<appname>:type=DataSource,*");

            for (ObjectName name : names) {
                int maxActive   = (int) mBeanServer.getAttribute(name, "MaxActive");
                int maxIdle     = (int) mBeanServer.getAttribute(name, "MaxIdle");
                int maxWait     = (int) mBeanServer.getAttribute(name, "MaxWait");
                int initialSize = (int) mBeanServer.getAttribute(name, "InitialSize");

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

If the file exists but uses a wrong domain in the `ObjectName` pattern, update it to
`com.twinspires.<appname>:type=DataSource,*`.

- PASS if `JmxTest.java` exists with the correct domain in the pattern
- FAIL if missing or using wrong domain — create or fix it

## Output Format

**DataSource Wrapper Class**

- The class name and file path found
- PASS if named exactly `DataSource`
- FAIL with the actual name and the recommended rename to `DataSource` if named otherwise

**@ManagedResource annotation**

- PASS if no `objectName` attribute is present
- FAIL if `objectName` is present — remove it and confirm

**DataSource package**

- PASS if in root application package
- FAIL if in sub-package — confirm moved with `git mv`

**Duplicate DataSource bean registrations**

- PASS if none found
- FAIL if found — confirm fixed

**JMX Properties** (per properties file found)

- File path
- Each required property with PASS or FAIL and the actual value found (or MISSING)
- If any were missing, confirm they have been added

**JmxTest**

- N/A if no JDBC DataSource wrapper class exists in the project (skip entirely)
- PASS if exists with correct domain pattern
- FAIL if missing or wrong domain — confirm created or fixed

End with a summary of total passing and failing checks (before fixes) and a list of any changes made.
