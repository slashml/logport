# Producers

Logport supports multiple producer types for sending log data to different destinations. The producer type is automatically detected based on the URL scheme used in the brokers configuration.

## Producer Type Detection

Logport automatically selects the appropriate producer based on the URL scheme:

- **Kafka Producer**: Used when brokers contain no scheme, `kafka://` scheme, or host:port format
- **HTTP Producer**: Used when brokers contain `http://` or `https://` scheme

### Examples

```bash
# Kafka Producer (implicit)
logport watch --brokers localhost:9092 --topic my_logs /var/log/syslog

# Kafka Producer (explicit scheme)
logport watch --brokers kafka://localhost:9092 --topic my_logs /var/log/syslog

# HTTP Producer
logport watch --brokers https://logs.example.com/ingest /var/log/syslog

# HTTP Producer with path
logport watch --brokers http://192.168.1.100:8080/v1/logs /var/log/syslog
```

## Kafka Producer

The Kafka Producer sends messages to Apache Kafka topics using librdkafka.

### Configuration

Configure Kafka producer settings using the `rdkafka.producer.` prefix:

```bash
# Set maximum messages to buffer
logport set rdkafka.producer.queue.buffering.max.messages 1000

# Set maximum time to buffer messages (ms)
logport set rdkafka.producer.queue.buffering.max.ms 100

# Set batch size
logport set rdkafka.producer.batch.num.messages 1000

# Set acknowledgment requirements
logport set rdkafka.producer.request.required.acks 1

# Set message timeout (ms)
logport set rdkafka.producer.message.timeout.ms 30000

# Set maximum in-flight requests
logport set rdkafka.producer.max.in.flight.requests.per.connection 1000000
```

### Available Settings

