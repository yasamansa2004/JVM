## Possible Causes of cgroup Detection Failure

JVM may fail to initialize container metrics when it cannot correctly detect or access the Linux cgroup configuration.

Common causes include:

### 1. Old JVM with cgroup v2 incompatibility

Older OpenJDK versions may have bugs or incomplete support for cgroup v2.

Typical error:

```text
java.lang.NullPointerException:
Cannot invoke "jdk.internal.platform.CgroupInfo.getMountPoint()"
because "anyController" is null
```

The failure occurs in:

```text
jdk.internal.platform.cgroupv2.CgroupV2Subsystem
```

### 2. Missing cgroup controllers

Required controllers such as:

```text
cpu
memory
cpuset
io
pids
```

may not be available or enabled.

Check:

```bash
cat /sys/fs/cgroup/cgroup.controllers
```

### 3. Incorrect or incomplete cgroup mount

Check:

```bash
mount | grep cgroup
```

For cgroup v2, the expected filesystem is:

```text
cgroup2 on /sys/fs/cgroup type cgroup2
```

Also check:

```bash
ls -la /sys/fs/cgroup/
```

### 4. cgroup namespace configuration

Docker can expose a different cgroup namespace to the container.

Check:

```bash
docker inspect <container> \
  --format '{{.HostConfig.CgroupnsMode}}'
```

For example:

```text
host
```

or:

```text
private
```

An unusual namespace configuration can expose cgroup information differently than expected by an older JVM.

### 5. Host/Container cgroup version mismatch

The host may use cgroup v2 while the application/JVM was designed or tested primarily with cgroup v1.

Check the host:

```bash
stat -fc %T /sys/fs/cgroup/
```

Result:

```text
cgroup2fs
```

indicates cgroup v2.

### 6. Missing or inaccessible cgroup information inside the container

Check from inside the container:

```bash
docker exec <container> sh -c '
mount | grep cgroup
ls -la /sys/fs/cgroup/
cat /sys/fs/cgroup/cgroup.controllers 2>/dev/null
'
```

If the JVM cannot access the expected cgroup information, container metrics initialization may fail.

### 7. JVM container-awareness bug

The failure can occur before the application itself starts, when Java initializes:

```text
OperatingSystemMXBean
        ↓
Container.metrics()
        ↓
CgroupV2Subsystem
```

In this situation, Spring Boot/Tomcat is only exposing the JVM problem through Micrometer.

## Workaround

When upgrading the JVM is not immediately possible:

```yaml
environment:
  JAVA_OPTS: "-XX:-UseContainerSupport"
```

This disables JVM container-awareness and bypasses the problematic cgroup detection code.

Docker CPU/memory limits remain enforced by Docker itself.

For example:

```yaml
environment:
  JAVA_OPTS: "-Xms5g -Xmx5g -XX:-UseContainerSupport"
```
