# Message Format

Logport transforms log lines into structured JSON messages with metadata before sending them to producers. This document describes the message format and how logport processes different log types.

## Message Structure

Every message sent by logport is a JSON object with the following fields:

```json
{
  "@timestamp": 1556352653.816769,
  "host": "my.sample.hostname",
  "source": "/var/log/syslog",
  "prd": "prd4096",
  "log_type": "system",
  "log": "original log line"
}
```

### Core Fields

| Field | Type | Description | Always Present |
|-------|------|-------------|----------------|
| `@timestamp` | float | Unix timestamp with microsecond precision | Yes |
| `host` | string | Hostname from configuration | If configured |
| `source` | string | Absolute path to the watched file | If available |
| `prd` | string | Product code from watch configuration | If configured |
| `log_type` | string | Log type category from watch configuration | If configured |
| `log` | string | Original log line (for plain text) | For plain text |
| `log_obj` | object | Parsed JSON object (for JSON logs) | For JSON logs |

## Input Format Detection

Logport automatically detects the input format by examining the first character of each log line.

### Plain Text Logs

If the line does **not** start with `{` or `[`, it's treated as plain text.

**Input**:
```
my unstructured original log line abc123
```

**Output**:
```json
{
  "@timestamp": 1555955180.385583,
  "host": "my.sample.hostname",
  "source": "/usr/local/logport/logport.log",
  "prd": "prd4096",
  "log_type": "system",
  "log": "my unstructured original log line abc123"
}
```

### JSON Logs

If the line starts with `{` or `[`, logport attempts to parse it as JSON.

**Input**:
```json
{"level":"info","message":"user login","user_id":12345}
```

**Output**:
```json
{
  "@timestamp": 1555955180.385583,
  "host": "my.sample.hostname",
  "source": "/var/log/application.log",
  "prd": "prd4096",
  "log_type": "application",
  "log_obj": {
    "level": "info",
    "message": "user login",
    "user_id": 12345
  }
}
```

### Invalid JSON

If a line starts with `{` or `[` but is not valid JSON, it's treated as plain text:

**Input**:
```
{this is not valid json
```

**Output**:
```json
{
  "@timestamp": 1555955180.385583,
  "host": "my.sample.hostname",
  "source": "/var/log/app.log",
  "prd": "prd4096",
  "log": "{this is not valid json"
}
```

## Field Details

### @timestamp

Unix timestamp with microsecond precision indicating when logport processed the log line.

**Format**: `seconds.microseconds` (e.g., `1556352653.816769`)

**Note**: This is the processing time, not necessarily when the log was originally written. If you need the original timestamp, include it in your application logs.

### host

The hostname that appears in messages. Set via:

1. Command-line: `--hostname my-server`
2. Environment: `LOGPORT_HOSTNAME=my-server`
3. Setting: `logport set default.hostname my-server`
4. Default: System hostname

**Use cases**:
- Distinguish logs from different servers
- Override system hostname for containers
- Use Kubernetes pod names
- Use custom identifiers

### source

Absolute path to the watched file. Always resolves symlinks to the real path.

**Examples**:
- `/var/log/syslog`
- `/var/log/application.log`
- `/home/user/app/logs/app.log`

**Special values for `logport adopt`**:
- `stdout` - Standard output from adopted process
- `stderr` - Standard error from adopted process
- `process_exit` - Process exit status messages

### prd (Product Code)

User-defined identifier for organizing logs by product, application, or team.

**Recommended format**: `prd` followed by a number (e.g., `prd1024`, `prd2048`, `prd4096`)

**Use cases**:
- Distinguish logs from different applications
- Filter logs by product in downstream systems
- Track logs by team or project
- Compliance and audit trail

### log_type

User-defined category for the log. No fixed values; use what makes sense for your organization.

**Common categories**:
- `system` - Operating system logs
- `application` - Application logs
- `security` - Security-related logs
- `audit` - Audit trails
- `access` - Access logs (web servers, APIs)
- `error` - Error logs
- `debug` - Debug logs
- `metrics` - Metric logs

**Use cases**:
- Route different log types to different indices
- Filter logs by type in queries
- Set different retention policies by type
- Create dashboards per log type

### log vs log_obj

