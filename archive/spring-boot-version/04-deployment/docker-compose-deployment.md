# Docker Compose Deployment Guide

This document provides instructions for deploying the First Viscount microservices platform using Docker Compose, suitable for development, testing, and small-scale production deployments.

## Prerequisites

- Docker Engine 24.0+
- Docker Compose 2.20+
- 8GB RAM minimum (16GB recommended)
- 20GB available disk space

## Directory Structure

```
deployment/
├── docker-compose.yml          # Main compose file
├── docker-compose.dev.yml      # Development overrides
├── docker-compose.prod.yml     # Production overrides
├── .env                        # Environment variables
├── config/                     # Configuration files
│   ├── nginx/
│   ├── prometheus/
│   └── grafana/
├── volumes/                    # Persistent data
└── scripts/                    # Helper scripts
```

## Environment Configuration

### Base Environment File

```bash
# .env
# Infrastructure
POSTGRES_VERSION=15-alpine
KAFKA_VERSION=7.4.0
REDIS_VERSION=7-alpine
MONGO_VERSION=6

# Database
POSTGRES_USER=fvadmin
POSTGRES_PASSWORD=securepassword123
POSTGRES_DB=firstviscount

# Kafka
KAFKA_BROKER_ID=1
KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181
KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092
KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1

# Services
JWT_SECRET=your-jwt-secret-here
LOG_LEVEL=INFO
SPRING_PROFILES_ACTIVE=docker

# Monitoring
GRAFANA_ADMIN_PASSWORD=admin123
```

## Main Docker Compose File

