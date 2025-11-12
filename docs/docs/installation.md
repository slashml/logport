# Installation

This guide covers installing logport on various platforms and configurations.

## System Requirements

- Ubuntu 20 (primary target)
- Ubuntu 18 (supported)
- 64-bit Linux
- libc 2.5+
- Linux kernel 2.6.9+
- /usr/local/logport/logport.db must not be on an NFS mount

## Quick Installation (Ubuntu)

### Download and Install

```bash
# Download binary and library
wget -O librdkafka.so.1 https://github.com/homer6/logport/blob/master/build/librdkafka.so.1?raw=true
wget -O logport https://github.com/homer6/logport/blob/master/build/logport?raw=true

# Make executable
chmod ugo+x logport

# Install as system service
sudo ./logport install

# Clean up downloaded files
rm librdkafka.so.1
rm logport
```

### What Gets Installed

The installation process:

1. **Copies binary** to `/usr/local/bin/logport`
2. **Copies library** to `/usr/local/lib/logport/librdkafka.so.1`
3. **Creates directory** `/usr/local/logport/` (mode 777)
4. **Creates database** `/usr/local/logport/logport.db`
5. **Installs init script** `/etc/init.d/logport`
6. **Installs logrotate config** `/etc/logrotate.d/logport`
7. **Creates log files**:
   - `/usr/local/logport/logport.log`
   - `/usr/local/logport/metrics.log`
   - `/usr/local/logport/events.log`
   - `/usr/local/logport/traces.log`
   - `/usr/local/logport/telemetry.log`

### Enable and Start

```bash
# Enable service on boot
logport enable

# Start service
logport start

# Check status
logport status
```

## Building from Source

### Prerequisites

```bash
sudo apt -y install cmake g++ librdkafka-dev libssl-dev libz-dev libpthread-stubs0-dev
```

### Build

```bash
# Clone repository
git clone https://github.com/homer6/logport.git
cd logport/

# Build with CMake
cmake .
make

# Install
cd build
sudo ./logport install
```

### Build Output

The build creates:
- `build/logport` - Main executable
- `build/librdkafka.so.1` - Kafka library (if built)

### Build Configuration

The CMakeLists.txt uses:
- **C++ Standard**: C++17 (previously C++98, recently upgraded)
- **Optimization**: -O3
- **Compiler**: GCC/G++

## Platform-Specific Notes

### Ubuntu 20.04 (Primary Target)

```bash
sudo apt update
sudo apt install -y cmake g++ librdkafka-dev libssl-dev libz-dev
git clone https://github.com/homer6/logport.git
cd logport && cmake . && make
cd build && sudo ./logport install
```

### Ubuntu 18.04

```bash
sudo apt update
sudo apt install -y cmake g++ libssl-dev libz-dev

# Install librdkafka from source
git clone https://github.com/edenhill/librdkafka.git
cd librdkafka
./configure && make && sudo make install
sudo ldconfig
cd ..

# Build logport
git clone https://github.com/homer6/logport.git
cd logport && cmake . && make
cd build && sudo ./logport install
```

### Oracle Enterprise Linux (OEL) 5.11

See `OEL511.compile` in the repository for specific build instructions.

**Note**: OEL 5.11 support requires C++98 compatibility. The main branch now uses C++17, so you may need an older version.

### Other Platforms

