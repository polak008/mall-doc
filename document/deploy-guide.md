# Deployment Guide — mall E-Commerce Platform

## Repositories

| Module | GitHub | Purpose |
|---|---|---|
| **Backend** | https://github.com/macrozheng/mall | Java Spring Boot (admin, portal, search) |
| **Admin Frontend** | https://github.com/macrozheng/mall-admin-web | Vue 3 + Element Plus admin panel |
| **Mobile App** | https://github.com/macrozheng/mall-app-web | React Native mobile storefront (optional) |
| **Learning Guide** | https://github.com/macrozheng/mall-learning | Tutorials and documentation |

---

## 1. Prerequisites

### 1.1 Hardware

| Environment | CPU | Memory | Disk |
|---|---|---|---|
| Development | 4 cores | 8 GB | 50 GB SSD |
| Production | 8+ cores | 16+ GB | 200 GB+ SSD |

### 1.2 Software

| Tool | Version | Required for |
|---|---|---|
| Docker & Docker Compose v2 | latest | Container deployment ***(recommended)*** |
| Java | 17+ | Building backend JARs |
| Maven | 3.6+ | Building backend |
| Node.js | 20 LTS | Building frontend |
| Git | 2.x | Cloning repos |

---

## 2. Deployment Architectures

### 2.1 All-in-One (Docker) — Recommended for Dev/Staging

```
┌──────────────────────────────────────────────────────────┐
│                     Single Host (Docker)                   │
│                                                            │
│  ┌─────────────────────┐   ┌───────────────────────────┐   │
│  │  Infrastructure      │   │  Application Services     │   │
│  │  (docker-compose-infra.yml)│  (docker-compose-app.yml)    │   │
│  │                      │   │                           │   │
│  │  MySQL 5.7  :3306    │   │  mall-admin    :8080      │   │
│  │  Redis 7    :6379    │   │  mall-portal   :8085      │   │
│  │  ES 7.17.3  :9200    │   │  mall-search   :8081      │   │
│  │  Mongo 4    :27017   │   │  Nginx (admin) :80        │   │
│  │  RabbitMQ   :5672    │   │                           │   │
│  │  MinIO ($)  :9090    │   │                           │   │
│  │  Kibana     :5601    │   │                           │   │
│  │  Logstash   :4560    │   │                           │   │
│  └─────────────────────┘   └───────────────────────────┘   │
└──────────────────────────────────────────────────────────┘
```

### 2.2 Network Mode (This Environment)

The current environment uses `network_mode: "host"` for backend services, meaning they reach infrastructure via `localhost`. This avoids Docker DNS issues but is Linux-only.

```
┌─────────────────────────────────────────────────────────────┐
│                         Host Machine                         │
│                                                              │
│  Infrastructure (ports exposed on host)                      │
│  ┌─────────┐ ┌──────────┐ ┌─────────┐ ┌────────┐ ┌──────┐ │
│  │ MySQL   │ │ Redis    │ │ ES      │ │ Mongo  │ │Rabbit│ │
│  │ :3308   │ │ :6379    │ │ :9200   │ │ :27017 │ │:5672 │ │
│  └─────────┘ └──────────┘ └─────────┘ └────────┘ └──────┘ │
│                                                              │
│  Application (ports exposed on host, use localhost for infra)│
│  ┌────────────┐ ┌─────────────┐ ┌────────────┐              │
│  │ mall-admin │ │ mall-portal │ │ mall-search│              │
│  │ :8086      │ │ :8082       │ │ :8087      │              │
│  └────────────┘ └─────────────┘ └────────────┘              │
│  ┌────────────────┐                                          │
│  │ mall-admin-web │  (Docker bridge, single `host` mode)    │
│  │ :8083          │                                          │
│  └────────────────┘                                          │
└─────────────────────────────────────────────────────────────┘
```

### 2.3 Production — Split Deployment

```
┌─────────────┐  ┌──────────────┐  ┌──────────────┐
│  Load       │  │  App Server 1│  │  App Server 2│
│  Balancer   │→ │  mall-admin  │  │  mall-admin  │
│  (Nginx/HA) │  │  mall-portal │  │  mall-portal │
│             │  │  mall-search │  │  mall-search │
└─────────────┘  └──────┬───────┘  └──────┬───────┘
                        │                  │
                 ┌──────┴──────────────────┴──────┐
                 │         Shared Cluster           │
                 │  MySQL (RDS/PXC)  Redis Sentinel │
                 │  ES Cluster  Mongo Replica Set   │
                 │  RabbitMQ Cluster  MinIO (S3)    │
                 └─────────────────────────────────┘
```

