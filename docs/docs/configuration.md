# Configuration

Logport can be configured through settings, environment variables, and command-line arguments. This document covers all configuration methods and options.

## Configuration Methods

### 1. Settings (Persistent)

Settings are stored in the SQLite database and persist across restarts.

```bash
# Set a setting
logport set <key> <value>

# View all settings
logport settings

# Remove a setting
logport unset <key>
```

### 2. Environment Variables

Environment variables override settings but are overridden by command-line arguments.

```bash
export LOGPORT_BROKERS="kafka:9092"
export LOGPORT_TOPIC="my_logs"
export LOGPORT_PRODUCT_CODE="prd4096"
export LOGPORT_LOG_TYPE="system"
export LOGPORT_HOSTNAME="my-server"
```

See [Environment Variables](environment-variables.md) for details.

### 3. Command-Line Arguments

Command-line arguments have the highest priority.

```bash
logport watch --brokers kafka:9092 --topic my_logs /var/log/syslog
```

## Default Settings

Configure default values for watches:

### default.brokers

Default broker list for watches.

```bash
logport set default.brokers "kafka1:9092,kafka2:9092,kafka3:9092"
```

**Accepts**:
- Kafka brokers: `host:port` or `kafka://host:port`
- HTTP endpoints: `http://host:port/path` or `https://host:port/path`
- Multiple values: comma-separated list

**Examples**:
```bash
# Single Kafka broker
logport set default.brokers "localhost:9092"

# Multiple Kafka brokers
logport set default.brokers "kafka1:9092,kafka2:9092,kafka3:9092"

# HTTP endpoint
logport set default.brokers "https://logs.example.com/ingest"

# Multiple HTTP endpoints
logport set default.brokers "https://logs1.example.com,https://logs2.example.com"
```

### default.topic

Default Kafka topic for watches.

```bash
logport set default.topic "my_logs"
```

**Note**: Only used for Kafka producers. HTTP producers use the URL path.

### default.product_code

Default product code for watches. Product codes identify organizational products or teams.

```bash
logport set default.product_code "prd4096"
```

**Recommendation**: Use a unique code per application or team (e.g., prd1024, prd2048, prd4096).

### default.log_type

Default log type for watches. Log types categorize logs by purpose.

```bash
logport set default.log_type "system"
```

**Common values**:
- `system` - System logs
- `application` - Application logs
- `security` - Security logs
- `audit` - Audit logs
- `access` - Access logs
- `error` - Error logs
- `debug` - Debug logs

### default.hostname

Default hostname that appears in log entries.

```bash
logport set default.hostname "my.sample.hostname"
```

**Default**: System hostname

## Producer Settings

### Kafka Producer Settings

Configure librdkafka using the `rdkafka.producer.` prefix.

#### Performance Settings

```bash
# Message throughput (default: 500,000)
logport set rdkafka.producer.queue.buffering.max.messages 1000

# Buffer time in milliseconds (default: 1000)
logport set rdkafka.producer.queue.buffering.max.ms 100

# Batch size (default: 10,000)
logport set rdkafka.producer.batch.num.messages 1000

# Message timeout (default: 300,000 ms)
logport set rdkafka.producer.message.timeout.ms 30000

# In-flight requests (default: 5)
logport set rdkafka.producer.max.in.flight.requests.per.connection 1000000
```

#### Reliability Settings

```bash
# Acknowledgment level: 0 (none), 1 (leader), -1 (all replicas)
logport set rdkafka.producer.request.required.acks 1

# Number of retries (default: 2)
logport set rdkafka.producer.message.send.max.retries 3

# Retry backoff (default: 100 ms)
logport set rdkafka.producer.retry.backoff.ms 200
```

#### Compression

```bash
# Compression type: none, gzip, snappy, lz4, zstd
logport set rdkafka.producer.compression.type snappy

# Compression level (codec-specific, -1 = default)
logport set rdkafka.producer.compression.level 6
```

#### Authentication

```bash
# SASL mechanism: PLAIN, SCRAM-SHA-256, SCRAM-SHA-512, GSSAPI
logport set rdkafka.producer.sasl.mechanism PLAIN
logport set rdkafka.producer.sasl.username myuser
logport set rdkafka.producer.sasl.password mypassword

# Security protocol: PLAINTEXT, SSL, SASL_PLAINTEXT, SASL_SSL
logport set rdkafka.producer.security.protocol SASL_SSL
```

