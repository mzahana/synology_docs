# ✈️ Complete Guide: Deploying Plane on Synology behind 5G/CGNAT

This document provides the full technical configuration and step-by-step instructions for hosting **Plane** on Synology DSM 7.2 using **Cloudflare Tunnels** to bypass 5G/CGNAT restrictions.

---

## 🏁 Phase 1: Domain & Cloudflare Preparation

1. **Register a Domain**: Purchase a domain (e.g., `mzahana.com`) through Cloudflare Registrar for the easiest integration.


2. **Activate Zero Trust**: In your Cloudflare dashboard, navigate to the **Zero Trust** section and set up a free account.


3. **Create Tunnel**:
* Go to **Networks > Tunnels** and click **Create a Tunnel**.


* Name it `Synology-Plane`.


* Select **Docker** as the environment and **copy the Token** provided in the command.





---

## 📂 Phase 2: Synology File Preparation

1. **Open File Station**: Navigate to your `docker` shared folder.


2. **Create Directories**:
* `/docker/plane/plane-app` (For Plane configuration).


* `/docker/cloudflared` (For the tunnel connector).




3. **SSH Access**: Ensure SSH is enabled in **Control Panel > Terminal & SNMP**.



---

## 📝 Phase 3: The Full `.env` Configuration

Navigate to `/docker/plane/plane-app`, create a file named **`.env`**, and paste the following content. This version is optimized for Cloudflare Tunnel access.

```env
APP_DOMAIN=plane.mzahana.com
APP_RELEASE=v1.3.0

WEB_REPLICAS=1
SPACE_REPLICAS=1
ADMIN_REPLICAS=1
API_REPLICAS=1
WORKER_REPLICAS=1
BEAT_WORKER_REPLICAS=1
LIVE_REPLICAS=1

# Internal ports for Synology Container Manager
LISTEN_HTTP_PORT=8090
LISTEN_HTTPS_PORT=8443

# Public Access URLs (Cloudflare handles HTTPS)
WEB_URL=https://${APP_DOMAIN}
CORS_ALLOWED_ORIGINS=https://${APP_DOMAIN}
DEBUG=0
API_BASE_URL=http://api:8000

# DB SETTINGS
PGHOST=plane-db
PGDATABASE=plane
POSTGRES_USER=plane
POSTGRES_PASSWORD=plane
POSTGRES_DB=plane
POSTGRES_PORT=5432
PGDATA=/var/lib/postgresql/data
DATABASE_URL=postgresql://plane:plane@plane-db/plane

# REDIS SETTINGS
REDIS_HOST=plane-redis
REDIS_PORT=6379
REDIS_URL=redis://plane-redis:6379/

# RabbitMQ Settings
RABBITMQ_HOST=plane-mq
RABBITMQ_PORT=5672
RABBITMQ_USER=plane
RABBITMQ_PASSWORD=plane
RABBITMQ_VHOST=plane
AMQP_URL=amqp://plane:plane@plane-mq:5672/plane

# Security Keys (Replace with your own 32+ character strings)
SECRET_KEY=60gp0byfz2dvffa45cxl20p1scy9xbpf6d8c5y0geejgkyp1b5
LIVE_SERVER_SECRET_KEY=2FiJk1U2aiVPEQtzLehYGlTSnTnrs7LW

# DATA STORE SETTINGS
USE_MINIO=1
AWS_REGION=
AWS_ACCESS_KEY_ID=access-key
AWS_SECRET_ACCESS_KEY=secret-key
AWS_S3_ENDPOINT_URL=http://plane-minio:9000
AWS_S3_BUCKET_NAME=uploads
FILE_SIZE_LIMIT=5242880

# API key rate limit
API_KEY_RATE_LIMIT=60/minute

DOCKERHUB_USER=makeplane
PULL_POLICY=if_not_present
CUSTOM_BUILD=false

```

---

## 🚀 Phase 4: Deploying Plane

1. **Open Container Manager**: Go to **Project > Create**.


2. **Settings**:
* **Project Name**: `plane`.