```yaml
# docker-compose.yml
version: '3.8'

x-common-variables: &common-variables
  SPRING_PROFILES_ACTIVE: ${SPRING_PROFILES_ACTIVE}
  JWT_SECRET: ${JWT_SECRET}
  LOG_LEVEL: ${LOG_LEVEL}

x-common-service: &common-service
  restart: unless-stopped
  networks:
    - firstviscount
  logging:
    driver: "json-file"
    options:
      max-size: "10m"
      max-file: "3"

services:
  # Infrastructure Services
  
  postgres:
    <<: *common-service
    image: postgres:${POSTGRES_VERSION}
    container_name: fv-postgres
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_INITDB_ARGS: "--encoding=UTF8 --locale=en_US.UTF-8"
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./config/postgres/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

  zookeeper:
    <<: *common-service
    image: confluentinc/cp-zookeeper:${KAFKA_VERSION}
    container_name: fv-zookeeper
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    volumes:
      - zookeeper-data:/var/lib/zookeeper/data
      - zookeeper-logs:/var/lib/zookeeper/log

  kafka:
    <<: *common-service
    image: confluentinc/cp-kafka:${KAFKA_VERSION}
    container_name: fv-kafka
    depends_on:
      zookeeper:
        condition: service_started
    environment:
      KAFKA_BROKER_ID: ${KAFKA_BROKER_ID}
      KAFKA_ZOOKEEPER_CONNECT: ${KAFKA_ZOOKEEPER_CONNECT}
      KAFKA_ADVERTISED_LISTENERS: ${KAFKA_ADVERTISED_LISTENERS}
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: ${KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR}
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_JMX_PORT: 9101
      KAFKA_JMX_HOSTNAME: localhost
    volumes:
      - kafka-data:/var/lib/kafka/data
    ports:
      - "9092:9092"
      - "9101:9101"
    healthcheck:
      test: ["CMD", "kafka-topics", "--bootstrap-server", "kafka:9092", "--list"]
      interval: 30s
      timeout: 10s
      retries: 5

  redis:
    <<: *common-service
    image: redis:${REDIS_VERSION}
    container_name: fv-redis
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD:-redis123}
    volumes:
      - redis-data:/data
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  mongodb:
    <<: *common-service
    image: mongo:${MONGO_VERSION}
    container_name: fv-mongodb
    environment:
      MONGO_INITDB_ROOT_USERNAME: ${MONGO_USER:-mongoadmin}
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_PASSWORD:-mongo123}
      MONGO_INITDB_DATABASE: notification_db
    volumes:
      - mongodb-data:/data/db
    ports:
      - "27017:27017"

  # Microservices
  
  platform-coordination:
    <<: *common-service
    image: firstviscount/platform-coordination:latest
    container_name: fv-platform-coordination
    depends_on:
      postgres:
        condition: service_healthy
      kafka:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      <<: *common-variables
      SERVER_PORT: 8080
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/coordination_db
      SPRING_DATASOURCE_USERNAME: ${POSTGRES_USER}
      SPRING_DATASOURCE_PASSWORD: ${POSTGRES_PASSWORD}
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      SPRING_REDIS_HOST: redis
      SPRING_REDIS_PASSWORD: ${REDIS_PASSWORD:-redis123}
    ports:
      - "8080:8080"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  product-service:
    <<: *common-service
    image: firstviscount/product-service:latest
    container_name: fv-product-service
    depends_on:
      postgres:
        condition: service_healthy
      kafka:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      <<: *common-variables
      SERVER_PORT: 8082
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/product_db
      SPRING_DATASOURCE_USERNAME: ${POSTGRES_USER}
      SPRING_DATASOURCE_PASSWORD: ${POSTGRES_PASSWORD}
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      SPRING_REDIS_HOST: redis
      SPRING_REDIS_PASSWORD: ${REDIS_PASSWORD:-redis123}
    ports:
      - "8082:8082"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8082/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  inventory-service:
    <<: *common-service
    image: firstviscount/inventory-service:latest
    container_name: fv-inventory-service
    depends_on:
      postgres:
        condition: service_healthy
      kafka:
        condition: service_healthy
    environment:
      <<: *common-variables
      SERVER_PORT: 8083
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/inventory_db
      SPRING_DATASOURCE_USERNAME: ${POSTGRES_USER}
      SPRING_DATASOURCE_PASSWORD: ${POSTGRES_PASSWORD}
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
    ports:
      - "8083:8083"

  order-service:
    <<: *common-service
    image: firstviscount/order-service:latest
    container_name: fv-order-service
    depends_on:
      postgres:
        condition: service_healthy
      kafka:
        condition: service_healthy
      inventory-service:
        condition: service_started
    environment:
      <<: *common-variables
      SERVER_PORT: 8084
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/order_db
      SPRING_DATASOURCE_USERNAME: ${POSTGRES_USER}
      SPRING_DATASOURCE_PASSWORD: ${POSTGRES_PASSWORD}
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      INVENTORY_SERVICE_URL: http://inventory-service:8083
    ports:
      - "8084:8084"

  delivery-service:
    <<: *common-service
    image: firstviscount/delivery-service:latest
    container_name: fv-delivery-service
    depends_on:
      postgres:
        condition: service_healthy
      kafka:
        condition: service_healthy
    environment:
      <<: *common-variables
      SERVER_PORT: 8085
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/delivery_db
      SPRING_DATASOURCE_USERNAME: ${POSTGRES_USER}
      SPRING_DATASOURCE_PASSWORD: ${POSTGRES_PASSWORD}
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
    ports:
      - "8085:8085"

  notification-service:
    <<: *common-service
    image: firstviscount/notification-service:latest
    container_name: fv-notification-service
    depends_on:
      mongodb:
        condition: service_started
      kafka:
        condition: service_healthy
    environment:
      <<: *common-variables
      SERVER_PORT: 8086
      SPRING_DATA_MONGODB_URI: mongodb://${MONGO_USER:-mongoadmin}:${MONGO_PASSWORD:-mongo123}@mongodb:27017/notification_db?authSource=admin
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
    ports:
      - "8086:8086"

  # API Gateway
  
  nginx:
    <<: *common-service
    image: nginx:alpine
    container_name: fv-nginx
    volumes:
      - ./config/nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./config/nginx/conf.d:/etc/nginx/conf.d:ro
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - platform-coordination
      - product-service
      - inventory-service
      - order-service
      - delivery-service
      - notification-service

  # Monitoring Stack
  
  prometheus:
    <<: *common-service
    image: prom/prometheus:latest
    container_name: fv-prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--web.enable-lifecycle'
    volumes:
      - ./config/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    ports:
      - "9090:9090"

  grafana:
    <<: *common-service
    image: grafana/grafana:latest
    container_name: fv-grafana
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_ADMIN_PASSWORD}
      GF_USERS_ALLOW_SIGN_UP: false
    volumes:
      - ./config/grafana/provisioning:/etc/grafana/provisioning:ro
      - ./config/grafana/dashboards:/var/lib/grafana/dashboards:ro
      - grafana-data:/var/lib/grafana
    ports:
      - "3000:3000"
    depends_on:
      - prometheus

  # Logging Stack
  
  elasticsearch:
    <<: *common-service
    image: docker.elastic.co/elasticsearch/elasticsearch:8.8.0
    container_name: fv-elasticsearch
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - xpack.security.enabled=false
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"

  kibana:
    <<: *common-service
    image: docker.elastic.co/kibana/kibana:8.8.0
    container_name: fv-kibana
    environment:
      ELASTICSEARCH_HOSTS: '["http://elasticsearch:9200"]'
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch

volumes:
  postgres-data:
  zookeeper-data:
  zookeeper-logs:
  kafka-data:
  redis-data:
  mongodb-data:
  prometheus-data:
  grafana-data:
  elasticsearch-data:

networks:
  firstviscount:
    driver: bridge
```

