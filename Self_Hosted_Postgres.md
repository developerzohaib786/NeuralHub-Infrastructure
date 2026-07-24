# Self-Hosted PostgreSQL with Docker — NeuralHub Setup Guide

Complete guide for setting up a self-hosted PostgreSQL database with Docker on the **same Ubuntu server** already running Qdrant and Redis for NeuralHub.

> **Context**: Your server already has Docker, Qdrant (`/opt/neuralhub/docker/qdrant/`), and Redis (`/opt/neuralhub/docker/redis/`) running. This guide follows the exact same patterns and directory conventions.

---

## Table of Contents

1. [How the URL Changes](#how-the-url-changes)
2. [Create Directory Structure](#create-directory-structure)
3. [Create PostgreSQL Configuration](#create-postgresql-configuration)
4. [Run PostgreSQL Container](#run-postgresql-container)
5. [Initialize the Database](#initialize-the-database)
6. [Update Your Environment Variables](#update-your-environment-variables)
7. [Run Prisma Migrations](#run-prisma-migrations)
8. [Add to Docker Compose](#add-to-docker-compose)
9. [Backup Strategy (Implement Later)](#backup-strategy-implement-later)
10. [Verify Everything Works](#verify-everything-works)
11. [Troubleshooting](#troubleshooting)

---

## How the URL Changes

Your current Neon (cloud) URL:
```
postgresql://neondb_owner:npg_EaWkQ6gV0ehu@ep-orange-leaf-ad9ehro5-pooler.c-2.us-east-1.aws.neon.tech/neondb?sslmode=require&channel_binding=require
```

Your **new self-hosted URL** after this setup (drop-in replacement):
```
postgresql://neuralhub_user:your-strong-password@localhost:5432/neuralhub_db
```

**What changes:**
| Part | Neon (old) | Self-hosted (new) |
|---|---|---|
| User | `neondb_owner` | `neuralhub_user` |
| Password | `npg_EaWkQ6gV0ehu` | `your-strong-password` (you choose) |
| Host | `ep-orange-leaf-...neon.tech` | `localhost` |
| Port | default (5432) | `5432` |
| Database | `neondb` | `neuralhub_db` |
| SSL params | `?sslmode=require&channel_binding=require` | *(removed — local connection, no SSL needed)* |

> **Why remove SSL params?** Neon requires SSL because it is a remote cloud database. Your self-hosted Postgres runs on the same server, so the app connects over localhost — no SSL needed, and `channel_binding` is not expected either. The app code in `src/database.py` and `src/prisma_client.py` works fine without these params.

---

## Create Directory Structure

SSH into your Ubuntu server and run the following commands. This follows the exact same `/opt/neuralhub/docker/` convention used for Qdrant and Redis:

```bash
# Create directories for PostgreSQL data and config
sudo mkdir -p /opt/neuralhub/docker/postgres/data
sudo mkdir -p /opt/neuralhub/docker/postgres/init
sudo mkdir -p /opt/neuralhub/backups/postgres

# Set correct ownership (same pattern as Qdrant/Redis directories)
sudo chown -R $USER:$USER /opt/neuralhub/docker/postgres
sudo chown -R $USER:$USER /opt/neuralhub/backups/postgres
```

---

## Create PostgreSQL Configuration

### 1. Create the initialization SQL script

This runs automatically the **first time** the container starts to create the application database and user:

```bash
nano /opt/neuralhub/docker/postgres/init/01-init.sql
```

Paste the following — **replace `your-strong-password` with a real password**:

```sql
-- NeuralHub PostgreSQL Initialization Script
-- Runs once on first container start

-- Create the application user
CREATE USER neuralhub_user WITH PASSWORD 'your-strong-password';

-- Create the application database
CREATE DATABASE neuralhub_db OWNER neuralhub_user;

-- Grant all privileges on the database
GRANT ALL PRIVILEGES ON DATABASE neuralhub_db TO neuralhub_user;

-- Connect to the new database and grant schema privileges
\c neuralhub_db

GRANT ALL ON SCHEMA public TO neuralhub_user;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO neuralhub_user;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO neuralhub_user;

-- Ensure future tables created by migrations are also accessible
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO neuralhub_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON SEQUENCES TO neuralhub_user;
```

Generate a secure password if you need one:

```bash
openssl rand -base64 24
```

### 2. Create the PostgreSQL config file (optional but recommended)

```bash
nano /opt/neuralhub/docker/postgres/postgresql.conf
```

```conf
# NeuralHub PostgreSQL Configuration
# Tuned for a 4-8GB RAM server running alongside Qdrant and Redis

# Connection Settings
listen_addresses = '*'
port = 5432
max_connections = 100

# Memory Settings (adjust based on available RAM)
# On a 4GB server: shared_buffers ~512MB, effective_cache_size ~1.5GB
# On an 8GB server: shared_buffers ~1GB,   effective_cache_size ~4GB
shared_buffers = 512MB
effective_cache_size = 1536MB
work_mem = 8MB
maintenance_work_mem = 128MB

# Write-Ahead Log (WAL) — required for crash durability
wal_level = replica
archive_mode = off

# Checkpoints
checkpoint_completion_target = 0.9
wal_buffers = 16MB
min_wal_size = 256MB
max_wal_size = 1GB

# Query Planner
random_page_cost = 1.1
effective_io_concurrency = 200

# Logging
log_min_duration_statement = 1000   # Log queries slower than 1 second
log_connections = off
log_disconnections = off
log_timezone = 'UTC'

# Timezone
timezone = 'UTC'
```

---

## Run PostgreSQL Container

Run the PostgreSQL container with persistent volume and auto-restart:

```bash
docker run -d \
  --name postgres \
  --restart unless-stopped \
  -p 127.0.0.1:5432:5432 \
  -v /opt/neuralhub/docker/postgres/data:/var/lib/postgresql/data:z \
  -v /opt/neuralhub/docker/postgres/init:/docker-entrypoint-initdb.d:ro \
  -e POSTGRES_PASSWORD=your-postgres-root-password \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_DB=postgres \
  -e PGDATA=/var/lib/postgresql/data/pgdata \
  postgres:16-alpine
```

**Key points:**
- `-p 127.0.0.1:5432:5432` — binds to **localhost only**, not exposed to the internet (same security model as your Redis config which uses `protected-mode yes`)
- `-v .../data:/var/lib/postgresql/data` — **all data is persisted on the host**. If the container crashes or is removed, your data stays safe at `/opt/neuralhub/docker/postgres/data/pgdata/`
- `POSTGRES_PASSWORD` is the root (`postgres` superuser) password — the app never uses this, it uses `neuralhub_user`
- The init SQL in `/docker-entrypoint-initdb.d/` creates your app user and database automatically on first start only

### Verify the container is running

```bash
docker ps | grep postgres
```

Expected output:
```
abc123  postgres:16-alpine  "docker-entrypoint.s..."  Up 30 seconds  127.0.0.1:5432->5432/tcp  postgres
```

Check startup logs:

```bash
docker logs postgres --tail 30
```

You should see: `database system is ready to accept connections`

---

## Initialize the Database

Wait ~15 seconds for PostgreSQL to fully start and run the init script, then verify:

```bash
# List all databases — you should see neuralhub_db
docker exec -it postgres psql -U postgres -c "\l"

# List all users — you should see neuralhub_user
docker exec -it postgres psql -U postgres -c "\du"

# Test connecting as the app user
docker exec -it postgres psql -U neuralhub_user -d neuralhub_db -c "SELECT current_user, current_database();"
```

Expected output for last command:
```
 current_user   | current_database
----------------+-----------------
 neuralhub_user | neuralhub_db
```

---

## Update Your Environment Variables

On your server, edit the backend `.env` file:

```bash
nano /opt/neuralhub/app/backend/.env
```

Replace only the database-related lines:

```env
# OLD (remove this Neon cloud URL):
# DATABASE_URL=postgresql://neondb_owner:npg_EaWkQ6gV0ehu@ep-orange-leaf-ad9ehro5-pooler.c-2.us-east-1.aws.neon.tech/neondb?sslmode=require&channel_binding=require

# NEW (self-hosted):
DATABASE_URL=postgresql://neuralhub_user:your-strong-password@localhost:5432/neuralhub_db

# For self-hosted, DIRECT_DATABASE_URL is the same URL (no pooler/direct endpoint distinction)
DIRECT_DATABASE_URL=postgresql://neuralhub_user:your-strong-password@localhost:5432/neuralhub_db
```

> **Critical**: Do NOT include `?sslmode=require` or `?channel_binding=require` in the self-hosted URL.
> These are Neon-specific cloud params. Adding them to a local connection will cause SSL handshake errors.

Set correct file permissions:

```bash
chmod 600 /opt/neuralhub/app/backend/.env
```

---

## Run Prisma Migrations

Apply your existing Prisma schema to the new database. Switch to the neuralhub user:

```bash
sudo su - neuralhub
cd ~/app/backend
```

Generate the Prisma client and push schema:

```bash
# Generate Prisma client (reads DATABASE_URL from .env automatically)
uv run prisma generate

# Push the full schema to your new database (creates all tables)
uv run prisma db push
```

For production (recommended — creates a tracked migration history):

```bash
uv run prisma migrate deploy
```

Verify tables were created:

```bash
docker exec -it postgres psql -U neuralhub_user -d neuralhub_db -c "\dt"
```

You should see all your Prisma tables: `ChatHistory`, `Patient`, `Pdf`, `Medication`, `PatientIntake`, etc.

---

## Add to Docker Compose

Update `/opt/neuralhub/docker/docker-compose.yml` to manage PostgreSQL alongside Qdrant and Redis:

```bash
nano /opt/neuralhub/docker/docker-compose.yml
```

Add the `postgres` service to your existing compose file:

```yaml
version: '3.8'

services:
  qdrant:
    image: qdrant/qdrant:latest
    container_name: qdrant
    restart: unless-stopped
    ports:
      - "6333:6333"
      - "6334:6334"
    volumes:
      - ./qdrant/storage:/qdrant/storage:z
      - ./qdrant/snapshots:/qdrant/snapshots:z
      - ./qdrant/config.yaml:/qdrant/config/production.yaml:ro
    environment:
      - QDRANT__SERVICE__API_KEY=${QDRANT_API_KEY}
    networks:
      - neuralhub-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:6333/"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  redis:
    image: redis:7-alpine
    container_name: redis
    restart: unless-stopped
    ports:
      - "6379:6379"
    volumes:
      - ./redis/data:/data:z
      - ./redis/redis.conf:/usr/local/etc/redis/redis.conf:ro
    command: redis-server /usr/local/etc/redis/redis.conf
    networks:
      - neuralhub-network
    healthcheck:
      test: ["CMD", "redis-cli", "--raw", "incr", "ping"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  # Self-hosted PostgreSQL
  postgres:
    image: postgres:16-alpine
    container_name: postgres
    restart: unless-stopped
    ports:
      - "127.0.0.1:5432:5432"           # Localhost only — not exposed to internet
    volumes:
      - ./postgres/data:/var/lib/postgresql/data:z       # Persistent data on host
      - ./postgres/init:/docker-entrypoint-initdb.d:ro   # Init scripts (first run only)
    environment:
      - POSTGRES_PASSWORD=${POSTGRES_ROOT_PASSWORD}
      - POSTGRES_USER=postgres
      - POSTGRES_DB=postgres
      - PGDATA=/var/lib/postgresql/data/pgdata
    networks:
      - neuralhub-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s

networks:
  neuralhub-network:
    driver: bridge

volumes:
  qdrant_storage:
  qdrant_snapshots:
  redis_data:
```

Update `/opt/neuralhub/docker/.env` to add the Postgres root password:

```bash
nano /opt/neuralhub/docker/.env
```

```env
# Qdrant Configuration
QDRANT_API_KEY=your-secure-qdrant-api-key

# Redis Configuration
REDIS_PASSWORD=your-secure-redis-password

# PostgreSQL root/superuser password (NOT used by the app — app uses neuralhub_user)
POSTGRES_ROOT_PASSWORD=your-postgres-root-password
```

```bash
chmod 600 /opt/neuralhub/docker/.env
```

Start all services with Compose:

```bash
cd /opt/neuralhub/docker
docker compose up -d
```

---

## Backup Strategy (Implement Later)

> These scripts are ready to use whenever you decide to enable automated backups. No action is needed right now.

### pg_dump Backup Script

```bash
nano /opt/neuralhub/scripts/backup-postgres.sh
```

```bash
#!/bin/bash

# PostgreSQL Backup Script
# Uses pg_dump inside the container for a consistent, crash-safe backup

set -e

BACKUP_DIR="/opt/neuralhub/backups/postgres"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=7
PG_USER="neuralhub_user"
PG_DB="neuralhub_db"

mkdir -p "$BACKUP_DIR"

echo "[$(date)] Starting PostgreSQL backup..."

# Custom format dump (best for restore, supports parallel restore)
docker exec postgres pg_dump \
  -U "$PG_USER" \
  -d "$PG_DB" \
  --format=custom \
  --compress=9 \
  > "$BACKUP_DIR/neuralhub_db_${TIMESTAMP}.dump"

echo "[$(date)] Backup complete: neuralhub_db_${TIMESTAMP}.dump"
echo "[$(date)] Size: $(du -sh $BACKUP_DIR/neuralhub_db_${TIMESTAMP}.dump | cut -f1)"

# Clean up old backups
find "$BACKUP_DIR" -name "*.dump" -mtime +$RETENTION_DAYS -delete
find "$BACKUP_DIR" -name "*.sql.gz" -mtime +$RETENTION_DAYS -delete

echo "[$(date)] PostgreSQL backup finished!"
```

```bash
chmod +x /opt/neuralhub/scripts/backup-postgres.sh
```

### Restore from Backup

```bash
# Restore from custom format dump
docker exec -i postgres pg_restore \
  -U neuralhub_user \
  -d neuralhub_db \
  --clean \
  --if-exists \
  < /opt/neuralhub/backups/postgres/neuralhub_db_TIMESTAMP.dump
```

### Add to Cron (when ready)

```bash
crontab -e
```

```cron
# Backup PostgreSQL daily at 3 AM (offset from Qdrant at 2 AM to avoid I/O contention)
0 3 * * * /opt/neuralhub/scripts/backup-postgres.sh >> /opt/neuralhub/logs/postgres-backup.log 2>&1
```

### Volume-level backup (no pg_dump — requires brief downtime)

```bash
# Stop postgres for a consistent snapshot
docker stop postgres

# Archive the entire data directory
tar -czf /opt/neuralhub/backups/postgres/postgres_volume_$(date +%Y%m%d).tar.gz \
  -C /opt/neuralhub/docker/postgres data

# Restart postgres
docker start postgres
```

---

## Verify Everything Works

### 1. Check container status

```bash
docker ps | grep postgres
```

### 2. Restart app services and health check

```bash
sudo supervisorctl restart neuralhub-api
sudo supervisorctl restart neuralhub-worker

# Wait 5 seconds then run health check
sleep 5
curl -f http://localhost:5003/health
```

### 3. Add PostgreSQL to the existing health check script

```bash
nano /opt/neuralhub/scripts/health-check.sh
```

Add these lines after the Redis section:

```bash
# Check PostgreSQL
echo "Checking PostgreSQL..."
if docker exec postgres pg_isready -U neuralhub_user -d neuralhub_db > /dev/null 2>&1; then
  echo "✅ PostgreSQL: HEALTHY"
else
  echo "❌ PostgreSQL: UNHEALTHY"
fi
```

### 4. Monitor disk usage

```bash
# Data directory size
du -sh /opt/neuralhub/docker/postgres/data

# All Docker resource usage
docker system df
```

---

## Troubleshooting

### Init script did not run (user/database not created)

The init scripts only run on the **very first start** when the data directory is empty. If you started the container before placing the init script:

```bash
# Stop and remove the container (data volume is safe)
docker stop postgres && docker rm postgres

# Clear the data directory to force re-initialization
rm -rf /opt/neuralhub/docker/postgres/data/*

# Start fresh — init script will now run
docker run -d \
  --name postgres \
  --restart unless-stopped \
  -p 127.0.0.1:5432:5432 \
  -v /opt/neuralhub/docker/postgres/data:/var/lib/postgresql/data:z \
  -v /opt/neuralhub/docker/postgres/init:/docker-entrypoint-initdb.d:ro \
  -e POSTGRES_PASSWORD=your-postgres-root-password \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_DB=postgres \
  -e PGDATA=/var/lib/postgresql/data/pgdata \
  postgres:16-alpine
```

### Cannot connect — connection refused

```bash
# Check container is running
docker ps | grep postgres

# Check logs for errors
docker logs postgres --tail 30

# Check port is listening
sudo ss -tlnp | grep 5432
```

### Authentication failed for user neuralhub_user

Verify the password in your `.env` matches what you put in the init SQL script:

```bash
# Test connection manually
docker exec -it postgres psql -U neuralhub_user -d neuralhub_db -W
# Enter the password when prompted
```

### Prisma error — SSL required or channel_binding not supported

Your `DATABASE_URL` still has Neon-specific params. Check:

```bash
grep DATABASE_URL /opt/neuralhub/app/backend/.env
```

The URL must look exactly like this (no `?` params at all):
```
DATABASE_URL=postgresql://neuralhub_user:password@localhost:5432/neuralhub_db
```

### Container restarted but data is missing

This should not happen with the volume setup. Verify the data directory is populated:

```bash
ls -la /opt/neuralhub/docker/postgres/data/pgdata/
```

If the directory is empty, the volume was not mounted correctly when the container first started. Follow the "Init script did not run" steps above, but the data **must be restored from a backup** in this case.

### Container crashes — data is always preserved

The data lives on the host at `/opt/neuralhub/docker/postgres/data/pgdata/`. Simply restart:

```bash
docker start postgres
# or
docker compose -f /opt/neuralhub/docker/docker-compose.yml up -d postgres
```

---

## Summary

After completing this setup:

| Setting | Value |
|---|---|
| Container | `postgres` (postgres:16-alpine) |
| Port | `5432` (localhost only) |
| Data volume | `/opt/neuralhub/docker/postgres/data/pgdata/` |
| App user | `neuralhub_user` |
| App database | `neuralhub_db` |
| **New DATABASE_URL** | `postgresql://neuralhub_user:your-strong-password@localhost:5432/neuralhub_db` |

Only these two lines in your `.env` need to change:

```env
DATABASE_URL=postgresql://neuralhub_user:your-strong-password@localhost:5432/neuralhub_db
DIRECT_DATABASE_URL=postgresql://neuralhub_user:your-strong-password@localhost:5432/neuralhub_db
```

Everything else — Prisma, SQLAlchemy, the background worker — picks up the new URL automatically on restart.

---

**Last Updated**: July 2026
**Maintained by**: NeuralHub Team
