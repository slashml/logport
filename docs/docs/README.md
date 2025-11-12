# Logport Documentation

Logport is a high-performance log forwarding agent that watches log files and sends changes to Kafka or HTTP endpoints. It provides modern observability capabilities with stable, performant, and scalable introspection into your applications.

## Table of Contents

- [Overview](overview.md)
- [Getting Started](getting-started.md)
- [Installation](installation.md)
- [Configuration](configuration.md)
- [Producers](producers.md)
  - [Kafka Producer](producers.md#kafka-producer)
  - [HTTP Producer](producers.md#http-producer)
- [Command Reference](commands.md)
- [Environment Variables](environment-variables.md)
- [Message Format](message-format.md)
- [Kubernetes Deployment](kubernetes.md)
- [Advanced Topics](advanced.md)

## Quick Start

```bash
# Download and install
wget -O librdkafka.so.1 https://github.com/homer6/logport/blob/master/build/librdkafka.so.1?raw=true
wget -O logport https://github.com/homer6/logport/blob/master/build/logport?raw=true
chmod ugo+x logport
sudo ./logport install

# Configure defaults
logport set default.brokers 192.168.1.91
logport set default.topic my_logs
logport set default.product_code prd4096

# Add a watch
logport watch /var/log/syslog

# Start the service
logport enable
logport start
```

## Key Features

- **Multiple Producer Types**: Send logs to Kafka or HTTP endpoints
- **Automatic Producer Detection**: Automatically detects producer type based on URL scheme
- **Message Persistence**: Saves unsuccessful messages and replays them to ensure no data loss
- **System Service Integration**: Runs as a system service that handles restarts
- **Offset Tracking**: Saves successful offsets to prevent replaying sent log entries
- **High Performance**: ~15k msg/s with only 1.8MB memory consumed
- **Minimal Dependencies**: librdkafka, libssl, libcrypto, libz
- **Automatic Restarts**: Automatically restarts watches on failure
- **Memory Monitoring**: Parent process monitors child processes and shuts them down if they exceed 256MB memory
- **System Telemetry**: Optionally collects system telemetry at configurable intervals
- **Product Codes**: Integrated product codes for enterprise adoption
- **Log Filtering**: Easily modify the code to filter or scrub data before sending

## System Requirements

- Ubuntu 20 (primary target)
- Ubuntu 18 (please open an issue if you need support)
- 64-bit Linux
- libc 2.5+
- Linux kernel 2.6.9+
- /usr/local/logport/logport.db must not be on an NFS mount

## Version

Current version: 0.3.0

## License

See LICENSE file in the repository.