---

## 3. Deployment Steps

### 3.1 Clone

```bash
git clone https://github.com/macrozheng/mall.git
cd mall

# For the frontend (needed if building manually):
git clone https://github.com/macrozheng/mall-admin-web.git
```

### 3.2 Start Infrastructure (Docker)

```bash
cd document/docker

# Create persistent data directories
sudo mkdir -p /mydata/mall/{redis,rabbitmq,elasticsearch,mongo,minio}/data
sudo chown -R 1000:1000 /mydata/mall/elasticsearch/data

# Start all middleware
sudo docker compose -f docker-compose-infra.yml up -d
```

**Verify:**

```bash
sudo docker ps | grep mall-
```

| Container | Expected |
|---|---|
| mall-redis | `Up` on :6379 |
| mall-rabbitmq | `Up` on :5672, :15672 |
| mall-es | `Up` on :9200, :9300 |
| mall-mongo | `Up` on :27017 |
| mall-minio | `Up` on :9090, :9005 |

### 3.3 MySQL Setup

**Option A:** Using an existing MySQL container:

```bash
# Create database
sudo docker exec mysql57 mysql -uroot -p'Admin@1234' -e \
  "CREATE DATABASE IF NOT EXISTS mall DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"

# Import schema + seed data
sudo docker exec -i mysql57 mysql -uroot -p'Admin@1234' mall < document/sql/mall.sql
```

**Option B:** Start a dedicated MySQL with the environment compose:

```bash
sudo docker compose -f docker-compose-env.yml up -d mysql
```

> **Note:** The schema file `document/sql/mall.sql` contains both DDL and seed data (demo admin account, sample products, brands, categories, etc.).

### 3.4 RabbitMQ Configuration

```bash
sudo docker exec mall-rabbitmq rabbitmqctl add_vhost /mall
sudo docker exec mall-rabbitmq rabbitmqctl add_user mall mall
sudo docker exec mall-rabbitmq rabbitmqctl set_permissions -p /mall mall ".*" ".*" ".*"
```

### 3.5 Elasticsearch IK Plugin

```bash
curl -sL "https://release.infinilabs.com/analysis-ik/stable/elasticsearch-analysis-ik-7.17.3.zip" -o /tmp/ik.zip
sudo docker cp /tmp/ik.zip mall-es:/tmp/
sudo docker exec mall-es elasticsearch-plugin install -b file:///tmp/ik.zip
sudo docker restart mall-es

# Verify
curl -s http://localhost:9200/_cat/plugins
# Should show: analysis-ik
```

### 3.6 MinIO Setup

Access console at http://localhost:9005 (minioadmin / minioadmin).

1. Create bucket named `mall`
2. Set bucket policy to **public** (for image URLs to work without authentication)
3. Or configure the `minio.endpoint` in `application-dev.yml`

### 3.7 Build & Deploy Backend Services

#### Method A: Docker Build

```bash
# From repository root
mvn clean package -DskipTests=true -T 2C
sudo docker compose build

# Start all application services
sudo docker compose up -d
```

#### Method B: Native JAR (for development)

```bash
mvn clean package -DskipTests=true -T 2C

# Start each service
java -jar mall-admin/target/mall-admin-1.0-SNAPSHOT.jar --spring.profiles.active=dev &
java -jar mall-portal/target/mall-portal-1.0-SNAPSHOT.jar --spring.profiles.active=dev &
java -jar mall-search/target/mall-search-1.0-SNAPSHOT.jar --spring.profiles.active=dev &
```

#### Method C: IDE (IntelliJ/Eclipse)

Open the root `pom.xml` as a project, then run each module's main class:

| Module | Main Class | VM Options |
|---|---|---|
| mall-admin | `com.macro.mall.MallAdminApplication` | `-Dspring.profiles.active=dev` |
| mall-portal | `com.macro.mall.portal.MallPortalApplication` | `-Dspring.profiles.active=dev` |
| mall-search | `com.macro.mall.search.MallSearchApplication` | `-Dspring.profiles.active=dev` |

