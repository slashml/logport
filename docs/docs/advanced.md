# Advanced Topics

This guide covers advanced logport features, customization, and troubleshooting.

## Architecture Deep Dive

### Process Model

Logport uses a multi-process architecture:

```
Main Process (PID stored in /var/run/logport.pid)
├── Watch Process 1 (watches /var/log/syslog)
├── Watch Process 2 (watches /var/log/auth.log)
└── Watch Process 3 (watches /var/log/app.log)
```

**Benefits**:
- Isolation: One watch failure doesn't affect others
- Resource limits: Each watch monitored independently
- Parallel processing: Watches run concurrently

### Process Monitoring

The main process monitors child processes for:

1. **Memory Usage**: Kills watches exceeding 250MB RSS
2. **CPU Time**: Kills watches exceeding 5 minutes CPU time
3. **Undelivered Log Size**: Restarts watches with stabilized undelivered logs
4. **Process Death**: Automatically restarts failed watches

Checks run every 60 seconds.

### Signal Handling

**Main Process**:
- `SIGTERM`/`SIGINT`: Graceful shutdown
- `SIGHUP`: Reload configuration
- `SIGUSR1`: Pause all watches (for logrotate)
- `SIGUSR2`: Resume all watches (after logrotate)

**Watch Processes**:
- `SIGTERM`/`SIGINT`: Graceful shutdown (7 second grace period)
- `SIGKILL`: Forced shutdown (after grace period)

## File Watching Mechanisms

### inotify

Logport uses Linux inotify for efficient file monitoring:

```c++
InotifyWatcher watcher(db, producer, watch, logport);
watcher.startWatching(); // Blocks, watching for file changes
```

**Events monitored**:
- `IN_MODIFY`: File content modified
- `IN_ATTRIB`: File attributes changed
- `IN_DELETE_SELF`: File deleted
- `IN_MOVE_SELF`: File moved

### Epoll

Level-triggered epoll for process adoption:

```c++
LevelTriggeredEpollWatcher stdout_watcher(stdout_fd);
if (stdout_watcher.watch(1000)) {
    // Data available to read
}
```

## Offset Management

### File Offsets

Logport tracks the byte offset of the last successfully sent message:

```sql
UPDATE watches SET file_offset = ? WHERE id = ?
```

**Benefits**:
- No duplicate messages after restart
- Resume from exact position
- Survive process crashes

### Offset Saving

Offsets are saved:
- After each batch is successfully sent
- Every N bytes (implementation-dependent)
- Before watch process exits

### Offset Loading

On watch start:
1. Load offset from database
2. Seek to offset in file
3. Read from current position
4. Skip previously sent bytes

## Undelivered Message Handling

### Undelivered Log File

Each watch has an undelivered log: `<watched_file>_undelivered`

**Purpose**: Store messages that failed to send (network issues, broker down, etc.)

### Replay Process

On watch restart:
1. Check if undelivered log exists and has content
2. If yes, send undelivered messages first
3. Then continue with new messages from watched file
4. Delete undelivered log when empty

### Monitoring

Main process monitors undelivered log size:
- If size is non-zero and stable (not decreasing)
- Watch is restarted to replay messages
- Prevents stuck watches

## Custom Filtering

### Scrubbing Sensitive Data

Edit `Watch::filterLogLine()` in `src/Watch.cc`:

```cpp
string Watch::filterLogLine( const string& unfiltered_log_line ) const{

    string filtered_log_line = unfiltered_log_line;

    // Example: Redact credit card numbers
    size_t card_pos = filtered_log_line.find("\"card_number\":\"");
    if (card_pos != string::npos) {
        size_t redacted_pos = filtered_log_line.find("\"card_number\":\"XXXX");
        if (redacted_pos == string::npos) {
            // Unredacted card number found - replace with tombstone
            json tombstone = json::object();
            tombstone["@timestamp"] = get_timestamp();
            tombstone["log"] = "tombstone";
            return tombstone.dump();
        }
    }

    // Continue with normal processing...
    json log_entry = json::object();
    log_entry["@timestamp"] = get_timestamp();
    // ... add fields ...

    return log_entry.dump();
}
```

