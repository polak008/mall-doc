# Changes from Original Repository

This document tracks all modifications made to the original [macrozheng/mall](https://github.com/macrozheng/mall) project to adapt it for this environment.

---

## 1. Database Configuration — MySQL Port

**Files modified:** 4 files

| File | Change |
|---|---|
| `mall-admin/src/main/resources/application-dev.yml:3` | `localhost:3306` → `localhost:3308` |
| `mall-portal/src/main/resources/application-dev.yml:6` | `localhost:3306` → `localhost:3308` |
| `mall-search/src/main/resources/application-dev.yml:3` | `localhost:3306` → `localhost:3308` |
| `mall-demo/src/main/resources/application.yml:8` | `localhost:3306` → `localhost:3308` |

**Reason:** An existing `mysql57` container was already running on port 3308. Port 3306 was occupied by another project's MySQL 8.0.

---

## 2. MySQL Root Password

The existing `mysql57` container had `MYSQL_ROOT_PASSWORD=Admin@1234`. The dev profile expects `root/root`. Changed the MySQL password to match:

```
ALTER USER 'root'@'localhost' IDENTIFIED BY 'root';
ALTER USER 'root'@'%' IDENTIFIED BY 'root';
```

---

## 3. Docker Base Image — `openjdk:17` → `eclipse-temurin:17-jre`

**Files modified:** 3 Dockerfiles

| File | Change |
|---|---|
| `mall-admin/Dockerfile` | `FROM openjdk:17` → `FROM eclipse-temurin:17-jre` |
| `mall-portal/Dockerfile` | `FROM openjdk:17` → `FROM eclipse-temurin:17-jre` |
| `mall-search/Dockerfile` | `FROM openjdk:17` → `FROM eclipse-temurin:17-jre` |

**Reason:** The `openjdk:17` image is no longer available on Docker Hub as of 2026. `eclipse-temurin:17-jre` is the recommended successor.

---

## 4. Dockerfiles — Profile Hardcoding Removed

**Files modified:** 3 Dockerfiles

The original Dockerfiles (from `pom.xml`'s docker-maven-plugin config) used:
```dockerfile
ENTRYPOINT ["java", "-jar", "-Dspring.profiles.active=prod","/${project.build.finalName}.jar"]
```

Custom Dockerfiles simplified to:
```dockerfile
FROM eclipse-temurin:17-jre
EXPOSE 8080
ARG JAR_FILE=target/mall-admin-1.0-SNAPSHOT.jar
COPY ${JAR_FILE} app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Reason:** The active profile is now passed via `SPRING_PROFILES_ACTIVE` environment variable in `docker-compose.yml`, making it configurable without rebuilding.

---

## 5. Docker Compose — Master Orchestration

**File created:** `docker-compose.yml`

Replaces the original `document/docker/docker-compose-app.yml` which:
- Referenced pre-pushed Docker images (`mall/mall-admin:1.0-SNAPSHOT`) that don't exist locally
- Used `external_links` to connect to infra containers by name
- Used default ports (8080, 8081, 8085) that conflict with existing services

New compose file:
- Builds images locally from `Dockerfile`s
- Uses `network_mode: "host"` so apps connect to infra via localhost
- Uses non-conflicting ports: 8086 (admin), 8087 (search), 8082 (portal), 8083 (frontend)
- Controls Spring profile and server port via environment variables

---

## 6. Infrastructure Docker Compose — Adapted

**File created:** `document/docker/docker-compose-infra.yml`

Modified from the original `document/docker/docker-compose-env.yml`:
- **Removed MySQL** — uses existing `mysql57` container
- **Removed Nginx** — uses existing `loginstic-hrms-nginx`
- **Removed Logstash/Kibana** — optional, not needed for basic operation
- **Changed MinIO console port** — 9001 → 9005 (9001 was occupied by `wateraid_web`)
- **Changed container names** — prefixed with `mall-` for clarity

---

## 7. Elasticsearch IK Plugin

**Installation method changed.**

Original project assumed the IK plugin was pre-installed or available via:
```
bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.17.3/elasticsearch-analysis-ik-7.17.3.zip
```

This URL now returns a 404 (the project moved to `infinilabs`). Installed via:
```bash
curl -sL "https://release.infinilabs.com/analysis-ik/stable/elasticsearch-analysis-ik-7.17.3.zip" -o /tmp/ik.zip
sudo docker cp /tmp/ik.zip mall-es:/tmp/
sudo docker exec mall-es elasticsearch-plugin install -b file:///tmp/ik.zip
```

---

## 8. RabbitMQ Vhost & User

**Created on first run.** The original project assumes these are pre-configured:
- Vhost: `/mall`
- User: `mall` / `mall`
- Permissions: full access on `/mall`

```bash
rabbitmqctl add_vhost /mall
rabbitmqctl add_user mall mall
rabbitmqctl set_permissions -p /mall mall ".*" ".*" ".*"
```

---

## 9. Frontend — mall-admin-web

**Action:** Cloned from https://github.com/macrozheng/mall-admin-web

**File modified:** `.env.development`

| Original | Modified |
|---|---|
| `VITE_BASE_SERVER_URL = http://localhost:8080` | `VITE_BASE_SERVER_URL = http://localhost:8086` |

**File created:** `Dockerfile`

Multi-stage build:
1. `node:20` — runs `npm ci && npm run build-only`
2. `nginx:alpine` — serves the built `dist/` directory

---

## 10. Docker Maven Plugin — Skipped

The project's `pom.xml` configures `io.fabric8:docker-maven-plugin` at `<docker.host>http://192.168.3.101:2375</docker.host>`. This remote Docker host is unreachable in this environment.

The plugin's `build-image` execution is bound to the `package` phase. It fails but does not block JAR creation. We build Docker images via our own `Dockerfile`s and `docker-compose.yml` instead.

**To fully skip:**
```bash
mvn clean package -DskipTests=true -Ddocker.skip=true
```

---

## 11. Application Ports — Summary

| Service | Original Port | Adapted Port | Reason |
|---|---|---|---|
| mall-admin | 8080 | 8086 | 8080 used by `loginstic-hrms-nginx` |
| mall-search | 8081 | 8087 | 8081 used by `loginstic-hrms-phpmyadmin` |
| mall-portal | 8085 | 8082 | 8085 free, changed for consistency |
| mall-admin-web | 8090 (varies) | 8083 | Free port |
| MySQL | 3306 | 3308 | 3306 used by `loginstic-hrms-db` |

---

## 12. Git — No Commits Made

All changes are **unstaged and uncommitted**. The original git history is preserved. To review changes:

```bash
git diff
git diff --cached
git status
```