Logport uses either `log` (plain text) or `log_obj` (JSON object), never both.

**Plain text** → `log` field:
```json
{
  "@timestamp": 1556352653.816769,
  "log": "This is a plain text message"
}
```

**JSON** → `log_obj` field:
```json
{
  "@timestamp": 1556352653.816769,
  "log_obj": {
    "level": "info",
    "message": "Structured log message",
    "user_id": 12345
  }
}
```

## Complete Examples

### System Log (Plain Text)

**Watch configuration**:
```bash
logport watch --brokers kafka:9092 \
              --topic system_logs \
              --product-code prd4096 \
              --log-type system \
              --hostname server-01 \
              /var/log/syslog
```

**Input**:
```
Jan 15 10:30:45 server-01 sshd[12345]: Accepted publickey for user from 192.168.1.100
```

**Output**:
```json
{
  "@timestamp": 1673778645.123456,
  "host": "server-01",
  "source": "/var/log/syslog",
  "prd": "prd4096",
  "log_type": "system",
  "log": "Jan 15 10:30:45 server-01 sshd[12345]: Accepted publickey for user from 192.168.1.100"
}
```

### Application Log (JSON)

**Watch configuration**:
```bash
logport watch --brokers https://logs.example.com/ingest \
              --product-code prd2048 \
              --log-type application \
              --hostname api-server-03 \
              /var/log/app/application.log
```

**Input**:
```json
{"timestamp":"2023-01-15T10:30:45Z","level":"info","message":"User login successful","user_id":12345,"ip":"192.168.1.100"}
```

**Output**:
```json
{
  "@timestamp": 1673778645.123456,
  "host": "api-server-03",
  "source": "/var/log/app/application.log",
  "prd": "prd2048",
  "log_type": "application",
  "log_obj": {
    "timestamp": "2023-01-15T10:30:45Z",
    "level": "info",
    "message": "User login successful",
    "user_id": 12345,
    "ip": "192.168.1.100"
  }
}
```

### Nginx Access Log

**Watch configuration**:
```bash
logport watch --topic nginx_logs \
              --product-code prd4096 \
              --log-type access \
              /var/log/nginx/access.log
```

**Input**:
```
192.168.1.100 - - [15/Jan/2023:10:30:45 +0000] "GET /api/users HTTP/1.1" 200 1234
```

**Output**:
```json
{
  "@timestamp": 1673778645.123456,
  "host": "web-server-01",
  "source": "/var/log/nginx/access.log",
  "prd": "prd4096",
  "log_type": "access",
  "log": "192.168.1.100 - - [15/Jan/2023:10:30:45 +0000] \"GET /api/users HTTP/1.1\" 200 1234"
}
```

### Adopted Process (stdout)

**Command**:
```bash
logport adopt --brokers kafka:9092 \
              --topic process_logs \
              --product-code prd8192 \
              my-application arg1 arg2
```

**Application output**:
```
Starting application...
Connected to database
Processing request for user 12345
```

**Messages sent**:
```json
{
  "@timestamp": 1673778645.123456,
  "host": "worker-01",
  "source": "stdout",
  "prd": "prd8192",
  "log": "Starting application..."
}
```
```json
{
  "@timestamp": 1673778645.234567,
  "host": "worker-01",
  "source": "stdout",
  "prd": "prd8192",
  "log": "Connected to database"
}
```
```json
{
  "@timestamp": 1673778645.345678,
  "host": "worker-01",
  "source": "stdout",
  "prd": "prd8192",
  "log": "Processing request for user 12345"
}
```

### Process Exit Message

When using `logport adopt`, process exit information is sent:

```json
{
  "@timestamp": 1673778650.123456,
  "host": "worker-01",
  "source": "process_exit",
  "prd": "prd8192",
  "log": "logport: PID (12345) exited with status 0"
}
```

## Custom Filtering

You can modify the `filterLogLine()` function in `Watch.cc` to filter or transform log lines before sending.

### Example: Redact Sensitive Data

