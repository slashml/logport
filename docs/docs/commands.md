# Command Reference

Complete reference for all logport commands.

## Command Line Syntax

```bash
logport [--version] [--help] <command> [<args>]
```

## System Service Commands

### install

Installs logport as a system service.

```bash
sudo ./logport install
```

**Actions**:
- Copies binary to `/usr/local/bin/logport`
- Copies library to `/usr/local/lib/logport/`
- Creates `/usr/local/logport/` directory
- Creates SQLite database
- Installs init script `/etc/init.d/logport`
- Installs logrotate configuration
- Creates log files

**Requirements**: Must run with sudo

**Output**:
```
Logport installed as a system service.
Please type 'logport start' and 'logport enable' to start and enable the service to start on boot.
The logport binary is now in your path.
```

### uninstall

Removes logport service and configuration.

```bash
logport uninstall
```

**Actions**:
- Stops the service
- Disables service from boot
- Removes init script
- Removes database
- Removes `/usr/local/logport/` directory
- Removes logrotate configuration

**Note**: Prints commands to manually remove binary and library.

### enable

Enables the service to start on bootup.

```bash
logport enable
```

**Supports**:
- systemd: `systemctl enable logport.service`
- chkconfig: `chkconfig logport on`

### disable

Disables the service from starting on bootup.

```bash
logport disable
```

**Supports**:
- systemd: `systemctl disable logport.service`
- chkconfig: `chkconfig logport off`

## Service Control Commands

### start

Starts the logport service.

```bash
logport start
```

**Behavior**:
- Forks to background (daemonizes)
- Writes PID to `/var/run/logport.pid`
- Starts all configured watches
- Each watch runs in a separate child process

**Output**:
```
Starting logport... started.
```

**Note**: If already running, displays existing PID.

### stop

Stops the logport service.

```bash
logport stop
```

**Behavior**:
- Sends SIGTERM to main process
- Main process gracefully stops all watch processes
- Removes PID file

**Output**:
```
Stopping logport - PID: 12345... stopped.
```

### restart

Restarts the service gracefully.

```bash
logport restart
```

**Behavior**:
- Stops service (waits 6 seconds)
- Starts service with new configuration
- Preserves watch offsets

**Use case**: Apply configuration changes that require restart.

### reload

Reloads configuration without stopping the service.

```bash
logport reload
```

**Behavior**:
- Sends SIGHUP to main process
- Reloads watches from database
- Starts new watches
- Continues running watches without changes

**Use case**: Add new watches without disrupting existing ones.

### status

Prints the running status of logport.

```bash
logport status
```

**Output (running)**:
```
The logport service is running (PID: 12345).
```

**Output (not running)**:
```
The logport service is not running.
```

## Watch Management Commands

### watch

Adds files to be watched permanently.

```bash
logport watch [OPTIONS] [FILES...]
```

**Options**:

| Option | Short | Argument | Description |
|--------|-------|----------|-------------|
| `--brokers` | `-b` | BROKERS | Comma-separated list of brokers or HTTP endpoints |
| `--topic` | `-t` | TOPIC | Kafka topic (not used for HTTP) |
| `--product-code` | `-p` | CODE | Product identifier |
| `--log-type` | `-l` | TYPE | Log category |
| `--hostname` | `-h` | HOSTNAME | Hostname to include in messages |

**Examples**:

```bash
# Use defaults
logport watch /var/log/syslog

# Full configuration
logport watch --brokers kafka1:9092,kafka2:9092 \
              --topic logs \
              --product-code prd4096 \
              --log-type system \
              --hostname server-01 \
              /var/log/syslog

# Multiple files
logport watch /var/log/syslog /var/log/auth.log

# Glob patterns
logport watch /var/log/*.log

# HTTP endpoint
logport watch --brokers https://logs.example.com/ingest \
              /var/log/app.log
```

**Behavior**:
- Resolves file paths to absolute paths (follows symlinks)
- Creates undelivered log file (`<file>_undelivered`)
- Adds watch to database
- If service is running, automatically starts the new watch

**Implicit install**: Runs `logport install` if not already installed.

### unwatch

Removes a watch.