### Adding Custom Fields

```cpp
string Watch::filterLogLine( const string& unfiltered_log_line ) const{

    // Normal processing...
    json log_entry = json::object();
    log_entry["@timestamp"] = get_timestamp();
    if (this->hostname.size()) log_entry["host"] = this->hostname;
    if (this->watched_filepath.size()) log_entry["source"] = this->watched_filepath;
    if (this->product_code.size()) log_entry["prd"] = this->product_code;
    if (this->log_type.size()) log_entry["log_type"] = this->log_type;

    // Add custom fields
    log_entry["environment"] = "production";
    log_entry["version"] = "1.2.3";
    log_entry["datacenter"] = "us-east-1";
    log_entry["team"] = "backend";

    // Process log line...
    if (filtered_log_line[0] != '{' && filtered_log_line[0] != '[') {
        log_entry["log"] = filtered_log_line;
    } else {
        try {
            json payload = json::parse(filtered_log_line);
            log_entry["log_obj"] = payload;
        } catch (exception& e) {
            log_entry["log"] = filtered_log_line;
        }
    }

    return log_entry.dump();
}
```

After modifying, rebuild:
```bash
cmake . && make
sudo cp build/logport /usr/local/bin/logport
logport restart
```

## Performance Tuning

### Kafka Producer Tuning

#### High Throughput

```bash
# Large buffers
logport set rdkafka.producer.queue.buffering.max.messages 1000000
logport set rdkafka.producer.batch.num.messages 100000

# Longer wait times for batching
logport set rdkafka.producer.queue.buffering.max.ms 100
logport set rdkafka.producer.linger.ms 50

# Compression
logport set rdkafka.producer.compression.type snappy
logport set rdkafka.producer.compression.level 6

# More in-flight requests
logport set rdkafka.producer.max.in.flight.requests.per.connection 5

logport reload
```

**Expected**: ~15,000-50,000 msg/s depending on message size and network

#### Low Latency

```bash
# Small buffers
logport set rdkafka.producer.queue.buffering.max.messages 100
logport set rdkafka.producer.batch.num.messages 10

# Immediate send
logport set rdkafka.producer.queue.buffering.max.ms 0
logport set rdkafka.producer.linger.ms 0

# No compression (adds latency)
logport set rdkafka.producer.compression.type none

# Fewer retries
logport set rdkafka.producer.message.send.max.retries 2

logport reload
```

**Expected**: Sub-millisecond latency, lower throughput

#### Reliability

```bash
# Wait for all replicas
logport set rdkafka.producer.request.required.acks -1

# More retries
logport set rdkafka.producer.message.send.max.retries 10
logport set rdkafka.producer.retry.backoff.ms 300

# Longer timeout
logport set rdkafka.producer.message.timeout.ms 120000

# Idempotence
logport set rdkafka.producer.enable.idempotence true

logport reload
```

**Expected**: Guaranteed delivery, higher latency

### HTTP Producer Tuning

```bash
# Batch size
logport set http.producer.batch.num.messages 1000

# Timeout
logport set http.producer.message.timeout.ms 10000

# Compression
logport set http.producer.compress true

# Metadata
logport set http.producer.metadata '{"datacenter":"us-east-1"}'

logport reload
```

## Database Management

### Database Location

`/usr/local/logport/logport.db` (SQLite)

**Requirements**:
- Must not be on NFS mount
- Should be on fast local disk
- Needs read/write permissions

### Schema

```sql
-- Watches table
CREATE TABLE watches (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    filepath TEXT NOT NULL,
    file_offset INTEGER NOT NULL DEFAULT 0,
    producer_type TEXT NOT NULL,
    brokers TEXT NOT NULL,
    topic TEXT,
    product_code TEXT,
    log_type TEXT,
    hostname TEXT NOT NULL,
    pid INTEGER NOT NULL DEFAULT -1
);

-- Settings table
CREATE TABLE settings (
    key TEXT PRIMARY KEY,
    value TEXT NOT NULL
);
```

### Manual Database Operations