All [librdkafka configuration properties](https://github.com/edenhill/librdkafka/blob/master/CONFIGURATION.md) are supported. Simply prefix them with `rdkafka.producer.`.

#### Common Settings

| Setting | Description | Default |
|---------|-------------|---------|
| `rdkafka.producer.queue.buffering.max.messages` | Maximum number of messages to buffer | 500,000 |
| `rdkafka.producer.queue.buffering.max.ms` | Maximum time to buffer messages (ms) | 1000 |
| `rdkafka.producer.batch.num.messages` | Batch size for sending | 10,000 |
| `rdkafka.producer.message.timeout.ms` | Message delivery timeout | 300,000 |
| `rdkafka.producer.request.required.acks` | Acknowledgment level (0, 1, -1) | -1 |

### Watch Configuration

```bash
# Basic Kafka watch
logport watch --brokers kafka1:9092,kafka2:9092 \
              --topic logs \
              --product-code prd4096 \
              /var/log/syslog

# Multiple brokers
logport watch --brokers kafka1:9092,kafka2:9092,kafka3:9092 \
              --topic my_system_logs \
              /var/log/*.log
```

### Performance Tuning

For high-throughput scenarios:

```bash
# Increase buffer sizes
logport set rdkafka.producer.queue.buffering.max.messages 1000000
logport set rdkafka.producer.batch.num.messages 100000

# Reduce latency for time-sensitive logs
logport set rdkafka.producer.queue.buffering.max.ms 10
logport set rdkafka.producer.batch.num.messages 100
```

For connections with potential timeouts:

```bash
# Lower throughput for stability
logport set rdkafka.producer.queue.buffering.max.messages 1000
logport set rdkafka.producer.queue.buffering.max.ms 100
logport set rdkafka.producer.batch.num.messages 1000
logport set rdkafka.producer.message.timeout.ms 30000
```

## HTTP Producer

The HTTP Producer sends messages to HTTP/HTTPS endpoints with support for batching, compression, and authentication.

### Configuration

Configure HTTP producer settings using the `http.producer.` prefix:

```bash
# Set batch size (1-100000)
logport set http.producer.batch.num.messages 1000

# Set message timeout (milliseconds)
logport set http.producer.message.timeout.ms 5000

# Enable/disable gzip compression
logport set http.producer.compress true

# Set content type format
logport set http.producer.format application/json

# Set metadata to include with each batch
logport set http.producer.metadata '{"source":"logport","environment":"production"}'
```

### Available Settings

| Setting | Description | Default |
|---------|-------------|---------|
| `http.producer.batch.num.messages` | Number of messages to batch before sending | 1000 |
| `http.producer.message.timeout.ms` | HTTP request timeout in milliseconds | 5000 |
| `http.producer.compress` | Enable gzip compression (`true`/`false`) | true |
| `http.producer.format` | Message format (see below) | application/json |
| `http.producer.metadata` | JSON object merged into each message | {} |

### Message Formats

#### application/json (default)

Standard JSON array format:

```json
{
  "messages": [
    {"@timestamp": 1556352653.816769, "host": "server1", "log": "message 1"},
    {"@timestamp": 1556352653.816770, "host": "server1", "log": "message 2"}
  ],
  "count": 2
}
```

#### application/vnd.kafka.json.v2+json

Kafka REST Proxy compatible format:

```json
{
  "records": [
    {
      "value": {"@timestamp": 1556352653.816769, "host": "server1", "log": "message 1"}
    },
    {
      "value": {"@timestamp": 1556352653.816770, "host": "server1", "log": "message 2"}
    }
  ]
}
```

### Authentication

HTTP Producer supports HTTP Basic Authentication through URL encoding:

```bash
# Basic authentication
logport watch --brokers https://username:password@logs.example.com/ingest /var/log/syslog

# With special characters (URL encode them)
logport watch --brokers https://user%40domain:p%40ssw0rd@logs.example.com/ingest /var/log/syslog
```

### Watch Configuration

```bash
# Basic HTTP watch
logport watch --brokers https://logs.example.com/ingest \
              --product-code prd4096 \
              /var/log/syslog

# With authentication and path
logport watch --brokers https://admin:secret@logs.example.com:8443/v1/ingest \
              --log-type system \
              /var/log/*.log

# Multiple HTTP endpoints (load balancing)
logport watch --brokers https://logs1.example.com/ingest,https://logs2.example.com/ingest \
              /var/log/app.log
```

### HTTPS Support

HTTP Producer fully supports HTTPS with:
- TLS/SSL encryption
- Certificate validation
- Keep-alive connections
- Automatic redirect following
- Gzip compression

### Performance Tuning

For high-throughput scenarios:

```bash
# Increase batch size
logport set http.producer.batch.num.messages 10000

# Longer timeout for larger batches
logport set http.producer.message.timeout.ms 15000
```

For real-time, low-latency scenarios:

```bash
# Smaller batches for lower latency
logport set http.producer.batch.num.messages 10

# Shorter timeout
logport set http.producer.message.timeout.ms 1000
```

### Thread Pool

HTTP Producer uses a thread pool (20 threads) to send messages asynchronously, ensuring file watching is not blocked by HTTP requests.

## Undelivered Messages

Both producers save undelivered messages to a local file (`<watched_file>_undelivered`) for replay on restart. This ensures no messages are lost even if the destination is temporarily unavailable.

### Undelivered Log Behavior

1. Messages that fail to send are appended to the undelivered log
2. On watch restart, undelivered messages are sent first before new messages
3. Undelivered log is automatically cleaned up after successful replay
4. If undelivered log size stabilizes (isn't decreasing), watch is automatically restarted

## Producer Comparison

| Feature | Kafka Producer | HTTP Producer |
|---------|---------------|---------------|
| Protocol | Kafka wire protocol | HTTP/HTTPS |
| Batching | Yes (librdkafka) | Yes (manual) |
| Compression | Yes | Yes (gzip) |
| Authentication | SASL, SSL | HTTP Basic Auth |
| Multiple endpoints | Yes (brokers list) | Yes (load balancing) |
| Topic support | Yes | No (use path) |
| Performance | Very high | High |
| Complexity | Kafka cluster required | Simple HTTP server |

## Choosing a Producer

### Use Kafka Producer when:
- You have an existing Kafka infrastructure
- You need guaranteed ordering and delivery
- You need to support multiple consumers
- You want to leverage Kafka's ecosystem (Kafka Streams, KSQL, etc.)
- You need very high throughput (millions of messages/sec)

### Use HTTP Producer when:
- You want a simpler deployment without Kafka
- You have an existing HTTP log ingestion endpoint
- You're sending to SaaS logging services
- You need easier debugging (standard HTTP)
- You want to integrate with serverless functions
- You're running in restricted environments where Kafka isn't allowed