## Development Override Configuration

```yaml
# docker-compose.dev.yml
version: '3.8'

services:
  postgres:
    ports:
      - "5432:5432"

  kafka:
    environment:
      KAFKA_LOG_RETENTION_HOURS: 1
      KAFKA_LOG_SEGMENT_BYTES: 1073741824
      KAFKA_LOG_RETENTION_CHECK_INTERVAL_MS: 300000

  # Mount source code for hot reload
  platform-coordination:
    volumes:
      - ./platform-coordination/src:/app/src:ro
    environment:
      SPRING_PROFILES_ACTIVE: dev
      SPRING_DEVTOOLS_RESTART_ENABLED: true

  product-service:
    volumes:
      - ./product-service/src:/app/src:ro
    environment:
      SPRING_PROFILES_ACTIVE: dev
      SPRING_DEVTOOLS_RESTART_ENABLED: true

  # Additional dev tools
  
  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: fv-kafka-ui
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
      KAFKA_CLUSTERS_0_ZOOKEEPER: zookeeper:2181
    ports:
      - "8090:8080"
    networks:
      - firstviscount

  mailhog:
    image: mailhog/mailhog
    container_name: fv-mailhog
    ports:
      - "1025:1025"
      - "8025:8025"
    networks:
      - firstviscount
```

## Production Override Configuration

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  postgres:
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 2G
        reservations:
          cpus: '1'
          memory: 1G
    environment:
      POSTGRES_INITDB_ARGS: "--encoding=UTF8 --locale=en_US.UTF-8 --data-checksums"
    command: >
      postgres
      -c max_connections=200
      -c shared_buffers=256MB
      -c effective_cache_size=1GB
      -c maintenance_work_mem=64MB
      -c checkpoint_completion_target=0.9
      -c wal_buffers=16MB
      -c default_statistics_target=100
      -c random_page_cost=1.1
      -c effective_io_concurrency=200

  kafka:
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '2'
          memory: 2G
    environment:
      KAFKA_NUM_PARTITIONS: 3
      KAFKA_DEFAULT_REPLICATION_FACTOR: 3
      KAFKA_MIN_INSYNC_REPLICAS: 2
      KAFKA_LOG_RETENTION_HOURS: 168

  redis:
    command: >
      redis-server
      --appendonly yes
      --requirepass ${REDIS_PASSWORD}
      --maxmemory 2gb
      --maxmemory-policy allkeys-lru

  nginx:
    volumes:
      - ./config/nginx/ssl:/etc/nginx/ssl:ro
      - ./config/nginx/nginx-prod.conf:/etc/nginx/nginx.conf:ro
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 512M

  # Production monitoring
  node-exporter:
    image: prom/node-exporter:latest
    container_name: fv-node-exporter
    restart: unless-stopped
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    ports:
      - "9100:9100"
    networks:
      - firstviscount
