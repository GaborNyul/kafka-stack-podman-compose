# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Repository Overview

This is a Docker Compose configuration repository for running Apache Kafka stacks in various topologies. It's designed to replicate real-world deployment configurations with separate Zookeeper and Kafka services, solving Docker networking challenges for local development and testing.

The repository is sponsored by Conduktor and provides multiple deployment configurations ranging from simple single-node setups to complex multi-node production-like environments.

## Architecture

### Multi-Listener Configuration Pattern
All Kafka brokers use a three-listener architecture:
- **INTERNAL** (port 19092): Inter-broker and internal container communication
- **EXTERNAL** (port 9092): Host machine access via `${DOCKER_HOST_IP:-127.0.0.1}`
- **DOCKER** (port 29092): Cross-container access via `host.docker.internal`

This pattern ensures connectivity from:
1. Within the Docker network (broker-to-broker, services-to-broker)
2. From the host machine (local development)
3. From other containers outside the compose network

### KRaft vs Zookeeper Mode
- **Zookeeper-based**: Most configurations use traditional Zookeeper for cluster coordination
- **KRaft mode**: `conduktor-kafka-single.yml` uses Kafka's newer KRaft mode (no Zookeeper), configured with `KAFKA_PROCESS_ROLES: broker,controller`

### Available Stack Configurations

#### Development Configurations
- `zk-single-kafka-single.yml`: Simplest setup (1 Zookeeper, 1 Kafka broker)
- `conduktor-kafka-single.yml`: KRaft mode Kafka + Conduktor Platform UI (recommended for most development)

#### Testing/Production-like Configurations
- `zk-single-kafka-multiple.yml`: 1 Zookeeper, 3 Kafka brokers (test replication/fault-tolerance)
- `zk-multiple-kafka-single.yml`: 3 Zookeepers, 1 Kafka broker (test Zookeeper fault-tolerance)
- `zk-multiple-kafka-multiple.yml`: 3 Zookeepers, 3 Kafka brokers (production-like)
- `zk-multiple-kafka-multiple-schema-registry.yml`: Above + Schema Registry

#### Full Stack
- `full-stack.yml`: Complete ecosystem with Kafka, Zookeeper, Schema Registry, REST Proxy, Kafka Connect, ksqlDB, and Conduktor Platform

### Service Port Mappings

Standard ports across all configurations:
- **Zookeeper**: 2181 (additional nodes: 2182, 2183)
- **Kafka**: 9092 (additional brokers: 9093, 9094)
- **Conduktor Platform**: 8080
- **Schema Registry**: 8081
- **REST Proxy**: 8082
- **Kafka Connect**: 8083
- **ksqlDB**: 8088
- **JMX**: 9999 or 9001 (monitoring)

### Podman Variants
Two podman-specific compose files exist (`conduktor-kafka-single-podman-v1.yml`, `conduktor-kafka-single-podman-v2.yml`) for users running Podman instead of Docker.

## Common Commands

### Starting/Stopping Stacks

```bash
# Start a configuration (runs in background)
docker compose -f <config-file>.yml up -d

# View logs
docker compose -f <config-file>.yml logs

# View running services
docker compose -f <config-file>.yml ps

# Stop and remove containers (preserves volumes)
docker compose -f <config-file>.yml down

# Stop and remove everything including volumes (fresh start)
docker compose -f <config-file>.yml down -v
```

Common configurations:
```bash
# Most common: Single Kafka with Conduktor UI
docker compose -f conduktor-kafka-single.yml up -d

# Full ecosystem
docker compose -f full-stack.yml up -d

# Simple dev setup
docker compose -f zk-single-kafka-single.yml up -d
```

### Testing

The `test.sh` script validates stack functionality:

```bash
# Syntax: ./test.sh <compose-file> <expected-container-count>
./test.sh zk-single-kafka-single.yml 2
./test.sh full-stack.yml 8
./test.sh conduktor-kafka-single.yml 3
```

The script:
1. Tears down any existing stack
2. Starts the specified configuration
3. Waits 30 seconds for services to initialize
4. Verifies expected number of containers are running
5. Creates a test topic with appropriate replication factor
6. Produces 100 messages to the topic
7. Consumes and validates all 100 messages
8. Tears down the stack

### Kafka Client Operations

Requires Kafka CLI tools to be installed separately (not included in compose files).

```bash
# Create topic
kafka-topics --create --topic <topic-name> \
  --replication-factor <num> \
  --partitions <num> \
  --bootstrap-server localhost:9092

# List topics
kafka-topics --list --bootstrap-server localhost:9092

# Describe topic
kafka-topics --describe --topic <topic-name> --bootstrap-server localhost:9092

# Produce messages
kafka-console-producer --broker-list localhost:9092 --topic <topic-name>

# Consume messages
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic <topic-name> \
  --from-beginning
```

### Managing Kafka Connect

```bash
# View installed connectors
curl http://localhost:8083/connector-plugins

# Add custom connectors
# Place connector JARs in ./connectors/ directory (auto-mounted)
mkdir -p connectors/my-connector
cp my-connector.jar connectors/my-connector/

# Or modify the command section in full-stack.yml to install via confluent-hub
```

## Environment Variables

### DOCKER_HOST_IP
Controls external listener address. Set before starting stacks:

```bash
# Default (localhost only)
docker compose -f <config>.yml up

# Expose to network
DOCKER_HOST_IP=192.168.1.100 docker compose -f <config>.yml up

# Or export it
export DOCKER_HOST_IP=192.168.1.100
```

## Configuration Modifications

### Changing Ports

When modifying ports, you must update BOTH the port mapping AND environment variables:

**Zookeeper example** (changing to 12181):
```yaml
zoo1:
  ports:
    - "12181:12181"
  environment:
    ZOO_PORT: 12181

kafka1:
  environment:
    KAFKA_ZOOKEEPER_CONNECT: "zoo1:12181"
```

**Kafka example** (changing to 12345):
```yaml
kafka1:
  ports:
    - "12345:12345"
  environment:
    KAFKA_ADVERTISED_LISTENERS: INTERNAL://kafka1:19092,EXTERNAL://${DOCKER_HOST_IP:-127.0.0.1}:12345,DOCKER://host.docker.internal:29092
```

### Reducing Disk Usage (Testing Only)

Add to Kafka service environment:
```yaml
KAFKA_LOG_SEGMENT_BYTES: 16777216      # 16MB segments
KAFKA_LOG_RETENTION_BYTES: 134217728   # 128MB retention
```

### Disabling Confluent Metrics

Add to Kafka service environment:
```yaml
KAFKA_CONFLUENT_SUPPORT_METRICS_ENABLE: false
```

### Apple M4 Compatibility

Uncomment in `conduktor.yml`:
```yaml
CONSOLE_JAVA_OPTS: "-XX:UseSVE=0"
```

## Data Persistence

Data is persisted in Docker volumes named after the configuration file:
- `zk-single-kafka-single/` for zk-single-kafka-single.yml
- `full-stack/` for full-stack.yml
- etc.

These directories are in `.gitignore`. Use `docker compose -f <config>.yml down -v` to remove persisted data.

## CI/CD

GitHub Actions workflow (`.github/workflows/main.yml`) tests all major configurations:
- Installs Confluent Platform OSS 2.11
- Runs `test.sh` against each compose file
- Validates broker connectivity and message production/consumption
- Tests run on push/PR to master branch

## Accessing Conduktor Platform

When using configurations with Conduktor:
1. Navigate to http://localhost:8080
2. Connect to cluster at `localhost:9092` (from host)
3. Conduktor can also connect via internal listener for advanced features