### 3.8 Build & Deploy Frontend

#### Method A: Docker

```bash
cd mall-admin-web
docker build -t mall/mall-admin-web:latest .
docker run -d -p 8083:80 --name mall-admin-web mall/mall-admin-web:latest
```

#### Method B: Native (Node.js dev server)

```bash
cd mall-admin-web
npm install
npm run dev
# Admin panel available at http://localhost:5173 (Vite default)
```

> **Note:** The admin frontend proxies API requests to `http://localhost:8086` (configurable in `vite.config.js`). If your admin API is on a different port, update the proxy target.

### 3.9 Post-Deployment Verification

```bash
# 1. Health checks
curl -s http://localhost:8086/actuator/health    # {"status":"UP"}
curl -s http://localhost:8082/actuator/health    # {"status":"UP"}
curl -s http://localhost:8087/actuator/health    # {"status":"UP"}

# 2. Admin login
TOKEN=$(curl -s -X POST http://localhost:8086/admin/login \
  -H "Content-Type: application/json" \
  -d '{"username":"test","password":"123456"}' | jq -r '.data.token')
echo "Token: $TOKEN"

# 3. Test admin API
curl -s http://localhost:8086/brand/list \
  -H "Authorization: Bearer $TOKEN" | jq '.data.list | length'
# Should return > 0 brands

# 4. Test public portal API
curl -s http://localhost:8082/home/content | jq '.code'
# Should be 200

# 5. Import products to Elasticsearch
curl -s -X POST http://localhost:8087/esProduct/importAll

# 6. Test search
curl -s "http://localhost:8087/esProduct/search?keyword=手机&pageNum=0&pageSize=5" | jq '.data.total'
# Should return products

# 7. Access admin UI
# Open http://localhost:8083 in browser, login with test/123456
```

---

## 4. Production Hardening

### 4.1 Application Configuration (`application-prod.yml`)

Create a production profile with these changes:

```yaml
# application-prod.yml
spring:
  datasource:
    url: jdbc:mysql://prod-db-host:3306/mall?useUnicode=true&characterEncoding=utf-8&serverTimezone=Asia/Shanghai&useSSL=true&requireSSL=true
    username: ${DB_USER}
    password: ${DB_PASSWORD}
  data:
    redis:
      host: ${REDIS_HOST}
      password: ${REDIS_PASSWORD}
      port: 6379
      timeout: 2000ms
      lettuce:
        pool:
          max-active: 50
          max-wait: -1ms
          max-idle: 20
          min-idle: 5

server:
  port: ${PORT:8080}

logging:
  level:
    root: warn
    com.macro.mall: info
```

Start with: `-Dspring.profiles.active=prod`

### 4.2 Security Hardening

| Area | Recommendation |
|---|---|
| **JWT Secret** | Change `jwt.secret` in `application.yml` to a 256+ bit random string |
| **JWT Expiry** | Reduce `jwt.expiration` from 7 days to 1 hour (with refresh token flow) |
| **MySQL** | Use strong passwords, restrict `root` to localhost, create app-specific user |
| **Redis** | Set `requirepass`, use dedicated Redis instance (not shared with other apps) |
| **RabbitMQ** | Change `guest` password, create per-service users with minimal permissions |
| **MinIO** | Change `minioadmin` password, set bucket to private, use presigned URLs |
| **HTTPS** | Terminate TLS at Nginx reverse proxy for all public endpoints |
| **Rate Limiting** | Add Nginx `limit_req` or Spring Cloud Gateway rate limiter |

### 4.3 Nginx Reverse Proxy (Production)

```nginx
# /etc/nginx/conf.d/mall.conf
upstream admin_api {
    server 127.0.0.1:8080;
    server 127.0.0.1:8081 backup;
}

upstream portal_api {
    server 127.0.0.1:8085;
}

upstream search_api {
    server 127.0.0.1:8086;
}

server {
    listen 443 ssl http2;
    server_name admin.yourdomain.com;

    ssl_certificate     /etc/nginx/ssl/yourdomain.crt;
    ssl_certificate_key /etc/nginx/ssl/yourdomain.key;

    location / {
        root /usr/share/nginx/html/admin;
        index index.html;
    }

    location /api/ {
        proxy_pass http://admin_api/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### 4.4 Database Optimization

```sql
-- Create app-specific user (not root)
CREATE USER 'mall_app'@'%' IDENTIFIED BY '<strong-password>';
GRANT SELECT, INSERT, UPDATE, DELETE ON mall.* TO 'mall_app'@'%';
FLUSH PRIVILEGES;

