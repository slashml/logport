# Kubernetes Deployment

This guide covers deploying applications with logport in Kubernetes environments.

## Overview

Logport in Kubernetes captures application logs (stdout/stderr) and forwards them to Kafka or HTTP endpoints. The recommended pattern is to use `logport adopt` as the container ENTRYPOINT.

## Quick Start

### Using Environment Variables

The simplest approach uses environment variables for configuration:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: myapp:latest
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

## Dockerfile Configuration

### Basic Dockerfile

```dockerfile
FROM ubuntu:latest

# Install logport
RUN mkdir -p /usr/local/lib/logport/install
COPY build/logport /usr/local/lib/logport/install/logport
COPY build/librdkafka.so.1 /usr/local/lib/logport/install/librdkafka.so.1

WORKDIR /usr/local/lib/logport/install
RUN /usr/local/lib/logport/install/logport install

# Set default configuration (can be overridden by K8s env vars)
ENV LOGPORT_BROKERS=kafka:9092
ENV LOGPORT_TOPIC=container_logs
ENV LOGPORT_PRODUCT_CODE=prd4096
ENV LOGPORT_HOSTNAME=my-container

# Use logport adopt to capture app output
ENTRYPOINT ["logport", "adopt"]
CMD ["/usr/local/bin/my-app"]
```

### Multi-stage Build

```dockerfile
# Build stage
FROM ubuntu:latest AS builder

RUN apt-get update && apt-get install -y \
    cmake g++ git wget \
    librdkafka-dev libssl-dev libz-dev libpthread-stubs0-dev

WORKDIR /build
RUN git clone https://github.com/homer6/logport.git
WORKDIR /build/logport
RUN cmake . && make

# Runtime stage
FROM ubuntu:latest

# Install runtime dependencies
RUN apt-get update && apt-get install -y \
    librdkafka1 libssl1.1 libz1 && \
    rm -rf /var/lib/apt/lists/*

# Copy logport
COPY --from=builder /build/logport/build/logport /usr/local/bin/
COPY --from=builder /build/logport/build/librdkafka.so.1 /usr/local/lib/logport/

# Install logport
RUN logport install

# Copy application
COPY my-app /usr/local/bin/my-app

# Default configuration
ENV LOGPORT_BROKERS=kafka:9092
ENV LOGPORT_TOPIC=logs

# Use logport to wrap application
ENTRYPOINT ["logport", "adopt"]
CMD ["/usr/local/bin/my-app"]
```

## Kubernetes Configuration Patterns

### Using ConfigMap for Shared Configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: logport-config
  namespace: default
data:
  LOGPORT_BROKERS: "kafka-headless.kafka.svc.cluster.local:9092"
  LOGPORT_TOPIC: "kubernetes_logs"
  LOGPORT_PRODUCT_CODE: "prd4096"
  LOGPORT_LOG_TYPE: "application"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest
        envFrom:
        - configMapRef:
            name: logport-config
        env:
        - name: LOGPORT_HOSTNAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
```

### Per-Application Configuration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  template:
    spec:
      containers:
      - name: app
        image: frontend:latest
        env:
        - name: LOGPORT_BROKERS
          value: "kafka:9092"
        - name: LOGPORT_TOPIC
          value: "frontend_logs"
        - name: LOGPORT_PRODUCT_CODE
          value: "prd1024"
        - name: LOGPORT_LOG_TYPE
          value: "frontend"
        - name: LOGPORT_HOSTNAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  template:
    spec:
      containers:
      - name: app
        image: backend:latest
        env:
        - name: LOGPORT_BROKERS
          value: "kafka:9092"
        - name: LOGPORT_TOPIC
          value: "backend_logs"
        - name: LOGPORT_PRODUCT_CODE
          value: "prd2048"
        - name: LOGPORT_LOG_TYPE
          value: "backend"
        - name: LOGPORT_HOSTNAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
```

