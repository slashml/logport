# Overview

Logport empowers application developers and system administrators with modern observability. This is a turn-key solution for stable, performant, and scalable introspection into what your applications are doing, right now.

## What is Logport?

[Logport](https://github.com/homer6/logport) watches log files and sends changes to Kafka or HTTP endpoints (one line per message). Logport enables your applications to easily produce observability types (obtypes): Metrics, application Events, Telemetry, Traces, and Logs (METTL). Once in Kafka, [Jetstream](https://github.com/homer6/jetstream) can ship your obtypes to compatible "heads" (indices or dashboards) such as Elasticsearch, Logz.io, and Loggly.

## Architecture

![Logport Architecture](../resources/logport_architecture.jpg)

Logport consists of several key components:

### Core Components

1. **LogPort**: Main application class that manages the entire lifecycle
2. **Watch**: Represents a file being watched with its associated configuration
3. **Producer**: Abstract base class for all producers (Kafka, HTTP)
4. **InotifyWatcher**: Uses Linux inotify to efficiently watch for file changes
5. **Database**: SQLite-based storage for watches, settings, and offsets
6. **Observer**: Internal logging system for the METTL paradigm
7. **Inspector**: System telemetry collection

### Producer Types

Logport supports two producer types:

1. **Kafka Producer**: Sends messages to Apache Kafka topics
2. **HTTP Producer**: Sends messages to HTTP/HTTPS endpoints

The producer type is automatically detected based on the URL scheme in the brokers configuration:
- `kafka://` or no scheme → Kafka Producer
- `http://` or `https://` → HTTP Producer

## How It Works

1. **Watch Setup**: You configure watches for specific log files with a destination (Kafka topic or HTTP endpoint)
2. **File Monitoring**: Logport uses inotify (or epoll) to efficiently monitor file changes
3. **Message Production**: When new lines are written to watched files, they're sent to the configured producer
4. **Offset Tracking**: Successfully sent offsets are saved to prevent replaying messages
5. **Error Handling**: Failed messages are saved to an undelivered log and replayed later
6. **Process Management**: Parent process monitors child processes (one per watch) for health

## Status

Stable. Deployed in production to several mission-critical projects.

## Use Cases

### On-Premises Deployment

- Monitor system logs (/var/log/*)
- Forward application logs to Kafka
- Centralized log aggregation
- Compliance and audit logging

### Kubernetes Deployment

- Capture stdout/stderr from containerized applications
- Forward container logs to Kafka or HTTP endpoints
- Sidecar pattern for log forwarding
- DaemonSet deployment for node-level logging

### Process Adoption

- Wrap existing processes with `logport adopt`
- Capture stdout, stderr, and exit codes
- Send process output to Kafka or HTTP endpoints
- No application code changes required