```bash
# Backup database
sudo cp /usr/local/logport/logport.db /usr/local/logport/logport.db.backup

# View watches directly
sqlite3 /usr/local/logport/logport.db "SELECT * FROM watches;"

# View settings directly
sqlite3 /usr/local/logport/logport.db "SELECT * FROM settings;"

# Delete a watch
sqlite3 /usr/local/logport/logport.db "DELETE FROM watches WHERE id = 3;"

# Update offset manually
sqlite3 /usr/local/logport/logport.db \
    "UPDATE watches SET file_offset = 0 WHERE id = 1;"
```

**Warning**: Manually modifying the database can cause issues. Prefer using logport commands.

## Logrotate Integration

### Configuration

File: `/etc/logrotate.d/logport`

```bash
/usr/local/logport/*.log
{
        rotate 7
        daily
        delaycompress
        missingok
        notifempty
        compress
        firstaction
            kill -USR1 `cat /var/run/logport.pid`
        endscript
        lastaction
            kill -USR2 `cat /var/run/logport.pid`
        endscript
}
```

### How It Works

1. **firstaction**: Sends `SIGUSR1` to pause watches
2. Logrotate moves/compresses log files
3. **lastaction**: Sends `SIGUSR2` to resume watches
4. Watches resume at correct offset in new file

### Testing

```bash
# Dry run
logrotate -d /etc/logrotate.d/logport

# Force rotation
logrotate -f /etc/logrotate.d/logport

# Check status
cat /var/lib/logrotate/status
```

## Telemetry Collection

### Inspector Component

The Inspector class collects system telemetry at various intervals.

### Manual Inspection

```bash
# All telemetry
logport inspect all

# 2-second metrics (CPU, memory)
logport inspect second

# 10-second metrics (network, disk)
logport inspect 10_second

# Daily summary
logport inspect day
```

### Output

Telemetry is written to `/usr/local/logport/telemetry.log`

### Monitoring Files

Example files that can be monitored:
- `/proc/cpuinfo` - CPU information
- `/proc/meminfo` - Memory usage
- `/proc/stat` - System statistics
- `/proc/net/dev` - Network statistics

## Security

### File Permissions

**Production Setup**:

```bash
# Restrict to logport group
sudo groupadd logport
sudo usermod -a -G logport <your-user>

# Set permissions
sudo chown root:logport /usr/local/logport
sudo chmod 770 /usr/local/logport
sudo chown root:logport /usr/local/logport/logport.db
sudo chmod 660 /usr/local/logport/logport.db

# Restart service
sudo logport restart
```

### Kafka Security

#### SASL/SCRAM

```bash
logport set rdkafka.producer.security.protocol SASL_SSL
logport set rdkafka.producer.sasl.mechanism SCRAM-SHA-256
logport set rdkafka.producer.sasl.username myuser
logport set rdkafka.producer.sasl.password mypassword
logport set rdkafka.producer.ssl.ca.location /etc/ssl/certs/ca-bundle.crt
logport reload
```

#### mTLS

```bash
logport set rdkafka.producer.security.protocol SSL
logport set rdkafka.producer.ssl.ca.location /path/to/ca-cert.pem
logport set rdkafka.producer.ssl.certificate.location /path/to/client-cert.pem
logport set rdkafka.producer.ssl.key.location /path/to/client-key.pem
logport set rdkafka.producer.ssl.key.password key-password
logport reload
```

### HTTP Security

```bash
# Basic Auth (in URL)
logport watch --brokers https://user:pass@logs.example.com/ingest /var/log/app.log

# Or via environment
export LOGPORT_BROKERS="https://user:pass@logs.example.com/ingest"
```

## Troubleshooting

### High Memory Usage

**Symptom**: Watch process using >100MB

**Causes**:
- Large message buffers
- Many pending messages
- Large undelivered log

**Solutions**:
```bash
# Reduce buffer sizes
logport set rdkafka.producer.queue.buffering.max.messages 10000
logport set rdkafka.producer.batch.num.messages 1000

# Check undelivered log
ls -lh /var/log/syslog_undelivered

# Restart watch
logport restart
```

### CPU Usage High

**Symptom**: Watch process using high CPU