-- Index recommendations (add if missing)
ALTER TABLE oms_order ADD INDEX idx_member_id_status (member_id, status);
ALTER TABLE oms_order ADD INDEX idx_create_time (create_time);
ALTER TABLE pms_product ADD INDEX idx_category_id (product_category_id);
ALTER TABLE pms_sku_stock ADD INDEX idx_product_id (product_id);
```

### 4.5 Elasticsearch Production Tuning

```yaml
# docker-compose-infra.yml — ES production settings
environment:
  - "discovery.type=single-node"
  - "ES_JAVA_OPTS=-Xms4g -Xmx4g"
  - "bootstrap.memory_lock=true"
  - "indices.memory.index_buffer_size=20%"
  - "indices.fielddata.cache.size=40%"
  - "thread_pool.search.queue_size=1000"
```

For a multi-node cluster, remove `single-node`, set `cluster.name`, and configure discovery via `discovery.seed_hosts`.

### 4.6 Suggested Modifications for Production

| Change | Priority | Effort | Reason |
|---|---|---|---|
| Add refresh token rotation | High | Medium | Current JWT has 7-day expiry with no refresh mechanism |
| Implement payment retry with idempotency key | High | Medium | No idempotency for Alipay notify (risk of double-charge) |
| Add order status idempotency guards | High | Low | `paySuccess` can be called multiple times; should check current status |
| Switch to Redis-based distributed locks for flash sales | High | Medium | Flash sale stock deduction is not atomic under high concurrency |
| Replace `NetworkMode: host` with bridge network + DNS | Medium | Low | Host mode doesn't work on Mac/Windows and is less secure |
| Add API versioning (`/api/v1/...`) | Medium | Low | Future-proof for mobile app upgrades |
| Add OpenTelemetry tracing | Medium | Medium | No distributed tracing currently; hard to debug latency issues |
| Add CI/CD pipeline (GitHub Actions) | Medium | Low | No automated build/test/deploy pipeline |
| Replace Alipay sandbox with production keys | High | Low | Sandbox cannot process real payments |
| Add rate limiting on login endpoint | High | Low | No brute-force protection |
| Centralize config with Spring Cloud Config or K8s ConfigMap | Low | High | Config is duplicated per module |
| Migrate to Spring Boot 3.x (done) → Spring Cloud 2024 | Low | Medium | Already on Spring Boot 3.5.x; cloud compatibility may lag |

---

## 5. Docker Compose Files Reference

### 5.1 Infrastructure (docker-compose-infra.yml)

```yaml
services:
  redis:        # port 6379, data at /mydata/mall/redis/data
  rabbitmq:     # ports 5672, 15672, data at /mydata/mall/rabbitmq/data
  elasticsearch:# port 9200, 9300, data at /mydata/mall/elasticsearch/data
  mongo:        # port 27017, data at /mydata/mall/mongo/db
  minio:        # ports 9090, 9005, data at /mydata/mall/minio/data
```

### 5.2 Environment + ELK (docker-compose-env.yml)

Adds: `mysql`, `nginx`, `logstash`, `kibana` — useful for a full dev stack on a single machine.

### 5.3 Application (docker-compose-app.yml)

```yaml
services:
  mall-admin:   # image mall/mall-admin:1.0-SNAPSHOT → port 8080
  mall-search:  # image mall/mall-search:1.0-SNAPSHOT → port 8081
  mall-portal:  # image mall/mall-portal:1.0-SNAPSHOT → port 8085
```

Uses `external_links` for DNS-based service discovery (legacy Docker Compose v1 style).

---

## 6. Kubernetes Deployment (Optional)

For K8s deployments, each service gets a standard Deployment + Service:

```yaml
# mall-admin-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mall-admin
spec:
  replicas: 2
  selector:
    matchLabels:
      app: mall-admin
  template:
    metadata:
      labels:
        app: mall-admin
    spec:
      containers:
      - name: mall-admin
        image: mall/mall-admin:1.0-SNAPSHOT
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "prod"
        - name: DB_HOST
          value: "mysql-service"
        - name: REDIS_HOST
          value: "redis-service"
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1"
---
apiVersion: v1
kind: Service
metadata:
  name: mall-admin-service