### Using HTTP Producer

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: logport-http-credentials
type: Opaque
stringData:
  username: admin
  password: secret123
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest
        env:
        - name: HTTP_USERNAME
          valueFrom:
            secretKeyRef:
              name: logport-http-credentials
              key: username
        - name: HTTP_PASSWORD
          valueFrom:
            secretKeyRef:
              name: logport-http-credentials
              key: password
        - name: LOGPORT_BROKERS
          value: "https://$(HTTP_USERNAME):$(HTTP_PASSWORD)@logs.example.com/ingest"
        - name: LOGPORT_PRODUCT_CODE
          value: "prd4096"
        - name: LOGPORT_HOSTNAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
```

### Multi-Container Pod

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      # Main application
      - name: app
        image: myapp:latest
        env:
        - name: LOGPORT_BROKERS
          value: "kafka:9092"
        - name: LOGPORT_TOPIC
          value: "app_logs"
        - name: LOGPORT_PRODUCT_CODE
          value: "prd2048"
        - name: LOGPORT_LOG_TYPE
          value: "application"
        - name: LOGPORT_HOSTNAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name

      # Sidecar service
      - name: sidecar
        image: sidecar:latest
        env:
        - name: LOGPORT_BROKERS
          value: "kafka:9092"
        - name: LOGPORT_TOPIC
          value: "sidecar_logs"
        - name: LOGPORT_PRODUCT_CODE
          value: "prd2048"
        - name: LOGPORT_LOG_TYPE
          value: "sidecar"
        - name: LOGPORT_HOSTNAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
```

## Using Kubernetes Downward API

Inject pod/container metadata into environment variables:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest
        env:
        # Use pod name as hostname
        - name: LOGPORT_HOSTNAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name

        # Use namespace as log type
        - name: LOGPORT_LOG_TYPE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace

        # Include node name in metadata
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName

        # Include pod IP
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP

        # Standard logport config
        - name: LOGPORT_BROKERS
          value: "kafka:9092"
        - name: LOGPORT_TOPIC
          value: "k8s_logs"
        - name: LOGPORT_PRODUCT_CODE
          value: "prd4096"
```

## StatefulSet Example

For StatefulSets, use the pod name to maintain consistent identity:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: myapp
spec:
  serviceName: myapp
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: myapp:latest
        env:
        - name: LOGPORT_BROKERS
          value: "kafka:9092"
        - name: LOGPORT_TOPIC
          value: "statefulset_logs"
        - name: LOGPORT_PRODUCT_CODE
          value: "prd4096"
        - name: LOGPORT_HOSTNAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name  # Will be myapp-0, myapp-1, myapp-2
```

## DaemonSet for Node Logs

Deploy logport as a DaemonSet to collect logs from all nodes:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: logport-node-logs
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: logport-node-logs
  template:
    metadata:
      labels:
        name: logport-node-logs
    spec:
      hostNetwork: true
      hostPID: true
      containers:
      - name: logport
        image: logport:latest
        securityContext:
          privileged: true
        env:
        - name: LOGPORT_BROKERS
          value: "kafka.kafka.svc.cluster.local:9092"
        - name: LOGPORT_TOPIC
          value: "node_logs"
        - name: LOGPORT_PRODUCT_CODE
          value: "prd8192"
        - name: LOGPORT_LOG_TYPE
          value: "node"
        - name: LOGPORT_HOSTNAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        volumeMounts:
        - name: varlog
          mountPath: /var/log
          readOnly: true
        command:
        - /bin/sh
        - -c
        - |
          logport watch /var/log/syslog
          logport watch /var/log/kern.log
          logport watch /var/log/auth.log
          logport start
          tail -f /usr/local/logport/logport.log
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
```

## CronJob Example

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-report
spec:
  schedule: "0 2 * * *"  # 2 AM daily
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: report
            image: report-generator:latest
            env:
            - name: LOGPORT_BROKERS
              value: "kafka:9092"
            - name: LOGPORT_TOPIC
              value: "cronjob_logs"
            - name: LOGPORT_PRODUCT_CODE
              value: "prd4096"
            - name: LOGPORT_LOG_TYPE
              value: "cronjob"
            - name: LOGPORT_HOSTNAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
          restartPolicy: OnFailure
```

## Kafka Deployment in Kubernetes

Example Kafka deployment for use with logport:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kafka-headless
  namespace: kafka
spec:
  clusterIP: None
  selector:
    app: kafka
  ports:
  - port: 9092
    name: kafka
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kafka
  namespace: kafka
