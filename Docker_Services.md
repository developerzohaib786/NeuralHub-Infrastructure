# Docker Services Setup - Qdrant & Redis

Complete guide for setting up self-hosted Qdrant Vector Database and Redis with Docker, including volume management, backup strategies, and production-ready configuration.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Docker Installation](#docker-installation)
3. [Qdrant Vector Database Setup](#qdrant-vector-database-setup)
4. [Redis Setup](#redis-setup)
5. [Docker Compose Configuration](#docker-compose-configuration)
6. [Backup Strategies](#backup-strategies)
7. [Monitoring & Maintenance](#monitoring--maintenance)
8. [Security Configuration](#security-configuration)
9. [Integration with NeuralHub](#integration-with-neuralhub)
10. [Troubleshooting](#troubleshooting)

---

## Prerequisites

### Server Requirements
- Ubuntu Server 20.04 LTS or newer (22.04 recommended)
- Minimum 4GB RAM (8GB recommended for production)
- 50GB+ storage (SSD recommended)
- Root or sudo access

### Software Requirements
- Docker Engine 20.10+
- Docker Compose v2.0+

---

## Docker Installation

### 1. Install Docker Engine

SSH into your Ubuntu server:

```bash
ssh username@your-server-ip
```

Update system packages:

```bash
sudo apt update && sudo apt upgrade -y
```

Install Docker:

```bash
# Remove old versions (if any)
sudo apt remove docker docker-engine docker.io containerd runc

# Install prerequisites
sudo apt install -y ca-certificates curl gnupg lsb-release

# Add Docker's official GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Set up Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Verify installation
sudo docker --version
sudo docker compose version
```

### 2. Configure Docker

Add your user to the docker group (optional, allows running docker without sudo):

```bash
sudo usermod -aG docker $USER
newgrp docker
```

Start and enable Docker service:

```bash
sudo systemctl start docker
sudo systemctl enable docker
sudo systemctl status docker
```

Test Docker installation:

```bash
docker run hello-world
```

---

## Qdrant Vector Database Setup

### 1. Create Directory Structure

```bash
# Create directories for Qdrant data and backups
sudo mkdir -p /opt/neuralhub/docker/qdrant/storage
sudo mkdir -p /opt/neuralhub/docker/qdrant/snapshots
sudo mkdir -p /opt/neuralhub/backups/qdrant
sudo chown -R $USER:$USER /opt/neuralhub/docker/qdrant
sudo chown -R $USER:$USER /opt/neuralhub/backups/qdrant
```

### 2. Create Qdrant Configuration File

Create a configuration file for Qdrant:

```bash
nano /opt/neuralhub/docker/qdrant/config.yaml
```

Add the following configuration:

```yaml
service:
  host: 0.0.0.0
  http_port: 6333
  grpc_port: 6334

storage:
  # Storage path for collections
  storage_path: /qdrant/storage
  
  # Snapshots path
  snapshots_path: /qdrant/snapshots
  
  # Performance tuning
  optimizers:
    # Default segment size in KB
    default_segment_number: 0
    
  # Enable Write-Ahead Log
  wal:
    wal_capacity_mb: 32
    wal_segments_ahead: 0

  # Limit the number of concurrent index jobs
  indexing_threshold: 20000

# Security - API Key authentication
service:
  api_key: "your-secure-api-key-change-this-in-production"

# Logging
log_level: INFO
```

**Important**: Replace `your-secure-api-key-change-this-in-production` with a strong random API key.

Generate a secure API key:

```bash
# Generate a random API key
openssl rand -base64 32
```

### 3. Run Qdrant Container

```bash
docker run -d \
  --name qdrant \
  --restart unless-stopped \
  -p 6333:6333 \
  -p 6334:6334 \
  -v /opt/neuralhub/docker/qdrant/storage:/qdrant/storage:z \
  -v /opt/neuralhub/docker/qdrant/snapshots:/qdrant/snapshots:z \
  -v /opt/neuralhub/docker/qdrant/config.yaml:/qdrant/config/production.yaml:ro \
  -e QDRANT__SERVICE__API_KEY="your-secure-api-key-change-this-in-production" \
  qdrant/qdrant:latest
```

### 4. Verify Qdrant Installation

Check if Qdrant is running:

```bash
docker ps | grep qdrant
```

Test Qdrant API (without API key for initial test):

```bash
curl http://localhost:6333/
```

Expected response:
```json
{
  "title": "qdrant - vector search engine",
  "version": "1.x.x"
}
```

Test with API key:

```bash
curl -H "api-key: your-secure-api-key-change-this-in-production" http://localhost:6333/collections
```

---

## Redis Setup

### 1. Create Directory Structure

```bash
# Create directories for Redis data and backups
sudo mkdir -p /opt/neuralhub/docker/redis/data
sudo mkdir -p /opt/neuralhub/backups/redis
sudo chown -R $USER:$USER /opt/neuralhub/docker/redis
sudo chown -R $USER:$USER /opt/neuralhub/backups/redis
```

### 2. Create Redis Configuration File

```bash
nano /opt/neuralhub/docker/redis/redis.conf
```

Add the following configuration:

```conf
# Network
bind 0.0.0.0
protected-mode yes
port 6379

# Authentication
requirepass your-secure-redis-password-change-this

# Persistence - RDB (Snapshots)
save 900 1      # Save after 900 seconds if at least 1 key changed
save 300 10     # Save after 300 seconds if at least 10 keys changed
save 60 10000   # Save after 60 seconds if at least 10000 keys changed

stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb
dir /data

# Persistence - AOF (Append Only File)
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# Memory Management
maxmemory 2gb
maxmemory-policy allkeys-lru

# Logging
loglevel notice
logfile ""

# Slow log
slowlog-log-slower-than 10000
slowlog-max-len 128

# Limits
maxclients 10000
timeout 300

# TLS/SSL (optional, for production security)
# tls-port 6380
# tls-cert-file /tls/redis.crt
# tls-key-file /tls/redis.key
# tls-ca-cert-file /tls/ca.crt
```

**Important**: Replace `your-secure-redis-password-change-this` with a strong password.

Generate a secure password:

```bash
# Generate a random password
openssl rand -base64 32
```

### 3. Run Redis Container

```bash
docker run -d \
  --name redis \
  --restart unless-stopped \
  -p 6379:6379 \
  -v /opt/neuralhub/docker/redis/data:/data:z \
  -v /opt/neuralhub/docker/redis/redis.conf:/usr/local/etc/redis/redis.conf:ro \
  redis:7-alpine redis-server /usr/local/etc/redis/redis.conf
```

### 4. Verify Redis Installation

Check if Redis is running:

```bash
docker ps | grep redis
```

Test Redis connection:

```bash
# Connect to Redis CLI
docker exec -it redis redis-cli

# Authenticate
AUTH your-secure-redis-password-change-this

# Test commands
PING
SET test "Hello Redis"
GET test
INFO server
EXIT
```

---

## Docker Compose Configuration

For easier management, create a Docker Compose file to manage both services.

### 1. Create Docker Compose File

```bash
nano /opt/neuralhub/docker/docker-compose.yml
```

Add the following configuration:

```yaml
version: '3.8'

services:
  qdrant:
    image: qdrant/qdrant:latest
    container_name: qdrant
    restart: unless-stopped
    ports:
      - "6333:6333"  # HTTP API
      - "6334:6334"  # gRPC API
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

networks:
  neuralhub-network:
    driver: bridge

volumes:
  qdrant_storage:
  qdrant_snapshots:
  redis_data:
```

### 2. Create Environment File

```bash
nano /opt/neuralhub/docker/.env
```

Add your credentials:

```env
# Qdrant Configuration
QDRANT_API_KEY=your-secure-qdrant-api-key

# Redis Configuration
REDIS_PASSWORD=your-secure-redis-password
```

**Security Note**: Set proper file permissions:

```bash
chmod 600 /opt/neuralhub/docker/.env
```

### 3. Start Services with Docker Compose

```bash
cd /opt/neuralhub/docker
docker compose up -d
```

### 4. Manage Services

```bash
# Check status
docker compose ps

# View logs
docker compose logs -f

# Stop services
docker compose stop

# Start services
docker compose start

# Restart services
docker compose restart

# Remove services (keeps volumes)
docker compose down

# Remove services and volumes (⚠️ deletes all data)
docker compose down -v
```

---

## Backup Strategies

### Qdrant Backup

#### 1. Create Qdrant Backup Script

```bash
nano /opt/neuralhub/scripts/backup-qdrant.sh
```

Add the following script:

```bash
#!/bin/bash

# Qdrant Backup Script
# Backs up Qdrant vector database using snapshots

set -e

# Configuration
QDRANT_API_KEY="your-secure-qdrant-api-key"
QDRANT_URL="http://localhost:6333"
BACKUP_DIR="/opt/neuralhub/backups/qdrant"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=7

# Create backup directory if it doesn't exist
mkdir -p "$BACKUP_DIR"

echo "[$(date)] Starting Qdrant backup..."

# Get list of all collections
COLLECTIONS=$(curl -s -H "api-key: $QDRANT_API_KEY" \
  "$QDRANT_URL/collections" | jq -r '.result.collections[].name')

if [ -z "$COLLECTIONS" ]; then
  echo "[$(date)] No collections found"
  exit 0
fi

# Backup each collection
for COLLECTION in $COLLECTIONS; do
  echo "[$(date)] Creating snapshot for collection: $COLLECTION"
  
  # Create snapshot
  SNAPSHOT_NAME=$(curl -s -X POST \
    -H "api-key: $QDRANT_API_KEY" \
    "$QDRANT_URL/collections/$COLLECTION/snapshots" | jq -r '.result.name')
  
  if [ -z "$SNAPSHOT_NAME" ] || [ "$SNAPSHOT_NAME" = "null" ]; then
    echo "[$(date)] Failed to create snapshot for $COLLECTION"
    continue
  fi
  
  echo "[$(date)] Snapshot created: $SNAPSHOT_NAME"
  
  # Download snapshot
  curl -s -H "api-key: $QDRANT_API_KEY" \
    "$QDRANT_URL/collections/$COLLECTION/snapshots/$SNAPSHOT_NAME" \
    -o "$BACKUP_DIR/${COLLECTION}_${TIMESTAMP}.snapshot"
  
  echo "[$(date)] Snapshot downloaded: ${COLLECTION}_${TIMESTAMP}.snapshot"
done

# Also backup the entire storage directory
echo "[$(date)] Creating full storage backup..."
tar -czf "$BACKUP_DIR/qdrant_storage_${TIMESTAMP}.tar.gz" \
  -C /opt/neuralhub/docker/qdrant storage

# Clean up old backups (older than RETENTION_DAYS)
echo "[$(date)] Cleaning up old backups..."
find "$BACKUP_DIR" -name "*.snapshot" -mtime +$RETENTION_DAYS -delete
find "$BACKUP_DIR" -name "*.tar.gz" -mtime +$RETENTION_DAYS -delete

echo "[$(date)] Qdrant backup completed successfully!"
```

Make the script executable:

```bash
chmod +x /opt/neuralhub/scripts/backup-qdrant.sh
```

#### 2. Test Qdrant Backup

```bash
/opt/neuralhub/scripts/backup-qdrant.sh
```

Check backups:

```bash
ls -lh /opt/neuralhub/backups/qdrant/
```

### Redis Backup

#### 1. Create Redis Backup Script

```bash
nano /opt/neuralhub/scripts/backup-redis.sh
```

Add the following script:

```bash
#!/bin/bash

# Redis Backup Script
# Backs up Redis data using RDB and AOF files

set -e

# Configuration
BACKUP_DIR="/opt/neuralhub/backups/redis"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=7
REDIS_PASSWORD="your-secure-redis-password"

# Create backup directory if it doesn't exist
mkdir -p "$BACKUP_DIR"

echo "[$(date)] Starting Redis backup..."

# Trigger BGSAVE to create a new RDB snapshot
docker exec redis redis-cli -a "$REDIS_PASSWORD" BGSAVE

# Wait for BGSAVE to complete
sleep 5

# Copy RDB file
echo "[$(date)] Copying RDB snapshot..."
docker cp redis:/data/dump.rdb "$BACKUP_DIR/redis_${TIMESTAMP}.rdb"

# Copy AOF file if it exists
if docker exec redis test -f /data/appendonly.aof; then
  echo "[$(date)] Copying AOF file..."
  docker cp redis:/data/appendonly.aof "$BACKUP_DIR/redis_${TIMESTAMP}.aof"
fi

# Create compressed archive of entire data directory
echo "[$(date)] Creating full data backup..."
tar -czf "$BACKUP_DIR/redis_data_${TIMESTAMP}.tar.gz" \
  -C /opt/neuralhub/docker/redis data

# Clean up old backups (older than RETENTION_DAYS)
echo "[$(date)] Cleaning up old backups..."
find "$BACKUP_DIR" -name "redis_*.rdb" -mtime +$RETENTION_DAYS -delete
find "$BACKUP_DIR" -name "redis_*.aof" -mtime +$RETENTION_DAYS -delete
find "$BACKUP_DIR" -name "*.tar.gz" -mtime +$RETENTION_DAYS -delete

echo "[$(date)] Redis backup completed successfully!"
```

Make the script executable:

```bash
chmod +x /opt/neuralhub/scripts/backup-redis.sh
```

#### 2. Test Redis Backup

```bash
/opt/neuralhub/scripts/backup-redis.sh
```

Check backups:

```bash
ls -lh /opt/neuralhub/backups/redis/
```

### Combined Backup Script

Create a master backup script that backs up both services:

```bash
nano /opt/neuralhub/scripts/backup-all.sh
```

```bash
#!/bin/bash

# Master Backup Script
# Backs up all Docker services

set -e

echo "=========================================="
echo "Starting NeuralHub Services Backup"
echo "Time: $(date)"
echo "=========================================="

# Backup Qdrant
/opt/neuralhub/scripts/backup-qdrant.sh

echo ""

# Backup Redis
/opt/neuralhub/scripts/backup-redis.sh

echo ""
echo "=========================================="
echo "All backups completed successfully!"
echo "=========================================="
```

Make it executable:

```bash
chmod +x /opt/neuralhub/scripts/backup-all.sh
```

### Schedule Automated Backups

Set up cron jobs for automated backups:

```bash
crontab -e
```

Add the following lines:

```cron
# Backup Qdrant and Redis daily at 2 AM
0 2 * * * /opt/neuralhub/scripts/backup-all.sh >> /opt/neuralhub/logs/backup.log 2>&1

# Backup Qdrant every 6 hours
0 */6 * * * /opt/neuralhub/scripts/backup-qdrant.sh >> /opt/neuralhub/logs/qdrant-backup.log 2>&1

# Backup Redis every 4 hours
0 */4 * * * /opt/neuralhub/scripts/backup-redis.sh >> /opt/neuralhub/logs/redis-backup.log 2>&1
```

Create log directory if it doesn't exist:

```bash
mkdir -p /opt/neuralhub/logs
```

### Restore from Backup

#### Restore Qdrant

```bash
# Stop Qdrant
docker stop qdrant

# Restore storage from backup
cd /opt/neuralhub/docker/qdrant
rm -rf storage/*
tar -xzf /opt/neuralhub/backups/qdrant/qdrant_storage_TIMESTAMP.tar.gz

# Or restore specific collection snapshot
# Copy snapshot to snapshots directory and use Qdrant API to recover

# Start Qdrant
docker start qdrant
```

#### Restore Redis

```bash
# Stop Redis
docker stop redis

# Restore RDB file
docker cp /opt/neuralhub/backups/redis/redis_TIMESTAMP.rdb redis:/data/dump.rdb

# Start Redis
docker start redis

# Verify data
docker exec -it redis redis-cli -a "your-password" DBSIZE
```

---

## Monitoring & Maintenance

### Monitor Container Health

```bash
# Check container status
docker ps

# View resource usage
docker stats

# Check logs
docker logs qdrant --tail 100
docker logs redis --tail 100

# Follow logs in real-time
docker logs -f qdrant
docker logs -f redis
```

### Health Check Scripts

Create health check script:

```bash
nano /opt/neuralhub/scripts/health-check.sh
```

```bash
#!/bin/bash

echo "=== Docker Services Health Check ==="
echo ""

# Check Qdrant
echo "Checking Qdrant..."
QDRANT_STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:6333/)
if [ "$QDRANT_STATUS" = "200" ]; then
  echo "✅ Qdrant: HEALTHY"
else
  echo "❌ Qdrant: UNHEALTHY (HTTP $QDRANT_STATUS)"
fi

# Check Redis
echo "Checking Redis..."
REDIS_STATUS=$(docker exec redis redis-cli -a "your-password" PING 2>/dev/null)
if [ "$REDIS_STATUS" = "PONG" ]; then
  echo "✅ Redis: HEALTHY"
else
  echo "❌ Redis: UNHEALTHY"
fi

echo ""
echo "=== Container Status ==="
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

Make it executable:

```bash
chmod +x /opt/neuralhub/scripts/health-check.sh
```

Run health check:

```bash
/opt/neuralhub/scripts/health-check.sh
```

### Monitor Disk Usage

```bash
# Check Docker disk usage
docker system df

# Check volume sizes
du -sh /opt/neuralhub/docker/qdrant/storage
du -sh /opt/neuralhub/docker/redis/data

# Check backup sizes
du -sh /opt/neuralhub/backups/*
```

### Clean Up Old Data

```bash
# Remove unused Docker resources
docker system prune -a --volumes

# Clean up old logs
find /opt/neuralhub/logs -name "*.log" -mtime +30 -delete
```

---

## Security Configuration

### 1. Firewall Configuration

Only allow local connections by default:

```bash
# If using UFW
sudo ufw deny 6333  # Qdrant HTTP
sudo ufw deny 6334  # Qdrant gRPC
sudo ufw deny 6379  # Redis

# Allow only from application server (if on different machine)
# sudo ufw allow from <app-server-ip> to any port 6333
# sudo ufw allow from <app-server-ip> to any port 6379
```