#### SSL/TLS

```bash
logport set rdkafka.producer.security.protocol SSL
logport set rdkafka.producer.ssl.ca.location /path/to/ca-cert
logport set rdkafka.producer.ssl.certificate.location /path/to/client-cert
logport set rdkafka.producer.ssl.key.location /path/to/client-key
logport set rdkafka.producer.ssl.key.password key-password
```

See [librdkafka CONFIGURATION.md](https://github.com/edenhill/librdkafka/blob/master/CONFIGURATION.md) for all available settings.

### HTTP Producer Settings

Configure HTTP producer using the `http.producer.` prefix.

```bash
# Batch size (1-100,000, default: 1000)
logport set http.producer.batch.num.messages 1000

# Request timeout in milliseconds (default: 5000)
logport set http.producer.message.timeout.ms 5000

# Enable gzip compression (default: true)
logport set http.producer.compress true

# Message format (default: application/json)
logport set http.producer.format "application/json"

# Additional metadata merged into each message (default: {})
logport set http.producer.metadata '{"source":"logport","environment":"production"}'
```

#### Format Options

**application/json** (default):
```json
{
  "messages": [...],
  "count": 2
}
```

**application/vnd.kafka.json.v2+json** (Kafka REST Proxy compatible):
```json
{
  "records": [
    {"value": {...}},
    {"value": {...}}
  ]
}
```

## Watch Configuration

Watches are configured when adding them and stored in the database.

### Adding Watches

```bash
logport watch [OPTIONS] [FILES...]
```

#### Options

| Option | Short | Description | Example |
|--------|-------|-------------|---------|
| `--brokers` | `-b` | Broker list (Kafka or HTTP) | `--brokers kafka:9092` |
| `--topic` | `-t` | Kafka topic | `--topic logs` |
| `--product-code` | `-p` | Product identifier | `--product-code prd4096` |
| `--log-type` | `-l` | Log category | `--log-type system` |
| `--hostname` | `-h` | Hostname in logs | `--hostname server-01` |

#### Examples

```bash
# Basic watch with defaults
logport watch /var/log/syslog

# Full configuration
logport watch --brokers kafka1:9092,kafka2:9092 \
              --topic system_logs \
              --product-code prd4096 \
              --log-type system \
              --hostname web-server-01 \
              /var/log/syslog

# Multiple files
logport watch --brokers kafka:9092 \
              --topic app_logs \
              /var/log/*.log

# HTTP endpoint
logport watch --brokers https://logs.example.com/ingest \
              --product-code prd2048 \
              /var/log/application.log

# Glob patterns
logport watch /var/log/*.log /var/log/app/*.log
```

### Listing Watches

```bash
logport watches
```

Output shows:
- `watch_id` - Unique identifier
- `watched_filepath` - File being watched
- `producer_type` - KAFKA or HTTP
- `brokers` - Destination brokers/endpoints
- `topic` - Kafka topic (N/A for HTTP)
- `product_code` - Product identifier
- `log_type` - Log category
- `hostname` - Hostname in messages
- `file_offset_sent` - Last sent byte offset
- `pid` - Process ID (-1 if not running)

### Removing Watches

```bash
logport unwatch <watch_id>
```

Get the watch_id from `logport watches`.

## Configuration Examples

### High-Throughput Kafka

```bash
# Set defaults
logport set default.brokers "kafka1:9092,kafka2:9092,kafka3:9092"
logport set default.topic "high_throughput_logs"

# Tune for throughput
logport set rdkafka.producer.queue.buffering.max.messages 1000000
logport set rdkafka.producer.batch.num.messages 100000
logport set rdkafka.producer.queue.buffering.max.ms 100
logport set rdkafka.producer.compression.type snappy

# Add watch
logport watch /var/log/high_volume.log
```

### Low-Latency Kafka

```bash
# Set defaults
logport set default.brokers "kafka:9092"
logport set default.topic "realtime_logs"

# Tune for latency
logport set rdkafka.producer.queue.buffering.max.messages 100
logport set rdkafka.producer.batch.num.messages 10
logport set rdkafka.producer.queue.buffering.max.ms 1
logport set rdkafka.producer.request.required.acks 1

# Add watch
logport watch /var/log/realtime.log
```

### Secure Kafka with SASL

```bash
# Set defaults
logport set default.brokers "kafka-ssl:9093"
logport set default.topic "secure_logs"

# Configure security
logport set rdkafka.producer.security.protocol SASL_SSL
logport set rdkafka.producer.sasl.mechanism SCRAM-SHA-256
logport set rdkafka.producer.sasl.username myuser
logport set rdkafka.producer.sasl.password mypassword
logport set rdkafka.producer.ssl.ca.location /etc/ssl/certs/ca-bundle.crt

# Add watch
logport watch /var/log/secure.log
```

### HTTP with Authentication

```bash
# Set defaults (with auth in URL)
logport set default.brokers "https://admin:secret@logs.example.com/v1/ingest"

# Configure HTTP producer
logport set http.producer.batch.num.messages 500
logport set http.producer.compress true
logport set http.producer.metadata '{"datacenter":"us-east-1"}'

# Add watch
logport watch /var/log/application.log
```

### Multi-Environment Setup

```bash
# Production
logport set default.brokers "kafka-prod:9092"
logport set default.topic "prod_logs"
logport set default.product_code "prd4096"

logport watch --log-type application /var/log/app.log
logport watch --log-type system /var/log/syslog
logport watch --log-type security /var/log/auth.log

# Different product code for another app
logport watch --product-code prd2048 \
              --log-type application \
              /var/log/other_app.log
```

## Configuration Best Practices

### 1. Use Defaults for Common Values

```bash
# Set once
logport set default.brokers "kafka:9092"
logport set default.topic "logs"
logport set default.product_code "prd4096"

# Use in multiple watches
logport watch /var/log/syslog
logport watch /var/log/auth.log
logport watch /var/log/app.log
```

### 2. Override When Needed

```bash
# Most logs go to default topic
logport watch /var/log/syslog
logport watch /var/log/auth.log

# Security logs go to special topic
logport watch --topic security_logs /var/log/secure.log
```

### 3. Use Product Codes Consistently

```bash
# By application
prd1024 -> Frontend application
prd2048 -> Backend API
prd4096 -> Database logs
prd8192 -> Infrastructure

# By team
prd1000 -> Engineering team
prd2000 -> DevOps team
prd3000 -> Security team
```

### 4. Categorize with Log Types

```bash
logport watch --log-type application /var/log/app.log
logport watch --log-type system /var/log/syslog
logport watch --log-type security /var/log/auth.log
logport watch --log-type access /var/log/nginx/access.log
logport watch --log-type error /var/log/nginx/error.log
```

### 5. Test Configuration Before Production

```bash
# Use 'now' for temporary testing
logport now --brokers test-kafka:9092 --topic test /tmp/test.log

# Watch in another terminal
kafkacat -C -b test-kafka:9092 -t test

# Generate test data
echo "test message" >> /tmp/test.log
```

## Resetting Configuration

### Clear All Settings

```bash
logport stop
logport destroy
logport start
```

**Warning**: This removes all watches and settings!

### Clear Specific Settings

```bash
logport unset default.brokers
logport unset default.topic
logport unset rdkafka.producer.batch.num.messages
```

## Troubleshooting Configuration

### View Current Configuration

```bash
# All settings
logport settings

# All watches
logport watches

# Environment variables
env | grep LOGPORT_
```

### Configuration Not Taking Effect

1. Check priority order: CLI args > env vars > settings > defaults
2. Restart service after changing settings:
   ```bash
   logport reload
   ```
3. For producer settings, restart the specific watch
4. Check logs:
   ```bash
   tail -f /usr/local/logport/logport.log
   ```

### Performance Issues

If experiencing timeouts or slow processing:

```bash
# Reduce throughput
logport set rdkafka.producer.queue.buffering.max.messages 1000
logport set rdkafka.producer.batch.num.messages 100

# Increase timeouts
logport set rdkafka.producer.message.timeout.ms 60000

# Reload configuration
logport reload
```