**Causes**:
- Very high log volume
- Inefficient filtering
- Network issues causing retries

**Solutions**:
```bash
# Check CPU time
ps aux | grep logport

# View logs
tail -f /usr/local/logport/logport.log

# Reduce retry frequency
logport set rdkafka.producer.retry.backoff.ms 500
logport reload
```

### Messages Not Sending

**Symptom**: Undelivered log growing

**Causes**:
- Broker unreachable
- Authentication failure
- Network issues
- Message too large

**Debug**:
```bash
# Check undelivered log size
ls -lh /var/log/syslog_undelivered

# Check logport logs
tail -100 /usr/local/logport/logport.log | grep -i error

# Test connectivity
kafkacat -L -b kafka:9092

# Check message size
logport set rdkafka.producer.message.max.bytes 10485760
logport reload
```

### Watch Dies Immediately

**Symptom**: PID is -1 after starting

**Debug**:
```bash
# View logs
tail -50 /usr/local/logport/logport.log

# Check file permissions
ls -la /var/log/syslog

# Test manually
logport now --brokers kafka:9092 --topic test /var/log/syslog
```

### Database Corruption

**Symptom**: "database disk image is malformed"

**Recovery**:
```bash
# Stop service
logport stop

# Backup database
sudo cp /usr/local/logport/logport.db /tmp/logport.db.backup

# Try to recover
sqlite3 /usr/local/logport/logport.db ".recover" | \
    sqlite3 /usr/local/logport/logport_recovered.db

# If that fails, restore factory settings
logport destroy

# Start fresh
logport start
```

## Migration

### Upgrading Logport

```bash
# Stop service
logport stop

# Backup database
sudo cp /usr/local/logport/logport.db /tmp/logport.db.backup

# Install new version
sudo ./logport install

# Start service
logport start

# Verify watches
logport watches
```

### Migrating to Different Kafka Cluster

```bash
# Update broker setting
logport set default.brokers new-kafka1:9092,new-kafka2:9092

# Reload service
logport reload

# Verify in new cluster
kafkacat -C -b new-kafka1:9092 -t my_logs -o -1
```

### Switching from Kafka to HTTP

```bash
# Stop service
logport stop

# Remove all watches
logport destroy

# Configure HTTP
logport set default.brokers https://logs.example.com/ingest

# Add watches back
logport watch /var/log/syslog /var/log/auth.log

# Start service
logport start
```

## Development

### Building

```bash
git clone https://github.com/homer6/logport.git
cd logport
cmake .
make
```

### Testing Changes

```bash
# Build
make

# Test locally
./build/logport install
logport watch /tmp/test.log
echo "test" >> /tmp/test.log
```

### Debug Build

```bash
# Edit CMakeLists.txt
set(LOGPORT_COMPILE_OPTIONS
    -Wall
    -Wextra
    -g      # Enable debugging symbols
    -O0     # Disable optimizations
    -std=c++17
    ...
)

# Rebuild
cmake . && make
```

### Running with gdb

```bash
# Start logport in gdb
gdb --args ./build/logport now --brokers kafka:9092 --topic test /tmp/test.log

# In gdb
(gdb) break Watch::runNow
(gdb) run
```

## Performance Benchmarks

### Typical Performance

- **Throughput**: 15,000 msg/s
- **Memory**: 1.8MB per watch
- **CPU**: <1% per watch (idle)
- **Latency**: Sub-millisecond (unloaded)

### Stress Test

```bash
# Generate high-volume logs
while true; do
    for i in {1..1000}; do
        echo "Test message $i at $(date)" >> /tmp/test.log
    done
    sleep 1
done
```

### Monitoring

```bash
# Watch resource usage
watch -n 1 'ps aux | grep logport'

# Watch throughput
kafkacat -C -b kafka:9092 -t test -o -1 | wc -l

# Watch latency
time echo "test" >> /tmp/test.log
```

## Next Steps

- Review [Configuration](configuration.md) for tuning options
- See [Producers](producers.md) for producer-specific configuration
- Check [Commands](commands.md) for complete reference
- Read [Message Format](message-format.md) for output structure