spec:
  selector:
    app: mall-admin
  ports:
  - port: 8080
    targetPort: 8080
```

---

## 7. Troubleshooting Guide

### 7.1 Infrastructure Services

| Symptom | Cause | Fix |
|---|---|---|
| MySQL won't start | Port 3306/3308 in use | Check `sudo ss -tlnp \| grep 3306`; stop conflicting container |
| Redis "Connection refused" | Redis not started or wrong `application.yml` | `sudo docker logs mall-redis`; check `spring.data.redis.host` |
| Elasticsearch exits immediately | Permission denied on `/mydata/mall/elasticsearch/data` | `sudo chown -R 1000:1000 /mydata/mall/elasticsearch/data` |
| ES "max virtual memory areas" error | Host kernel setting too low | `sudo sysctl -w vm.max_map_count=262144` |
| RabbitMQ can't create vhost | Management plugin not enabled | Already enabled in `management` tag image; verify with `docker logs` |
| MinIO console unreachable | Port 9005 mapped to wrong console address | `command: server /data --console-address ":9005"` |
| Mongo won't start | Data directory permissions | `sudo chown -R 999:999 /mydata/mall/mongo/db` (Mongo runs as uid 999) |

### 7.2 Backend Application

#### Application fails to start

```bash
# Check logs
sudo docker logs mall-admin
sudo docker logs mall-portal
sudo docker logs mall-search

# Common failures:
```

| Error | Cause | Fix |
|---|---|---|
| `CommunicationsException: Communications link failure` | MySQL unreachable | Check MySQL container is running and port is correct |
| `ERR Unknown database 'mall'` | Database not created | `CREATE DATABASE mall;` then import SQL |
| `Access denied for user 'root'@'...'` | Wrong MySQL password | Update `application-dev.yml` or `docker-compose` env vars |
| `Port 8080 already in use` | Another process on same port | Change `SERVER_PORT` env var or stop the other process |
| `No bean type 'Queue' available` | RabbitMQ not configured | Create vhost/user or disable RabbitMQ config if not using |
| `index_not_found_exception` | ES index not created | Call `POST /esProduct/importAll` on mall-search |
| `ik_max_word not found` | IK plugin not installed | Install IK analyzer plugin and restart ES |

#### Application runs but APIs fail

```bash
# Test connectivity
curl -s http://localhost:8086/actuator/health

# Login test
curl -s -X POST http://localhost:8086/admin/login \
  -H "Content-Type: application/json" \
  -d '{"username":"test","password":"123456"}'
```

| Symptom | Check |
|---|---|
| Login returns 401 | Seed data not imported; check `mall.sql` was loaded |
| Login returns data but subsequent calls return 401 | `Authorization: Bearer <token>` header not sent; verify `tokenHead` prefix |
| `POST /product/list` returns empty | No products in DB; seed data missing |
| `POST /esProduct/importAll` returns 0 imported | MySQL products not accessible from mall-search; check MySQL config in `mall-search`'s `application.yml` |
| Portal returns 404 for all endpoints | Wrong profile; mall-portal uses port 8085 by default; check URL |
| CORS error in browser | `vite.config.js` proxy not configured correctly (admin frontend dev mode) |

### 7.3 Frontend

| Symptom | Cause | Fix |
|---|---|---|
| Blank page on http://localhost:8083 | Docker frontend serving but API calls failing | Open browser DevTools → Network; check if API calls return 401/404 |
| `Failed to load resource: net::ERR_CONNECTION_REFUSED` | Admin API not reachable from frontend | Frontend expects API at `localhost:8086`; verify mall-admin is up there |
| Login succeeds but redirects to 404 | Vue Router history mode without Nginx fallback | Add `try_files $uri $uri/ /index.html;` to Nginx config |
| Images not loading | MinIO bucket private or endpoint wrong | Set bucket public or configure presigned URLs; verify `minio.endpoint` |
| `npm run dev` fails with module errors | Node.js version mismatch | Use Node 20 LTS; delete `node_modules` and `npm ci` |

### 7.4 Runtime Issues

| Issue | Root Cause | Solution |
|---|---|---|
| Order auto-cancel not working | RabbitMQ config missing; vhost/user not created | Create `/mall` vhost and `mall` user; restart mall-portal |
| ES search returns no results | Products not imported | `curl -X POST http://localhost:8087/esProduct/importAll` |
| ES search returns irrelevant results | IK analyzer not installed | Install IK plugin and re-index |
| Cart items disappear after container restart | Cart stored in Redis (memory) — no persistence | Verify `--appendonly yes` in Redis command; check data volume mount |
| File upload fails with 403 | MinIO bucket permissions | Set bucket policy to `public` in MinIO console |
| "Too many connections" MySQL | Connection leak in Druid pool | Check `spring.datasource.druid.max-active` and `remove-abandoned=true` |
| Flash sale oversold | Stock not atomically decremented | Add Redis distributed lock or `UPDATE ... SET stock = stock - 1 WHERE stock > 0` in SQL |

