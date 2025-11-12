# Getting Started

This guide will help you get logport up and running quickly.

## Prerequisites

- Ubuntu Linux (18.04 or 20.04 recommended)
- Kafka cluster or HTTP endpoint for log ingestion
- Root access for installation

## Step 1: Install Logport

```bash
# Download binary and library
wget -O librdkafka.so.1 https://github.com/homer6/logport/blob/master/build/librdkafka.so.1?raw=true
wget -O logport https://github.com/homer6/logport/blob/master/build/logport?raw=true
chmod ugo+x logport

# Install
sudo ./logport install

# Clean up
rm librdkafka.so.1 logport
```

See [Installation Guide](installation.md) for detailed instructions.

## Step 2: Configure Default Settings

Set default values that will be used for all watches:

```bash
# Set Kafka broker(s)
logport set default.brokers 192.168.1.91

# Set default topic
logport set default.topic my_logs

# Set product code (organizational identifier)
logport set default.product_code prd4096

# (Optional) Set log type
logport set default.log_type system

# (Optional) Override hostname
logport set default.hostname my.sample.hostname
```

### Alternative: Use Environment Variables

For container deployments, use environment variables:

```bash
export LOGPORT_BROKERS="192.168.1.91"
export LOGPORT_TOPIC="my_logs"
export LOGPORT_PRODUCT_CODE="prd4096"
```

See [Environment Variables](environment-variables.md) for more details.

## Step 3: Add Watches

Add files to watch:

```bash
# Watch a single file
logport watch /var/log/syslog

# Watch multiple files
logport watch /var/log/syslog /var/log/auth.log

# Watch with glob patterns
logport watch /var/log/*.log

# Watch with specific configuration
logport watch --brokers kafka1:9092,kafka2:9092 \
              --topic system_logs \
              --product-code prd4096 \
              /var/log/syslog
```

### List Watches

```bash
logport watches
```

Output example:
```
 watch_id | watched_filepath | producer_type | brokers      | topic    | product_code | log_type | hostname           | file_offset_sent | pid
--------------------------------------------------------------------------------------------------------------------------------------
        1 | /var/log/syslog  | KAFKA         | 192.168.1.91 | my_logs  | prd4096      | system   | my.sample.hostname |                0 |  -1
```

## Step 4: Enable and Start Service

```bash
# Enable service on boot
logport enable

# Start the service
logport start
```

### Check Status

```bash
# View service status
logport status

# View logs
tail -f /usr/local/logport/logport.log

# View watches with PIDs
logport watches
```

## Step 5: Verify Logs Are Being Sent

### Using kafkacat (Kafka)

```bash
# Install kafkacat
sudo apt install kafkacat

# Watch messages
kafkacat -C -b 192.168.1.91 -t my_logs -o -10

# or with formatting
kafkacat -C -b 192.168.1.91 -t my_logs -o -10 \
  -f 'Topic %t [%p] at offset %o: key %k: %s\n'
```

### Generate Test Data

```bash
# Generate test log entries
echo "Test log entry $(date)" | sudo tee -a /var/log/syslog

# Or create a test file
while true; do
  echo "Sample log entry at $(date)" >> /tmp/sample.log
  sleep 1
done
```

## Common Use Cases

### Use Case 1: System Logs to Kafka

Forward all system logs to Kafka:

```bash
# Configure defaults
logport set default.brokers "kafka1:9092,kafka2:9092,kafka3:9092"
logport set default.topic "system_logs"
logport set default.product_code "prd4096"
logport set default.log_type "system"

# Add system log watches
logport watch /var/log/syslog
logport watch /var/log/auth.log
logport watch /var/log/kern.log

# Start service
logport enable
logport start
```

### Use Case 2: Application Logs to HTTP Endpoint

Send application logs to an HTTP endpoint:

```bash
# Watch with HTTP endpoint
logport watch --brokers https://logs.example.com/ingest \
              --product-code prd2048 \
              --log-type application \
              /var/log/myapp/*.log

# Start service
logport start
```

### Use Case 3: Container Logs (Docker)

Capture logs from a containerized application:

Create a Dockerfile:

```dockerfile
FROM ubuntu:latest

# Install logport
COPY logport /usr/local/bin/logport
COPY librdkafka.so.1 /usr/local/lib/logport/librdkafka.so.1
RUN logport install

# Configure via environment
ENV LOGPORT_BROKERS=kafka:9092
ENV LOGPORT_TOPIC=container_logs
ENV LOGPORT_PRODUCT_CODE=prd4096

# Use logport adopt to capture stdout/stderr
ENTRYPOINT ["logport", "adopt"]
CMD ["/usr/local/bin/my-app"]
```

### Use Case 4: Multiple Applications

Watch logs from multiple applications with different configurations:

```bash
# Application A logs
logport watch --product-code prd1024 \
              --log-type application \
              --topic app_a_logs \
              /var/log/app_a/*.log

# Application B logs
logport watch --product-code prd2048 \
              --log-type application \
              --topic app_b_logs \
              /var/log/app_b/*.log

# Infrastructure logs
logport watch --product-code prd4096 \
              --log-type infrastructure \
              --topic infra_logs \
              /var/log/syslog /var/log/auth.log

logport start
```

