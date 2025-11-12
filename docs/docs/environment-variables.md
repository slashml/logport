# Environment Variables

Logport supports environment variables for configuring default watch parameters. This is particularly useful for containerized deployments where configuration through environment variables is preferred over command-line arguments or settings.

## Supported Environment Variables

### LOGPORT_BROKERS

Set the default brokers list for watches.

```bash
export LOGPORT_BROKERS="kafka1:9092,kafka2:9092,kafka3:9092"
```

**Priority**: Environment variable > `default.brokers` setting > `localhost:9092`

**Examples**:
```bash
# Kafka brokers
export LOGPORT_BROKERS="kafka1:9092,kafka2:9092"

# HTTP endpoint
export LOGPORT_BROKERS="https://logs.example.com/ingest"

# Multiple HTTP endpoints
export LOGPORT_BROKERS="https://logs1.example.com,https://logs2.example.com"
```

### LOGPORT_TOPIC

Set the default Kafka topic for watches.

```bash
export LOGPORT_TOPIC="my_application_logs"
```

**Priority**: Environment variable > `default.topic` setting > `logport_logs`

**Note**: This is only used for Kafka producers. HTTP producers use the URL path instead.

**Examples**:
```bash
export LOGPORT_TOPIC="system_logs"
export LOGPORT_TOPIC="application_events"
```

### LOGPORT_PRODUCT_CODE

Set the default product code for watches. Product codes are identifiers used to tag logs for organizational purposes.

```bash
export LOGPORT_PRODUCT_CODE="prd4096"
```

**Priority**: Environment variable > `default.product_code` setting > empty string

**Examples**:
```bash
export LOGPORT_PRODUCT_CODE="prd1024"  # Application A
export LOGPORT_PRODUCT_CODE="prd2048"  # Application B
export LOGPORT_PRODUCT_CODE="prd4096"  # Infrastructure
```

### LOGPORT_LOG_TYPE

Set the default log type for watches. Log type is a user-defined category for organizing logs.

```bash
export LOGPORT_LOG_TYPE="system"
```

**Priority**: Environment variable > `default.log_type` setting > empty string

**Examples**:
```bash
export LOGPORT_LOG_TYPE="system"       # System logs
export LOGPORT_LOG_TYPE="application"  # Application logs
export LOGPORT_LOG_TYPE="security"     # Security logs
export LOGPORT_LOG_TYPE="audit"        # Audit logs
export LOGPORT_LOG_TYPE="access"       # Access logs
```

### LOGPORT_HOSTNAME

Set the default hostname that appears in log entries.

```bash
export LOGPORT_HOSTNAME="web-server-01.example.com"
```

**Priority**: Environment variable > `default.hostname` setting > system hostname

**Examples**:
```bash
export LOGPORT_HOSTNAME="$(hostname)"           # Use system hostname
export LOGPORT_HOSTNAME="web-01.production"    # Custom hostname
export LOGPORT_HOSTNAME="$KUBERNETES_POD_NAME" # Kubernetes pod name
```

## Priority Order

When determining configuration values, Logport uses the following priority order (highest to lowest):

1. **Command-line arguments**: `--brokers`, `--topic`, etc.
2. **Environment variables**: `LOGPORT_BROKERS`, `LOGPORT_TOPIC`, etc.
3. **Settings**: `default.brokers`, `default.topic`, etc.
4. **Built-in defaults**: Hard-coded defaults

### Example

```bash
# Set environment variable
export LOGPORT_TOPIC="env_logs"

# Set setting
logport set default.topic setting_logs

# Add watch with command-line argument
logport watch --topic cli_logs /var/log/syslog
# Result: Uses "cli_logs" (command-line wins)

# Add watch without command-line argument
logport watch /var/log/auth.log
# Result: Uses "env_logs" (environment variable beats setting)
```

## Kubernetes / Docker Usage

Environment variables are ideal for containerized deployments:

### Docker

```dockerfile
FROM ubuntu:latest

# Install logport
COPY logport /usr/local/bin/logport
RUN logport install

# Configure via environment variables
ENV LOGPORT_BROKERS=kafka:9092
ENV LOGPORT_TOPIC=container_logs
ENV LOGPORT_PRODUCT_CODE=prd4096
ENV LOGPORT_LOG_TYPE=application
ENV LOGPORT_HOSTNAME=my-container

ENTRYPOINT ["logport", "adopt"]
CMD ["/usr/local/bin/my-app"]
```

### Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-application
spec:
  template:
    spec:
      containers:
      - name: app
        image: my-app:latest
        env:
        - name: LOGPORT_BROKERS
          value: "kafka.default.svc.cluster.local:9092"
        - name: LOGPORT_TOPIC
          value: "kubernetes_logs"
        - name: LOGPORT_PRODUCT_CODE
          value: "prd2048"
        - name: LOGPORT_LOG_TYPE
          value: "application"
        - name: LOGPORT_HOSTNAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
```

### Kubernetes ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: logport-config
data:
  LOGPORT_BROKERS: "kafka1:9092,kafka2:9092,kafka3:9092"
  LOGPORT_TOPIC: "kubernetes_logs"
  LOGPORT_PRODUCT_CODE: "prd4096"
  LOGPORT_LOG_TYPE: "container"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-application
spec:
  template:
    spec:
      containers:
      - name: app
        image: my-app:latest
        envFrom:
        - configMapRef:
            name: logport-config
        env:
        - name: LOGPORT_HOSTNAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
```

## Best Practices

### 1. Use Environment Variables for Container Deployments

Environment variables are the preferred method for configuring logport in containerized environments.

```dockerfile
# Good: Use environment variables in Dockerfile
ENV LOGPORT_BROKERS=kafka:9092
ENV LOGPORT_TOPIC=app_logs

# Avoid: Using settings in containers (requires database access)
# RUN logport set default.brokers kafka:9092
```

### 2. Override at Different Levels

Use the priority system to your advantage:

```yaml
# Set common defaults in ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: logport-defaults
data:
  LOGPORT_BROKERS: "kafka:9092"
  LOGPORT_PRODUCT_CODE: "prd4096"
---
# Override per deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: special-app
spec:
  template:
    spec:
      containers:
      - name: app
        envFrom:
        - configMapRef:
            name: logport-defaults
        env:
        - name: LOGPORT_TOPIC
          value: "special_logs"  # Override just the topic
```

### 3. Use Kubernetes Field References

Leverage Kubernetes downward API for dynamic values:

```yaml
env:
- name: LOGPORT_HOSTNAME
  valueFrom:
    fieldRef:
      fieldPath: metadata.name  # Pod name

- name: LOGPORT_LOG_TYPE
  valueFrom:
    fieldRef:
      fieldPath: metadata.namespace  # Namespace
```

### 4. Validate Required Variables

Ensure critical environment variables are set:

```bash
#!/bin/bash
if [ -z "$LOGPORT_BROKERS" ]; then
  echo "Error: LOGPORT_BROKERS must be set"
  exit 1
fi

if [ -z "$LOGPORT_TOPIC" ]; then
  echo "Error: LOGPORT_TOPIC must be set"
  exit 1
fi

# Start application with logport
logport adopt /usr/local/bin/my-app
```

## Settings vs Environment Variables

### When to Use Settings

- On-premises installations
- Long-lived servers
- When configuration rarely changes
- When you want persistent configuration

```bash
logport set default.brokers kafka1:9092,kafka2:9092
logport set default.topic my_logs
logport set default.product_code prd4096
```

### When to Use Environment Variables

- Container deployments
- Kubernetes/Docker environments
- CI/CD pipelines
- When configuration differs per environment
- When you need dynamic configuration

```bash
export LOGPORT_BROKERS=kafka1:9092,kafka2:9092
export LOGPORT_TOPIC=my_logs
export LOGPORT_PRODUCT_CODE=prd4096
```

## Troubleshooting

### Check Current Configuration

To see which values are being used:

```bash
# Check environment variables
env | grep LOGPORT_

# Check settings
logport settings

# Test with verbose output
logport watch --brokers test:9092 --topic test /tmp/test.log
# Look at the output to see which broker/topic is used
```

### Environment Variable Not Taking Effect

1. Verify the variable is exported:
   ```bash
   echo $LOGPORT_BROKERS
   ```

2. Check if a setting is overriding it:
   ```bash
   logport settings | grep default.brokers
   ```

3. Verify the variable is available to logport:
   ```bash
   sudo -E logport watch /var/log/syslog
   # The -E flag preserves environment variables with sudo
   ```

### Docker Environment Variables Not Working

Ensure variables are set before the ENTRYPOINT:

```dockerfile
# Good
ENV LOGPORT_BROKERS=kafka:9092
ENTRYPOINT ["logport", "adopt"]

# Bad - environment won't be available
ENTRYPOINT ["logport", "adopt"]
ENV LOGPORT_BROKERS=kafka:9092
```