* **Path**: Select `/docker/plane/plane-app`.


* **Source**: Select **Create docker-compose.yml**.




3. **Compose Content**: Paste the following `docker-compose.yaml` (ensure image versions are hardcoded to `v1.3.0` to avoid Synology GUI errors).
```docker
version: '3.8'

x-db-env: &db-env
  PGHOST: ${PGHOST}
  PGDATABASE: ${PGDATABASE}
  POSTGRES_USER: ${POSTGRES_USER}
  POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
  POSTGRES_DB: ${POSTGRES_DB}
  POSTGRES_PORT: ${POSTGRES_PORT}
  PGDATA: ${PGDATA}

x-redis-env: &redis-env
  REDIS_HOST: ${REDIS_HOST}
  REDIS_PORT: ${REDIS_PORT}
  REDIS_URL: ${REDIS_URL}

x-minio-env: &minio-env
  MINIO_ROOT_USER: ${AWS_ACCESS_KEY_ID}
  MINIO_ROOT_PASSWORD: ${AWS_SECRET_ACCESS_KEY}

x-aws-s3-env: &aws-s3-env
  AWS_REGION: ${AWS_REGION}
  AWS_ACCESS_KEY_ID: ${AWS_ACCESS_KEY_ID}
  AWS_SECRET_ACCESS_KEY: ${AWS_SECRET_ACCESS_KEY}
  AWS_S3_ENDPOINT_URL: ${AWS_S3_ENDPOINT_URL}
  AWS_S3_BUCKET_NAME: ${AWS_S3_BUCKET_NAME}

x-proxy-env: &proxy-env
  APP_DOMAIN: ${APP_DOMAIN}
  FILE_SIZE_LIMIT: ${FILE_SIZE_LIMIT}
  LISTEN_HTTP_PORT: ${LISTEN_HTTP_PORT}
  LISTEN_HTTPS_PORT: ${LISTEN_HTTPS_PORT}
  BUCKET_NAME: ${AWS_S3_BUCKET_NAME}

x-mq-env: &mq-env
  RABBITMQ_HOST: ${RABBITMQ_HOST}
  RABBITMQ_PORT: ${RABBITMQ_PORT}
  RABBITMQ_DEFAULT_USER: ${RABBITMQ_USER}
  RABBITMQ_DEFAULT_PASS: ${RABBITMQ_PASSWORD}
  RABBITMQ_DEFAULT_VHOST: ${RABBITMQ_VHOST}
  RABBITMQ_VHOST: ${RABBITMQ_VHOST}

x-app-env: &app-env
  WEB_URL: ${WEB_URL}
  CORS_ALLOWED_ORIGINS: ${CORS_ALLOWED_ORIGINS}
  DATABASE_URL: ${DATABASE_URL}
  SECRET_KEY: ${SECRET_KEY}
  AMQP_URL: ${AMQP_URL}
  LIVE_SERVER_SECRET_KEY: ${LIVE_SERVER_SECRET_KEY}

services:
  web:
    image: makeplane/plane-frontend:v1.3.0
    restart: always
    depends_on: [api, worker]

  space:
    image: makeplane/plane-space:v1.3.0
    restart: always
    depends_on: [api, worker, web]

  admin:
    image: makeplane/plane-admin:v1.3.0
    restart: always
    depends_on: [api, web]

  live:
    image: makeplane/plane-live:v1.3.0
    environment: { <<: [*live-env, *redis-env] }
    restart: always
    depends_on: [api, web]

  api:
    image: makeplane/plane-backend:v1.3.0
    command: ./bin/docker-entrypoint-api.sh
    restart: always
    volumes: [logs_api:/code/plane/logs]
    environment: { <<: [*app-env, *db-env, *redis-env, *minio-env, *aws-s3-env, *proxy-env] }
    depends_on: [plane-db, plane-redis, plane-mq]

  worker:
    image: makeplane/plane-backend:v1.3.0
    command: ./bin/docker-entrypoint-worker.sh
    restart: always
    volumes: [logs_worker:/code/plane/logs]
    environment: { <<: [*app-env, *db-env, *redis-env, *minio-env, *aws-s3-env, *proxy-env] }
    depends_on: [api, plane-db, plane-redis, plane-mq]

  beat-worker:
    image: makeplane/plane-backend:v1.3.0
    command: ./bin/docker-entrypoint-beat.sh
    restart: always
    volumes: [logs_beat-worker:/code/plane/logs]
    environment: { <<: [*app-env, *db-env, *redis-env, *minio-env, *aws-s3-env, *proxy-env] }
    depends_on: [api, plane-db, plane-redis, plane-mq]

  migrator:
    image: makeplane/plane-backend:v1.3.0
    command: ./bin/docker-entrypoint-migrator.sh
    restart: on-failure
    volumes: [logs_migrator:/code/plane/logs]
    environment: { <<: [*app-env, *db-env, *redis-env, *minio-env, *aws-s3-env, *proxy-env] }
    depends_on: [plane-db, plane-redis]

  plane-db:
    image: postgres:15.7-alpine
    command: postgres -c 'max_connections=1000'
    restart: always
    environment: { <<: *db-env }
    volumes: [pgdata:/var/lib/postgresql/data]

  plane-redis:
    image: valkey/valkey:7.2.11-alpine
    restart: always
    volumes: [redisdata:/data]

  plane-mq:
    image: rabbitmq:3.13.6-management-alpine
    restart: always
    environment: { <<: *mq-env }
    volumes: [rabbitmq_data:/var/lib/rabbitmq]

  plane-minio:
    image: minio/minio:latest
    command: server /export --console-address ":9090"
    restart: always
    environment: { <<: *minio-env }
    volumes: [uploads:/export]

  proxy:
    image: makeplane/plane-proxy:v1.3.0
    restart: always
    environment: { <<: *proxy-env }
    ports:
      - "${LISTEN_HTTP_PORT}:80"
      - "${LISTEN_HTTPS_PORT}:443"
    volumes: [proxy_config:/config, proxy_data:/data]
    depends_on: [web, api, space, admin, live]

volumes:
  pgdata:
  redisdata:
  uploads:
  logs_api:
  logs_worker:
  logs_beat-worker:
  logs_migrator:
  rabbitmq_data:
  proxy_config:
  proxy_data:
```