```cpp
string Watch::filterLogLine( const string& unfiltered_log_line ) const{

    string filtered_log_line = unfiltered_log_line;

    // Example: Redact credit card numbers
    size_t card_number_location = filtered_log_line.find( "\"card_number\":\"" );
    if( card_number_location != std::string::npos ){
        size_t redacted_location = filtered_log_line.find( "\"card_number\":\"XXX" );
        if( redacted_location == std::string::npos ){
            // Card number found and not already redacted - drop this log
            json tombstone_entry = json::object();
            tombstone_entry["@timestamp"] = get_timestamp();
            tombstone_entry["log"] = "tombstone";
            return tombstone_entry.dump();
        }
    }

    // Continue with normal processing...
}
```

### Example: Add Custom Fields

```cpp
string Watch::filterLogLine( const string& unfiltered_log_line ) const{

    // Normal processing creates log_entry...
    json log_entry = json::object();
    log_entry["@timestamp"] = get_timestamp();
    // ... other fields ...

    // Add custom fields
    log_entry["environment"] = "production";
    log_entry["version"] = "1.2.3";
    log_entry["datacenter"] = "us-east-1";

    return log_entry.dump();
}
```

## Downstream Processing

### Elasticsearch

Logport messages work well with Elasticsearch:

```json
{
  "@timestamp": 1556352653.816769,  // Elasticsearch timestamp field
  "host": "server-01",
  "source": "/var/log/syslog",
  "prd": "prd4096",
  "log_type": "system",
  "log": "Log message"
}
```

**Index template**:
```json
{
  "mappings": {
    "properties": {
      "@timestamp": {"type": "date", "format": "epoch_second"},
      "host": {"type": "keyword"},
      "source": {"type": "keyword"},
      "prd": {"type": "keyword"},
      "log_type": {"type": "keyword"},
      "log": {"type": "text"},
      "log_obj": {"type": "object"}
    }
  }
}
```

### Logstash

If you need to parse plain text logs further:

```ruby
filter {
  json {
    source => "message"
  }

  if [log] {
    # Parse plain text logs
    grok {
      match => {
        "log" => "%{SYSLOGLINE}"
      }
    }
  }

  if [log_obj] {
    # JSON logs are already parsed
    # Access fields as [log_obj][field_name]
  }
}
```

### Splunk

```conf
[logport]
INDEXED_EXTRACTIONS = json
TIME_PREFIX = "@timestamp":
TIME_FORMAT = %s.%6N
KV_MODE = json
```

## Message Size Limits

### Kafka

Default max message size in Kafka is 1MB. Configure with:

```bash
logport set rdkafka.producer.message.max.bytes 1048576
```

### HTTP

HTTP producer has no built-in size limit, but:
- Your HTTP endpoint may have limits
- Larger batches increase memory usage
- Network timeouts may occur with very large payloads

**Recommendation**: Keep log lines under 10KB for best performance.

## Empty Lines

Empty lines are silently dropped and not sent to producers.

## Message Ordering

Messages are sent in the order they are read from files:
- Single file: strict ordering guaranteed
- Multiple files: each file's ordering guaranteed independently
- Kafka: ordering within a partition (use single partition or keyed messages)
- HTTP: ordering not guaranteed across concurrent requests

## Best Practices

### 1. Use JSON for Structured Logs

Application logs should output JSON for better downstream processing:

```javascript
// Good
console.log(JSON.stringify({
  level: 'info',
  message: 'User login',
  user_id: 12345,
  timestamp: new Date().toISOString()
}));

// Less optimal (plain text)
console.log('INFO: User login - user_id: 12345');
```

### 2. Include Original Timestamps

Logport adds `@timestamp` at processing time. Include your own timestamp:

```json
{
  "app_timestamp": "2023-01-15T10:30:45Z",
  "level": "info",
  "message": "User login"
}
```

### 3. Use Consistent Product Codes

Establish a product code registry:
```
prd1024 = Frontend
prd2048 = Backend API
prd4096 = Database
prd8192 = Infrastructure
```

### 4. Categorize with log_type

Use log_type to route logs to different indices or apply different retention:
```bash
logport watch --log-type critical /var/log/critical.log     # 1 year retention
logport watch --log-type debug /var/log/debug.log           # 7 days retention
```

### 5. Keep Lines Under 10KB

Very large log lines can cause issues. If needed:
- Split large messages across multiple lines
- Use log aggregation within your application
- Increase Kafka max message size if necessary
