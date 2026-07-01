# Build & Run Guide

## Architecture (Dockerized)

```
┌──────────────────────────────────────────────────┐
│                   Host Machine                    │
│                                                    │
│  ┌────────────────────┐  ┌─────────────────────┐  │
│  │  Existing Infra    │  │  mall stack          │  │
│  │  (Docker)          │  │  (Docker Compose)    │  │
│  │                    │  │                      │  │
│  │  mysql57:3308      │  │  mall-admin:8086     │  │
│  │  mall-redis:6379   │  │  mall-portal:8082    │  │
│  │  mall-es:9200      │  │  mall-search:8087    │  │
│  │  mall-mongo:27017  │  │  mall-admin-web:8083 │  │
│  │  mall-rabbitmq:5672│  │                      │  │
│  │  mall-minio:9090   │  │  (network_mode: host)│  │
│  └────────────────────┘  └─────────────────────┘  │
└──────────────────────────────────────────────────┘
```

## Prerequisites

- Docker & Docker Compose v2
- Java 17+ & Maven 3.6+ (only for initial build)

## Quick Start

### 1. Infrastructure Services

```bash
cd document/docker

# Create data directories
sudo mkdir -p /mydata/mall/{redis,rabbitmq,elasticsearch,mongo,minio}/data
sudo chown -R 1000:1000 /mydata/mall/elasticsearch/data

# Start middleware
sudo docker compose -f docker-compose-infra.yml up -d
```

| Service | Port | Container Name |
|---|---|---|
| Redis | 6379 | mall-redis |
| RabbitMQ | 5672 / 15672 (UI) | mall-rabbitmq |
| Elasticsearch | 9200 / 9300 | mall-es |
| MongoDB | 27017 | mall-mongo |
| MinIO | 9090 / 9005 (Console) | mall-minio |

#### RabbitMQ setup

```bash
sudo docker exec mall-rabbitmq rabbitmqctl add_vhost /mall
sudo docker exec mall-rabbitmq rabbitmqctl add_user mall mall
sudo docker exec mall-rabbitmq rabbitmqctl set_permissions -p /mall mall ".*" ".*" ".*"
```

#### Elasticsearch IK plugin

```bash
curl -sL "https://release.infinilabs.com/analysis-ik/stable/elasticsearch-analysis-ik-7.17.3.zip" -o /tmp/ik.zip
sudo docker cp /tmp/ik.zip mall-es:/tmp/
sudo docker exec mall-es elasticsearch-plugin install -b file:///tmp/ik.zip
sudo docker restart mall-es
```

### 2. Database

Requires a MySQL 5.7+ instance (an existing `mysql57` container on port 3308 is expected):

```bash
# Create database
sudo docker exec mysql57 mysql -uroot -p'Admin@1234' -e \
  "CREATE DATABASE IF NOT EXISTS mall DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"

# Import schema
sudo docker exec -i mysql57 mysql -uroot -p'Admin@1234' mall < document/sql/mall.sql

# Update root password for dev profile
sudo docker exec mysql57 mysql -uroot -p'Admin@1234' -e \
  "ALTER USER 'root'@'localhost' IDENTIFIED BY 'root'; ALTER USER 'root'@'%' IDENTIFIED BY 'root'; FLUSH PRIVILEGES;"
```

### 3. Build

```bash
# Build all JARs
mvn clean package -DskipTests=true -T 2C

# Build Docker images and start
sudo docker compose build
sudo docker compose up -d
```

### 4. Access

| Service | URL |
|---|---|
| **Admin UI (Frontend)** | http://localhost:8083 |
| Admin API (Swagger) | http://localhost:8086/swagger-ui.html |
| Search API (Swagger) | http://localhost:8087/swagger-ui.html |
| Portal API (Swagger) | http://localhost:8082/swagger-ui.html |
| RabbitMQ Console | http://localhost:15672 (guest/guest) |
| MinIO Console | http://localhost:9005 (minioadmin/minioadmin) |

**Login:** Username `test` / Password `123456`

### 5. Verify

```bash
# Health
curl -s http://localhost:8086/actuator/health
curl -s http://localhost:8087/actuator/health
curl -s http://localhost:8082/actuator/health

# Admin login
curl -s -X POST http://localhost:8086/admin/login \
  -H "Content-Type: application/json" \
  -d '{"username":"test","password":"123456"}'

# List brands
TOKEN="<token>"
curl -s http://localhost:8086/brand/list -H "Authorization: Bearer $TOKEN"

# Portal home
curl -s http://localhost:8082/home/content

# Import products to Elasticsearch
curl -X POST http://localhost:8087/esProduct/importAll
```

## Docker Compose Reference

### File: `docker-compose.yml`

| Service | Image | Port (host) | Network |
|---|---|---|---|
| mall-admin | mall/mall-admin:1.0-SNAPSHOT | 8086 | host |
| mall-search | mall/mall-search:1.0-SNAPSHOT | 8087 | host |
| mall-portal | mall/mall-portal:1.0-SNAPSHOT | 8082 | host |
| mall-admin-web | mall/mall-admin-web:latest | 8083 | bridge |

All backend services use `network_mode: "host"` to directly reach infrastructure services on localhost. The frontend uses a standard bridged network with port mapping.

## Default Credentials

| Role | Username | Password |
|---|---|---|
| Admin UI | test | 123456 |

## Troubleshooting

### Port conflicts

If any target port is in use, stop the conflicting container or change the mapping in `docker-compose.yml`:
- Admin API: `SERVER_PORT=8086` (environment variable)
- Search API: `SERVER_PORT=8087`
- Portal API: `SERVER_PORT=8082`
- Frontend: `"8083:80"` (ports section)

### docker-maven-plugin fails

The project's `pom.xml` configures a remote Docker host at `192.168.3.101:2375` which is unreachable in this environment. This is non-fatal — JARs are built successfully. Use our custom Dockerfiles and compose instead.

### Elasticsearch fails to start

```bash
sudo chown -R 1000:1000 /mydata/mall/elasticsearch/data
```

### Application container exits immediately

Check logs: `sudo docker logs mall-admin`. Common causes:
- Port already in use on host
- MySQL not reachable (wrong port or credentials)
- Elasticsearch not ready yet (mall-search)