If you need support for other platforms, please [open an issue](https://github.com/homer6/logport/issues).

## Service Management

Logport supports both systemd and chkconfig.

### Systemd (Ubuntu 18.04+, Modern Linux)

```bash
# Enable on boot
sudo systemctl enable logport.service

# Disable on boot
sudo systemctl disable logport.service

# Start
sudo systemctl start logport.service

# Stop
sudo systemctl stop logport.service

# Restart
sudo systemctl restart logport.service

# Status
sudo systemctl status logport.service
```

### Chkconfig (Older Systems)

```bash
# Enable on boot
sudo chkconfig logport on

# Disable on boot
sudo chkconfig logport off

# Start
sudo service logport start

# Stop
sudo service logport stop

# Restart
sudo service logport restart

# Status
sudo service logport status
```

### Logport Commands (Works on All Systems)

```bash
# Enable on boot
logport enable

# Disable on boot
logport disable

# Start
logport start

# Stop
logport stop

# Restart
logport restart

# Status
logport status

# Reload configuration
logport reload
```

## Docker Installation

### Using Pre-built Binary

Create a Dockerfile:

```dockerfile
FROM ubuntu:latest

# Create directories
RUN mkdir -p /usr/local/lib/logport/install

# Copy logport files
ADD build/logport /usr/local/lib/logport/install/logport
ADD build/librdkafka.so.1 /usr/local/lib/logport/install/librdkafka.so.1

# Install
WORKDIR /usr/local/lib/logport/install
RUN /usr/local/lib/logport/install/logport install

# Configure with environment variables
ENV LOGPORT_BROKERS=kafka:9092
ENV LOGPORT_TOPIC=docker_logs
ENV LOGPORT_PRODUCT_CODE=prd4096
ENV LOGPORT_HOSTNAME=my-container

# Use logport to wrap application
ENTRYPOINT ["logport", "adopt"]
CMD ["/usr/local/bin/my-app"]
```

Build and run:

```bash
docker build -t my-app-with-logport .
docker run my-app-with-logport
```

### Multi-stage Build

```dockerfile
# Build stage
FROM ubuntu:latest AS builder

RUN apt-get update && apt-get install -y \
    cmake g++ librdkafka-dev libssl-dev libz-dev git

WORKDIR /build
RUN git clone https://github.com/homer6/logport.git
WORKDIR /build/logport
RUN cmake . && make

# Runtime stage
FROM ubuntu:latest

RUN apt-get update && apt-get install -y librdkafka1 libssl1.1 && \
    rm -rf /var/lib/apt/lists/*

COPY --from=builder /build/logport/build/logport /usr/local/bin/
COPY --from=builder /build/logport/build/librdkafka.so.1 /usr/local/lib/logport/

RUN logport install

ENV LOGPORT_BROKERS=kafka:9092
ENV LOGPORT_TOPIC=logs

ENTRYPOINT ["logport", "adopt"]
```

## Kubernetes Installation

See [Kubernetes Deployment Guide](kubernetes.md) for detailed instructions.

### Quick Kubernetes Install

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: logport-config
data:
  LOGPORT_BROKERS: "kafka:9092"
  LOGPORT_TOPIC: "k8s_logs"
  LOGPORT_PRODUCT_CODE: "prd4096"
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

## Dependencies

### Runtime Dependencies

- **librdkafka** (≥1.0.0) - Kafka client library
- **libssl** - OpenSSL for HTTPS
- **libcrypto** - Cryptography library
- **libz** - Compression library
- **pthread** - POSIX threads

### Build Dependencies

- **cmake** (≥2.6.4)
- **g++** with C++17 support
- **librdkafka-dev**
- **libssl-dev**
- **libz-dev**
- **libpthread-stubs0-dev**

## Uninstalling

### Complete Uninstall

```bash
# Stop and disable service
logport stop
logport disable

# Uninstall
logport uninstall

# Manually remove remaining files (printed by uninstall command)
rm /usr/local/lib/logport/librdkafka.so.1
rmdir /usr/local/lib/logport
rm /usr/local/bin/logport
```

**Note**: Uninstall removes:
- Init script: `/etc/init.d/logport`
- Database: `/usr/local/logport/logport.db`
- Log directory: `/usr/local/logport/`
- Logrotate config: `/etc/logrotate.d/logport`

### Reset to Factory Defaults

Keep logport installed but clear all configuration:

```bash
logport destroy
```

**Warning**: This removes all watches and settings!

## Upgrading

### Upgrade Process

```bash
# Stop service
logport stop

# Download new version
wget -O librdkafka.so.1 https://github.com/homer6/logport/blob/master/build/librdkafka.so.1?raw=true
wget -O logport https://github.com/homer6/logport/blob/master/build/logport?raw=true
chmod ugo+x logport

# Install (preserves database and configuration)
sudo ./logport install

# Clean up
rm librdkafka.so.1
rm logport

# Start service
logport start
```

### Upgrade Notes

- Database schema is maintained across versions
- Watches and settings are preserved
- Log files are preserved
- Service is not automatically restarted

## Post-Installation

### Verify Installation

```bash
# Check version
logport --version

# Check binary location
which logport

# Check database
ls -la /usr/local/logport/

# Check service status
logport status
```

### Configure Logrotate

Logport installs a logrotate configuration at `/etc/logrotate.d/logport`:

```
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

This rotates logs daily and keeps 7 days of history.

### Test Logrotate

```bash
logrotate -v -f /etc/logrotate.d/logport
```

## Troubleshooting

### Installation Fails

**Symptom**: `logport install` fails

**Solutions**:
1. Check you're running with sudo: `sudo ./logport install`
2. Verify directories are writable:
   ```bash
   sudo mkdir -p /usr/local/logport
   sudo chmod 777 /usr/local/logport
   ```
3. Check disk space: `df -h`

### Service Won't Start

**Symptom**: `logport start` fails or service immediately stops

**Solutions**:
1. Check logs: `cat /usr/local/logport/logport.log`
2. Verify database: `ls -la /usr/local/logport/logport.db`
3. Check permissions: `/usr/local/logport` should be writable
4. Ensure database is not on NFS mount

### Library Not Found

**Symptom**: `error while loading shared libraries: librdkafka.so.1`

**Solutions**:
1. Verify library exists:
   ```bash
   ls -la /usr/local/lib/logport/librdkafka.so.1
   ```
2. Update library cache:
   ```bash
   sudo ldconfig
   ```
3. Check LD_LIBRARY_PATH:
   ```bash
   export LD_LIBRARY_PATH=/usr/local/lib/logport:$LD_LIBRARY_PATH
   ```

### Build Fails

**Symptom**: `cmake` or `make` fails

**Solutions**:
1. Install all dependencies:
   ```bash
   sudo apt install cmake g++ librdkafka-dev libssl-dev libz-dev libpthread-stubs0-dev
   ```
2. Check GCC version: `g++ --version` (need C++17 support)
3. Clean and rebuild:
   ```bash
   rm -rf CMakeCache.txt CMakeFiles/
   cmake .
   make clean
   make
   ```

### Permission Denied

**Symptom**: Permission errors when running logport

**Solutions**:
1. Run install with sudo: `sudo ./logport install`
2. Check directory permissions:
   ```bash
   ls -la /usr/local/logport/
   ```
3. Fix permissions:
   ```bash
   sudo chmod 777 /usr/local/logport
   sudo chmod ugo+w /usr/local/logport/logport.db
   ```

## Security Considerations

### File Permissions

- `/usr/local/logport/` - 777 (world writable)
- `/usr/local/logport/logport.db` - 666 (world writable)
- `/usr/local/bin/logport` - 755 (executable by all)

**Note**: These permissive permissions allow any user to add watches. For production:

```bash
# Restrict to specific group
sudo chown root:logusers /usr/local/logport
sudo chmod 770 /usr/local/logport
sudo chown root:logusers /usr/local/logport/logport.db
sudo chmod 660 /usr/local/logport/logport.db
```

### Running as Non-Root

Logport needs root to:
- Install as system service
- Access system log files in /var/log

For non-root usage:
- Install with sudo but run watches as regular user
- Watch files in user-accessible locations
- Use `logport now` for temporary watches

```bash
# Install as root
sudo ./logport install

# Run as regular user
logport watch ~/app/logs/*.log
```

## Next Steps

After installation:

1. [Configure default settings](configuration.md)
2. [Add watches](getting-started.md#adding-watches)
3. [Start the service](getting-started.md#starting-the-service)
4. [View logs](getting-started.md#viewing-logs)