```

## Configuration Files

### Nginx Configuration

```nginx
# config/nginx/nginx.conf
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
    use epoll;
    multi_accept on;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;

    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;

    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml text/javascript 
               application/json application/javascript application/xml+rss 
               application/rss+xml application/atom+xml image/svg+xml;

    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
    limit_req_status 429;

    # Include server configurations
    include /etc/nginx/conf.d/*.conf;
}
```

```nginx
# config/nginx/conf.d/api-gateway.conf
upstream platform_coordination {
    server platform-coordination:8080 max_fails=3 fail_timeout=30s;
}

upstream product_service {
    server product-service:8082 max_fails=3 fail_timeout=30s;
}

upstream inventory_service {
    server inventory-service:8083 max_fails=3 fail_timeout=30s;
}

upstream order_service {
    server order-service:8084 max_fails=3 fail_timeout=30s;
}

upstream delivery_service {
    server delivery-service:8085 max_fails=3 fail_timeout=30s;
}

upstream notification_service {
    server notification-service:8086 max_fails=3 fail_timeout=30s;
}

server {
    listen 80;
    server_name api.firstviscount.local;

    # API rate limiting
    limit_req zone=api_limit burst=20 nodelay;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # Health check endpoint
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }

    # Platform coordination service
    location /api/v1/platform/ {
        proxy_pass http://platform_coordination/api/v1/platform/;
        include /etc/nginx/conf.d/proxy-params.conf;
    }

    # Product service
    location /api/v1/products {
        proxy_pass http://product_service/api/v1/products;
        include /etc/nginx/conf.d/proxy-params.conf;
    }

    # Inventory service
    location /api/v1/inventory {
        proxy_pass http://inventory_service/api/v1/inventory;
        include /etc/nginx/conf.d/proxy-params.conf;
    }

    # Order service
    location /api/v1/orders {
        proxy_pass http://order_service/api/v1/orders;
        include /etc/nginx/conf.d/proxy-params.conf;
    }

    # Delivery service
    location /api/v1/delivery {
        proxy_pass http://delivery_service/api/v1/delivery;
        include /etc/nginx/conf.d/proxy-params.conf;
    }

    # Notification service
    location /api/v1/notifications {
        proxy_pass http://notification_service/api/v1/notifications;
        include /etc/nginx/conf.d/proxy-params.conf;
    }
}
```

```nginx
# config/nginx/conf.d/proxy-params.conf
proxy_set_header Host $http_host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Request-ID $request_id;

proxy_connect_timeout 30s;
proxy_send_timeout 30s;
proxy_read_timeout 30s;

proxy_buffer_size 4k;
proxy_buffers 8 4k;
proxy_busy_buffers_size 8k;

proxy_http_version 1.1;
proxy_set_header Connection "";
```

### Prometheus Configuration

```yaml
# config/prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'firstviscount-services'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets:
          - 'platform-coordination:8080'
          - 'product-service:8082'
          - 'inventory-service:8083'
          - 'order-service:8084'
          - 'delivery-service:8085'
          - 'notification-service:8086'

  - job_name: 'kafka'
    static_configs:
      - targets: ['kafka:9101']

  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres-exporter:9187']

  - job_name: 'redis'
    static_configs:
      - targets: ['redis-exporter:9121']

  - job_name: 'nginx'
    static_configs:
      - targets: ['nginx-exporter:9113']
```

## Deployment Scripts

### Start Script

```bash
#!/bin/bash
# scripts/start.sh

set -e

ENVIRONMENT=${1:-dev}

echo "Starting First Viscount in $ENVIRONMENT mode..."

# Load environment variables
if [ -f .env.$ENVIRONMENT ]; then
    export $(cat .env.$ENVIRONMENT | grep -v '^#' | xargs)
fi

# Pull latest images
docker-compose pull

# Start infrastructure first
docker-compose up -d postgres kafka redis mongodb

# Wait for infrastructure
echo "Waiting for infrastructure..."
./scripts/wait-for-it.sh localhost:5432 -t 60
./scripts/wait-for-it.sh localhost:9092 -t 60
./scripts/wait-for-it.sh localhost:6379 -t 60

# Run database migrations
echo "Running database migrations..."
./scripts/migrate.sh

# Start services
if [ "$ENVIRONMENT" = "prod" ]; then
    docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d
else
    docker-compose -f docker-compose.yml -f docker-compose.dev.yml up -d
fi

echo "First Viscount is starting..."
echo "Access the application at: http://localhost"
echo "Access Grafana at: http://localhost:3000"
echo "Access Kibana at: http://localhost:5601"
```

### Health Check Script

```bash
#!/bin/bash
# scripts/health-check.sh

SERVICES=(
    "platform-coordination:8080"
    "product-service:8082"
    "inventory-service:8083"
    "order-service:8084"
    "delivery-service:8085"
    "notification-service:8086"
)

echo "Checking service health..."

for service in "${SERVICES[@]}"; do
    name="${service%%:*}"
    port="${service#*:}"
    
    if curl -f -s "http://localhost:$port/actuator/health" > /dev/null; then
        echo "✓ $name is healthy"
    else
        echo "✗ $name is not responding"
    fi
done
```

### Backup Script

```bash
#!/bin/bash
# scripts/backup.sh

BACKUP_DIR="./backups/$(date +%Y%m%d_%H%M%S)"
mkdir -p "$BACKUP_DIR"

echo "Starting backup..."

# Backup PostgreSQL
docker exec fv-postgres pg_dumpall -U fvadmin > "$BACKUP_DIR/postgres_backup.sql"

# Backup MongoDB
docker exec fv-mongodb mongodump --out /backup
docker cp fv-mongodb:/backup "$BACKUP_DIR/mongodb_backup"

# Backup volumes
docker run --rm -v firstviscount_postgres-data:/data -v "$BACKUP_DIR":/backup alpine tar czf /backup/postgres-data.tar.gz -C /data .
docker run --rm -v firstviscount_redis-data:/data -v "$BACKUP_DIR":/backup alpine tar czf /backup/redis-data.tar.gz -C /data .

echo "Backup completed: $BACKUP_DIR"
```

## Monitoring and Logging

### Grafana Dashboard Import

```json
{
  "dashboard": {
    "title": "First Viscount Services",
    "panels": [
      {
        "title": "Request Rate",
        "targets": [
          {
            "expr": "sum(rate(http_server_requests_seconds_count[5m])) by (service)"
          }
        ]
      },
      {
        "title": "Error Rate",
        "targets": [
          {
            "expr": "sum(rate(http_server_requests_seconds_count{status=~\"5..\"}[5m])) by (service)"
          }
        ]
      },
      {
        "title": "Response Time",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, sum(rate(http_server_requests_seconds_bucket[5m])) by (service, le))"
          }
        ]
      }
    ]
  }
}
```

## Production Deployment Checklist

- [ ] Update all service images to production versions
- [ ] Configure proper secrets in `.env.prod`
- [ ] Set up SSL certificates for nginx
- [ ] Configure backup strategy
- [ ] Set up monitoring alerts
- [ ] Configure log aggregation
- [ ] Test disaster recovery procedures
- [ ] Set up health checks and auto-restart policies
- [ ] Configure resource limits for all services
- [ ] Document deployment procedures
- [ ] Set up automated deployment pipeline

## Troubleshooting

### Common Issues

1. **Services not starting**
   ```bash
   # Check logs
   docker-compose logs -f service-name
   
   # Check container status
   docker-compose ps
   ```

2. **Database connection issues**
   ```bash
   # Check database is running
   docker exec fv-postgres pg_isready
   
   # Check network connectivity
   docker exec service-name ping postgres
   ```

3. **Kafka issues**
   ```bash
   # List topics
   docker exec fv-kafka kafka-topics --list --bootstrap-server localhost:9092
   
   # Check consumer groups
   docker exec fv-kafka kafka-consumer-groups --list --bootstrap-server localhost:9092
   ```

---
*Last Updated: January 2025*