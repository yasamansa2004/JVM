# JVM cgroup v2 Workaround

## Problem

In some Docker environments using **cgroup v2**, the JVM may fail while detecting container CPU/memory information.

The application can fail during Spring Boot startup with errors similar to:

```text
java.lang.InternalError: java.lang.reflect.InvocationTargetException
```

and:

```text
java.lang.NullPointerException:
Cannot invoke "jdk.internal.platform.CgroupInfo.getMountPoint()"
because "anyController" is null
```

The error is usually triggered while Spring Boot Actuator initializes system or Tomcat metrics:

```text
ProcessorMetrics
    ↓
OperatingSystemMXBean
    ↓
Container.metrics()
    ↓
CgroupV2Subsystem
    ↓
NullPointerException
```

## Configuration

The following configuration can be used as a workaround:

```yaml
environment:
  JAVA_OPTS: "-XX:-UseContainerSupport"
  SPRING_AUTOCONFIGURE_EXCLUDE: org.springframework.boot.actuate.autoconfigure.metrics.SystemMetricsAutoConfiguration
  MANAGEMENT_METRICS_BINDERS_TOMCAT_ENABLED: "false"
```

### JAVA_OPTS

```yaml
JAVA_OPTS: "-XX:-UseContainerSupport"
```

Disables JVM container-awareness.

This prevents the affected JVM from trying to read Docker/cgroup information through the problematic cgroup v2 implementation.

> This does **not** disable Docker CPU or memory limits. Docker continues to enforce the container limits.

If the application already has JVM options, append the flag instead of replacing the existing options. For example:

```yaml
JAVA_OPTS: "-Xms5g -Xmx5g -XX:-UseContainerSupport"
```

### SPRING_AUTOCONFIGURE_EXCLUDE

```yaml
SPRING_AUTOCONFIGURE_EXCLUDE: org.springframework.boot.actuate.autoconfigure.metrics.SystemMetricsAutoConfiguration
```

Prevents Spring Boot Actuator from automatically configuring the system metrics components, including `ProcessorMetrics`.

### MANAGEMENT_METRICS_BINDERS_TOMCAT_ENABLED

```yaml
MANAGEMENT_METRICS_BINDERS_TOMCAT_ENABLED: "false"
```

Disables the Tomcat metrics binder.

This prevents Micrometer from registering Tomcat-specific metrics.

## Recommended Configuration

For an application affected by the JVM/cgroup v2 issue:

```yaml
environment:
  JAVA_OPTS: "-Xms5g -Xmx5g -XX:-UseContainerSupport"
  SPRING_AUTOCONFIGURE_EXCLUDE: org.springframework.boot.actuate.autoconfigure.metrics.SystemMetricsAutoConfiguration
  MANAGEMENT_METRICS_BINDERS_TOMCAT_ENABLED: "false"
```

## Verification

After updating the Compose file, recreate the container:

```bash
docker-compose up -d --force-recreate <service-name>
```

Verify that the JVM option is applied:

```bash
docker exec <container-name> sh -c 'ps aux | grep "[j]ava"'
```

The Java command should contain:

```text
-XX:-UseContainerSupport
```

Check application logs:

```bash
docker logs --tail 200 <container-name>
```

The application should start without the `CgroupV2Subsystem` / `ProcessorMetrics` initialization error.

## Important

These settings are a **workaround** for JVM/cgroup compatibility issues.

They should not be considered a replacement for upgrading an affected JVM when a compatible Java version becomes available.