```bash
logport unwatch <watch_id>
```

**Arguments**:
- `watch_id`: ID from `logport watches` output

**Example**:
```bash
logport unwatch 3
```

**Behavior**:
- Stops the watch process if running
- Removes watch from database

### watches

Lists all configured watches.

```bash
logport watches
```

**Output**:
```
 watch_id | watched_filepath          | producer_type | brokers      | topic    | product_code | log_type | hostname   | file_offset_sent | pid
----------------------------------------------------------------------------------------------------------------------------------------
        1 | /var/log/syslog           | KAFKA         | kafka:9092   | my_logs  | prd4096      | system   | server-01  |           123456 | 12345
        2 | /var/log/auth.log         | KAFKA         | kafka:9092   | my_logs  | prd4096      | system   | server-01  |            45678 | 12346
        3 | /var/log/app/error.log    | HTTP          | https://...  |          | prd2048      | error    | server-01  |            78901 | 12347
```

**Columns**:
- `watch_id`: Unique identifier for the watch
- `watched_filepath`: Absolute path to watched file
- `producer_type`: KAFKA or HTTP
- `brokers`: Destination brokers/endpoints
- `topic`: Kafka topic (empty for HTTP)
- `product_code`: Product identifier
- `log_type`: Log category
- `hostname`: Hostname in messages
- `file_offset_sent`: Byte offset of last sent message
- `pid`: Process ID (-1 if not running)

### now

Watches a file temporarily (blocks, not persistent).

```bash
logport now [OPTIONS] [FILE]
```

**Options**: Same as `watch` command

**Example**:
```bash
logport now --brokers localhost:9092 \
            --topic test_logs \
            /tmp/test.log
```

**Behavior**:
- Runs in foreground (blocks)
- Only watches the first file provided
- Watch is not saved to database
- Stops when you press Ctrl+C
- Good for testing configuration

**Use case**: Testing broker connectivity and configuration before creating permanent watches.

### adopt

Wraps a process, capturing stdout, stderr, and exit codes.

```bash
logport adopt [OPTIONS] [EXECUTABLE] [EXECUTABLE_ARGS...]
```

**Options**: Same as `watch` command (except file)

**Examples**:

```bash
# Basic adoption
logport adopt /usr/local/bin/my-app

# With arguments
logport adopt /usr/local/bin/my-app --config /etc/app.conf arg1 arg2

# Full configuration
logport adopt --brokers kafka:9092 \
              --topic process_logs \
              --product-code prd8192 \
              /usr/local/bin/my-app
```

**Behavior**:
- Forks and runs the executable
- Captures stdout → messages with `"source":"stdout"`
- Captures stderr → messages with `"source":"stderr"`
- Captures exit status → message with `"source":"process_exit"`
- Blocks until process exits
- Returns process exit code

**Use case**:
- Docker ENTRYPOINT to capture container logs
- Wrap cron jobs
- Capture one-off script output

## Settings Management Commands

### set

Sets a configuration value.

```bash
logport set <key> <value>
```

**Examples**:

```bash
# Default settings
logport set default.brokers kafka:9092
logport set default.topic my_logs
logport set default.product_code prd4096
logport set default.log_type system
logport set default.hostname my-server

# Kafka producer settings
logport set rdkafka.producer.batch.num.messages 1000
logport set rdkafka.producer.compression.type snappy

# HTTP producer settings
logport set http.producer.batch.num.messages 1000
logport set http.producer.compress true
```

**Behavior**:
- Stores value in database
- Requires `logport reload` or `logport restart` to take effect for running watches

### unset

Removes a setting.

```bash
logport unset <key>
```

**Example**:
```bash
logport unset default.hostname
logport unset rdkafka.producer.batch.num.messages
```

### settings

Lists all configured settings.

```bash
logport settings
```

**Output**:
```
 key                                         | value
--------------------------------------------------------
 default.brokers                             | kafka:9092
 default.topic                               | my_logs
 default.product_code                        | prd4096
 rdkafka.producer.batch.num.messages         | 1000
 rdkafka.producer.compression.type           | snappy
```

## Maintenance Commands

### destroy

Restores logport to factory defaults.

```bash
logport destroy
```