4. **Wait**: Building takes 5–10 minutes. The `migrator` container will eventually stop—this is normal.



---

## ☁️ Phase 5: Cloudflare Tunnel Deployment

To bypass your 5G/CGNAT, we deploy the tunnel "connector" inside the Plane network.

1. **Verify Network**: In Container Manager, check **Network**. Locate the network created by Plane (usually `plane_default`).


2. **Create Tunnel Project**: Create a new Project named `cloudflared` with this YAML:



```yaml
version: '3.9'
services:
  tunnel:
    container_name: cloudflared-tunnel
    image: cloudflare/cloudflared:latest
    restart: always
    command: tunnel --no-autoupdate run --token YOUR_CLOUDFLARE_TOKEN
    networks:
      - plane_default

networks:
  plane_default:
    external: true

```

3. **Finalize in Cloudflare**:
* In the Tunnel Dashboard, go to **Public Hostname**.


* Add Hostname: `plane.mzahana.com`.


* Service Type: `HTTP`.


* URL: `proxy:80` (This targets the internal Plane proxy service).





---

## 🛠 Phase 6: Troubleshooting

> **Redirected to `synology.me`?**
> This happens if the containers cached the old URL. Go to the Plane project in Container Manager, select **Action > Clean**, then **Build** again. This forces the containers to adopt the new `.env` variables.
> 
> 

> **"Unable to reach origin service"?**
> Ensure the `cloudflared-tunnel` is on the exact same Docker network as Plane. Use `proxy:80` as the Service URL in Cloudflare, as this allows the tunnel to talk directly to the Plane Nginx container internally.
> 
> 

> **Infinite Loading Screen?**
> Clear your browser cache or test in an Incognito window. Most modern browsers cache old 301 redirects aggressively.
> 
>
