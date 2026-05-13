# Installing Plane on Synology DSM 7.2 (Container Manager)

This guide provides a step-by-step walkthrough for deploying Plane—the open-source project management platform—on a Synology NAS using the native **Container Manager** (Docker Compose).

## Prerequisites

* **Synology NAS** running DSM 7.2 or higher.
* **Container Manager** package installed.
* **Text Editor** package installed from the Package Center.
* **SSH Access** enabled (briefly for initial setup via Control Panel > Terminal & SNMP).

---

## Step 1: Prepare the Directory

1. Open **File Station**.
2. Navigate to your `docker` shared folder.
3. Create a new folder named `plane`.
4. Inside `plane`, create another folder named `plane-app`.

## Step 2: Download Configuration Templates

We need to fetch the official configuration files. Since the official script has a bug with Synology's version of Docker, we use it only to download the files.

1. SSH into your Synology: `ssh your_user@your_nas_ip`.
2. Navigate to the folder: `cd /volume1/docker/plane/plane-app`.
3. Download and run the script:
```bash
curl -fsSL -o setup.sh https://raw.githubusercontent.com/makeplane/plane/master/deploy/selfhost/install.sh
chmod +x setup.sh
sudo ./setup.sh

```


4. Choose **Option 1 (Install)**.
5. The script will error out with `unknown flag: --policy`. **Ignore this.** The files `docker-compose.yaml` and `variables.env` are already in your folder.
6. Type **8** to exit and close your SSH session.

## Step 3: Configure Environment Variables

1. In **File Station**, go to `docker/plane/plane-app`.
2. Rename `variables.env` to exactly **`.env`**.
3. Right-click `.env` and choose **Open with Text Editor**.
4. Replace the entire content with the text below, making sure to update `APP_DOMAIN` to your NAS IP:

```env
APP_DOMAIN=192.168.1.X  # <--- REPLACE WITH YOUR NAS IP
APP_RELEASE=v1.3.0

WEB_REPLICAS=1
SPACE_REPLICAS=1
ADMIN_REPLICAS=1
API_REPLICAS=1
WORKER_REPLICAS=1
BEAT_WORKER_REPLICAS=1
LIVE_REPLICAS=1

# Use 8090/8443 to avoid conflict with DSM system ports
LISTEN_HTTP_PORT=8090
LISTEN_HTTPS_PORT=8443

WEB_URL=http://${APP_DOMAIN}:8090
DEBUG=0
CORS_ALLOWED_ORIGINS=http://${APP_DOMAIN}:8090
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

# Secret Keys
SECRET_KEY=60gp0byfz2dvffa45cxl20p1scy9xbpf6d8c5y0geejgkyp1b5
LIVE_SERVER_SECRET_KEY=2FiJk1U2aiVPEQtzLehYGlTSnTnrs7LW

# DATA STORE SETTINGS
USE_MINIO=1
AWS_ACCESS_KEY_ID=access-key
AWS_SECRET_ACCESS_KEY=secret-key
AWS_S3_ENDPOINT_URL=http://plane-minio:9000
AWS_S3_BUCKET_NAME=uploads
FILE_SIZE_LIMIT=5242880

DOCKERHUB_USER=makeplane
PULL_POLICY=if_not_present
CUSTOM_BUILD=false

```

## Step 4: Clean the Docker-Compose File

Synology Container Manager requires a cleaner YAML format. Replace the content of your **`docker-compose.yaml`** with this:

```yaml
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

## Step 5: Build in Container Manager

1. Open **Container Manager**.
2. Go to **Project** and click **Create**.
3. **Project Name:** `plane`.
4. **Path:** Select the `/docker/plane/plane-app` folder.
5. **Source:** Choose `Use existing docker-compose.yml`.
6. Click **Next** and **Done**.

## Step 6: First Run

Wait **5-10 minutes** for the migrator to finish and the database to start. Then, access Plane at:
`http://YOUR_NAS_IP:8090`