spec:
  serviceName: kafka-headless
  replicas: 3
  selector:
    matchLabels:
      app: kafka
  template:
    metadata:
      labels:
        app: kafka
    spec:
      containers:
      - name: kafka
        image: confluentinc/cp-kafka:latest
        ports:
        - containerPort: 9092
        env:
        - name: KAFKA_ZOOKEEPER_CONNECT
          value: "zookeeper:2181"
        - name: KAFKA_ADVERTISED_LISTENERS
          value: "PLAINTEXT://kafka-headless:9092"
```

Use in logport: `LOGPORT_BROKERS=kafka-headless.kafka.svc.cluster.local:9092`

## Best Practices

### 1. Use Pod Name as Hostname

```yaml
- name: LOGPORT_HOSTNAME
  valueFrom:
    fieldRef:
      fieldPath: metadata.name
```

This makes it easy to identify which pod generated each log message.

### 2. Use ConfigMaps for Shared Config

```yaml
envFrom:
- configMapRef:
    name: logport-config
```

This ensures consistent configuration across deployments.

### 3. Use Secrets for Credentials

```yaml
- name: LOGPORT_BROKERS
  value: "https://$(USERNAME):$(PASSWORD)@logs.example.com"
env:
- name: USERNAME
  valueFrom:
    secretKeyRef:
      name: http-creds
      key: username
```

### 4. Use Appropriate Product Codes

Assign unique product codes per application:
```yaml
frontend: prd1024
backend: prd2048
database: prd4096
infrastructure: prd8192
```

### 5. Set Resource Limits

```yaml
containers:
- name: app
  resources:
    requests:
      memory: "64Mi"
      cpu: "100m"
    limits:
      memory: "256Mi"
      cpu: "500m"
```

Logport itself uses minimal resources (~1.8MB), but set limits based on your application.

### 6. Use Health Checks

```yaml
containers:
- name: app
  livenessProbe:
    exec:
      command:
      - /bin/sh
      - -c
      - "ps aux | grep -v grep | grep my-app"
    initialDelaySeconds: 10
    periodSeconds: 30
```

### 7. Configure HTTP Producer Settings

For HTTP endpoints, tune batch size:

```yaml
- name: app
  command:
  - /bin/sh
  - -c
  - |
    logport set http.producer.batch.num.messages 500
    exec logport adopt /usr/local/bin/my-app
```

## Troubleshooting

### Pod Logs Show Logport Errors

View logs:
```bash
kubectl logs <pod-name>
```

Common issues:
- **Connection refused**: Kafka not accessible
- **Timeout**: Increase `http.producer.message.timeout.ms`
- **Authorization failed**: Check credentials

### Logport Not Capturing Logs

Verify ENTRYPOINT:
```bash
kubectl describe pod <pod-name>
```

Should show:
```
Command:
  logport
  adopt
Args:
  /usr/local/bin/my-app
```

### Messages Not Appearing in Kafka

Check from within pod:
```bash
kubectl exec -it <pod-name> -- /bin/bash
kafkacat -L -b kafka:9092
kafkacat -C -b kafka:9092 -t kubernetes_logs -o -10
```

### Environment Variables Not Set

Check env vars:
```bash
kubectl exec -it <pod-name> -- env | grep LOGPORT_
```

## Example: Complete Application

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: logport-config
  namespace: myapp
data:
  LOGPORT_BROKERS: "kafka.kafka.svc.cluster.local:9092"
  LOGPORT_TOPIC: "myapp_logs"
  LOGPORT_PRODUCT_CODE: "prd2048"
  LOGPORT_LOG_TYPE: "application"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: myapp:1.0.0
        envFrom:
        - configMapRef:
            name: logport-config
        env:
        - name: LOGPORT_HOSTNAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        resources:
          requests:
            memory: "128Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: myapp
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer
```

Deploy:
```bash
kubectl apply -f myapp.yaml
kubectl -n myapp get pods
kubectl -n myapp logs -f <pod-name>
```

## Next Steps

- Review [Environment Variables](environment-variables.md) for all configuration options
- See [Message Format](message-format.md) to understand output structure
- Check [Producers](producers.md) for Kafka vs HTTP configuration
- Read [Advanced Topics](advanced.md) for performance tuning