**Actions**:
- Stops service if running
- Removes `/usr/local/logport/` directory
- Recreates directory structure
- Creates fresh database
- Removes all watches and settings
- Starts service if it was running

**Warning**: This is destructive and cannot be undone!

**Use case**:
- Start fresh after testing
- Fix corrupted database
- Clear all configuration

## Telemetry Commands

### inspect

Produces system telemetry to telemetry log file.

```bash
logport inspect [SUBSET]
```

**Arguments**:
- `all`: All telemetry (2s, 10s, and daily)
- `second`: 2-second interval telemetry
- `10_second`: 10-second interval telemetry
- `day`: Daily telemetry

**Examples**:

```bash
# All telemetry
logport inspect all

# Just CPU/memory snapshot
logport inspect second

# Network/disk stats
logport inspect 10_second

# Daily summary
logport inspect day
```

**Output**: Writes to `/usr/local/logport/telemetry.log`

**Use case**:
- Debugging performance issues
- Monitoring resource usage
- Collecting metrics for observability

## Information Commands

### --help / -h / help

Displays help message.

```bash
logport --help
logport -h
logport help
```

**Output**: Complete command reference

### --version / -v / version

Displays version information.

```bash
logport --version
logport -v
logport version
```

**Output**:
```
logport version 0.3.0
```

## Signal Handling

Logport responds to Unix signals:

### SIGTERM / SIGINT

Graceful shutdown.

```bash
kill -TERM <pid>
# or
kill -INT <pid>
```

**Behavior**:
- Sets `run = false`
- Stops accepting new messages
- Flushes pending messages
- Stops all watch processes
- Exits cleanly

### SIGHUP

Reload configuration.

```bash
kill -HUP <pid>
```

**Behavior**:
- Sets `reload_required = true`
- Reloads watches from database
- Starts new watches
- Keeps existing watches running

**Equivalent to**: `logport reload`

### SIGUSR1

Pause all watches (for logrotate).

```bash
kill -USR1 <pid>
```

**Behavior**:
- Stops all watch processes
- Keeps main process running
- Allows log files to be rotated

**Use case**: Called by logrotate before rotating logs

### SIGUSR2

Resume all watches (for logrotate).

```bash
kill -USR2 <pid>
```

**Behavior**:
- Restarts all watch processes
- Resumes normal operation

**Use case**: Called by logrotate after rotating logs

## Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| -1 | General error |
| 1 | Command failed or unknown command |
| 2 | General exception |

## Environment Variables

All commands respect these environment variables:

- `LOGPORT_BROKERS`: Default brokers
- `LOGPORT_TOPIC`: Default topic
- `LOGPORT_PRODUCT_CODE`: Default product code
- `LOGPORT_LOG_TYPE`: Default log type
- `LOGPORT_HOSTNAME`: Default hostname

See [Environment Variables](environment-variables.md) for details.

## Common Command Patterns

### Initial Setup

```bash
sudo ./logport install
logport set default.brokers kafka:9092
logport set default.topic my_logs
logport set default.product_code prd4096
logport watch /var/log/syslog
logport enable
logport start
```

### Add Watch to Running Service

```bash
logport watch /var/log/newapp.log
# Watch automatically starts
```

### Change Configuration

```bash
logport set rdkafka.producer.batch.num.messages 5000
logport reload
```

### Troubleshooting

```bash
logport status
logport watches
tail -f /usr/local/logport/logport.log
```

### Testing

```bash
# Test configuration
logport now --brokers test:9092 --topic test /tmp/test.log

# Test process adoption
logport adopt echo "hello world"
```

### Complete Removal

```bash
logport stop
logport disable
logport uninstall
sudo rm /usr/local/bin/logport
sudo rm -rf /usr/local/lib/logport
```

## Best Practices

1. **Use `reload` instead of `restart`** when adding watches
2. **Test with `now`** before creating permanent watches
3. **Check `watches`** regularly to ensure PIDs are active
4. **Monitor logs** in `/usr/local/logport/logport.log`
5. **Use settings** for on-premises installations
6. **Use environment variables** for containers
7. **Use specific product codes** per application
8. **Use log types** to categorize logs