### 7.5 Docker-Specific

```bash
# Full log dump for a service
sudo docker logs mall-admin --tail 100 -f

# Inspect container resource usage
sudo docker stats mall-admin

# Restart a single service
sudo docker compose restart mall-admin

# Rebuild a single service after code change
sudo docker compose build mall-admin
sudo docker compose up -d mall-admin

# Clean everything (data loss!)
sudo docker compose -f docker-compose-infra.yml down -v
sudo rm -rf /mydata/mall

# Enter a container
sudo docker exec -it mall-admin /bin/bash
```

### 7.6 Diagnostic Commands

```bash
# Check all running containers
sudo docker ps

# Check port bindings
sudo ss -tlnp | grep -E '808[0-9]|330[0-9]|6379|9200|27017|5672|9090'

# Check Java process inside a container
sudo docker top mall-admin

# Check if MySQL has the mall database
sudo docker exec mysql57 mysql -uroot -proot -e "SHOW DATABASES;"

# Check ES indices exist
curl -s http://localhost:9200/_cat/indices

# Check Redis keys
sudo docker exec mall-redis redis-cli keys '*cart*'

# Check RabbitMQ queues
sudo docker exec mall-rabbitmq rabbitmqctl list_queues -p /mall

# Test MinIO
curl -s http://localhost:9000/minio/health/live
```

---

## 8. Quick Start (TL;DR)

```bash
# 1. Clone
git clone https://github.com/macrozheng/mall.git
cd mall

# 2. Infrastructure
cd document/docker
sudo mkdir -p /mydata/mall/{redis,rabbitmq,elasticsearch,mongo,minio}/data
sudo chown -R 1000:1000 /mydata/mall/elasticsearch/data
sudo docker compose -f docker-compose-infra.yml up -d

# 3. MySQL
sudo docker exec mysql57 mysql -uroot -p'Admin@1234' -e "CREATE DATABASE IF NOT EXISTS mall DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
sudo docker exec -i mysql57 mysql -uroot -p'Admin@1234' mall < ../sql/mall.sql

# 4. RabbitMQ
sudo docker exec mall-rabbitmq rabbitmqctl add_vhost /mall
sudo docker exec mall-rabbitmq rabbitmqctl add_user mall mall
sudo docker exec mall-rabbitmq rabbitmqctl set_permissions -p /mall mall ".*" ".*" ".*"

# 5. ES IK plugin
curl -sL "https://release.infinilabs.com/analysis-ik/stable/elasticsearch-analysis-ik-7.17.3.zip" -o /tmp/ik.zip
sudo docker cp /tmp/ik.zip mall-es:/tmp/
sudo docker exec mall-es elasticsearch-plugin install -b file:///tmp/ik.zip
sudo docker restart mall-es

# 6. Build & start apps (back to repo root)
cd ../..
mvn clean package -DskipTests=true -T 2C
sudo docker compose build
sudo docker compose up -d

# 7. Verify
curl -s http://localhost:8086/actuator/health
curl -s -X POST http://localhost:8086/admin/login -H "Content-Type: application/json" -d '{"username":"test","password":"123456"}'
curl -s -X POST http://localhost:8087/esProduct/importAll

# 8. Open http://localhost:8083 — login test/123456
```