## Service Management Commands

```bash
# Start service
logport start

# Stop service
logport stop

# Restart service
logport restart

# Check status
logport status

# Reload configuration (without restart)
logport reload
```

## Temporary Watching (logport now)

Test without adding permanent watches:

```bash
# Watch temporarily (blocks)
logport now --brokers localhost:9092 \
            --topic test_logs \
            /tmp/test.log

# In another terminal
echo "test message" >> /tmp/test.log

# Watch with kafkacat
kafkacat -C -b localhost:9092 -t test_logs
```

Press Ctrl+C to stop the temporary watch.

## Adopting Processes

Wrap a process to capture stdout, stderr, and exit codes:

```bash
# Basic adoption
logport adopt /usr/local/bin/my-app

# With arguments
logport adopt /usr/local/bin/my-app --config /etc/app.conf

# With full configuration
logport adopt --brokers kafka:9092 \
              --topic process_logs \
              --product-code prd8192 \
              /usr/local/bin/my-app arg1 arg2
```

This will:
1. Start your application
2. Capture stdout → messages with `"source":"stdout"`
3. Capture stderr → messages with `"source":"stderr"`
4. Capture exit → messages with `"source":"process_exit"`

## Configuration Tuning

### For High Throughput

```bash
# Increase buffer sizes
logport set rdkafka.producer.queue.buffering.max.messages 1000000
logport set rdkafka.producer.batch.num.messages 100000
logport set rdkafka.producer.compression.type snappy

# Reload configuration
logport reload
```

### For Low Latency

```bash
# Decrease buffer sizes and timeouts
logport set rdkafka.producer.queue.buffering.max.messages 100
logport set rdkafka.producer.batch.num.messages 10
logport set rdkafka.producer.queue.buffering.max.ms 1

# Reload configuration
logport reload
```

### For Unstable Connections

```bash
# Reduce throughput, increase timeouts
logport set rdkafka.producer.queue.buffering.max.messages 1000
logport set rdkafka.producer.batch.num.messages 1000
logport set rdkafka.producer.message.timeout.ms 60000
logport set rdkafka.producer.request.required.acks 1

# Reload configuration
logport reload
```

## Viewing Logs

### Logport's Own Logs

```bash
# Main log
tail -f /usr/local/logport/logport.log

# Telemetry (if enabled)
tail -f /usr/local/logport/telemetry.log

# Metrics
tail -f /usr/local/logport/metrics.log

# Events
tail -f /usr/local/logport/events.log

# Traces
tail -f /usr/local/logport/traces.log
```

### Kafka Logs (using kafkacat)

```bash
# Latest messages
kafkacat -C -b kafka:9092 -t my_logs -o -10

# All messages from beginning
kafkacat -C -b kafka:9092 -t my_logs -o beginning

# Formatted output
kafkacat -C -b kafka:9092 -t my_logs -o -10 | jq '.'
```

### HTTP Logs

Check your HTTP endpoint's logs or dashboard.

## Troubleshooting

### No Messages Appearing in Kafka

1. **Check service is running**:
   ```bash
   logport status
   logport watches  # Check PIDs are not -1
   ```

2. **Check logport logs**:
   ```bash
   tail -f /usr/local/logport/logport.log
   ```

3. **Verify Kafka connectivity**:
   ```bash
   kafkacat -L -b kafka:9092
   ```

4. **Test with temporary watch**:
   ```bash
   logport now --brokers kafka:9092 --topic test /tmp/test.log
   echo "test" >> /tmp/test.log
   ```

### Watch Process Dies Immediately

1. **Check file exists and is readable**:
   ```bash
   ls -la /var/log/syslog
   ```

2. **Check undelivered log**:
   ```bash
   ls -la /var/log/syslog_undelivered
   ```

3. **Check for errors in logs**:
   ```bash
   tail -50 /usr/local/logport/logport.log | grep -i error
   ```

### Kafka Connection Timeouts

```bash
# Increase timeout
logport set rdkafka.producer.message.timeout.ms 60000

# Reduce throughput
logport set rdkafka.producer.queue.buffering.max.messages 1000
logport set rdkafka.producer.batch.num.messages 1000

# Reload
logport reload
```

### Memory Usage High

Logport automatically restarts watches that exceed 256MB. If this happens frequently:

```bash
# Reduce buffer sizes
logport set rdkafka.producer.queue.buffering.max.messages 10000
logport set rdkafka.producer.batch.num.messages 1000

# Reload
logport reload
```

## Next Steps

- Learn about [Configuration](configuration.md) options
- Understand [Message Format](message-format.md)
- Explore [Producer Types](producers.md) (Kafka vs HTTP)
- Deploy in [Kubernetes](kubernetes.md)
- Read [Advanced Topics](advanced.md)

## Quick Reference

```bash
# Install
sudo ./logport install

# Configure
logport set default.brokers kafka:9092
logport set default.topic my_logs
logport set default.product_code prd4096

# Add watches
logport watch /var/log/syslog

# Start
logport enable
logport start

# Check
logport status
logport watches
tail -f /usr/local/logport/logport.log

# View messages
kafkacat -C -b kafka:9092 -t my_logs -o -10

# Stop
logport stop

# Remove all
logport uninstall
```
