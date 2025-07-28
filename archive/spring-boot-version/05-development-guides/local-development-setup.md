# Local Development Setup Guide

This document provides comprehensive instructions for setting up a local development environment for the First Viscount microservices platform.

## Prerequisites

### Required Software

- **Java Development Kit (JDK) 21**
  - Download from [Adoptium](https://adoptium.net/) or [Oracle](https://www.oracle.com/java/technologies/downloads/)
  - Verify: `java -version`

- **Maven 3.9+**
  - Download from [Apache Maven](https://maven.apache.org/download.cgi)
  - Verify: `mvn -version`

- **Docker Desktop**
  - [Docker Desktop for Mac](https://docs.docker.com/desktop/mac/install/)
  - [Docker Desktop for Windows](https://docs.docker.com/desktop/windows/install/)
  - [Docker Engine for Linux](https://docs.docker.com/engine/install/)
  - Verify: `docker --version` and `docker-compose --version`

- **Git**
  - Download from [git-scm.com](https://git-scm.com/downloads)
  - Verify: `git --version`

### Recommended IDE

- **IntelliJ IDEA** (Ultimate or Community)
  - Download from [JetBrains](https://www.jetbrains.com/idea/download/)
  - Install plugins:
    - Spring Boot
    - Lombok
    - Docker
    - Kubernetes

- **Visual Studio Code** (Alternative)
  - Download from [code.visualstudio.com](https://code.visualstudio.com/)
  - Install extensions:
    - Extension Pack for Java
    - Spring Boot Extension Pack
    - Docker
    - Kubernetes

### Development Tools

```bash
# Install development tools on macOS
brew install kubectl k9s httpie jq yq watch

# Install development tools on Ubuntu/Debian
sudo apt-get update
sudo apt-get install -y kubectl k9s httpie jq yq

# Install development tools on Windows (using Chocolatey)
choco install kubernetes-cli k9s httpie jq yq
```

## Project Setup

### 1. Clone Repository

```bash
# Clone the repository
git clone https://github.com/firstviscount/platform.git
cd platform

# Create your feature branch
git checkout -b feature/your-feature-name
```

### 2. Environment Configuration

Create local environment files:

```bash
# Create .env.local file
cat > .env.local << EOF
# Database
POSTGRES_USER=fvadmin
POSTGRES_PASSWORD=localpass123
POSTGRES_DB=firstviscount

# Redis
REDIS_PASSWORD=redislocal123

# MongoDB
MONGO_USER=mongoadmin
MONGO_PASSWORD=mongolocal123

# JWT
JWT_SECRET=local-jwt-secret-key-change-in-production

# Spring Profiles
SPRING_PROFILES_ACTIVE=local

# Kafka
KAFKA_BOOTSTRAP_SERVERS=localhost:9092

# Service URLs
INVENTORY_SERVICE_URL=http://localhost:8083
ORDER_SERVICE_URL=http://localhost:8084
EOF

# Create application-local.yml for each service
for service in platform-coordination product-service inventory-service order-service delivery-service notification-service; do
  mkdir -p $service/src/main/resources
  cat > $service/src/main/resources/application-local.yml << EOF
spring:
  profiles:
    active: local
  
  datasource:
    url: jdbc:postgresql://localhost:5432/${service/_/-}_db
    username: fvadmin
    password: localpass123
  
  kafka:
    bootstrap-servers: localhost:9092
  
  redis:
    host: localhost
    port: 6379
    password: redislocal123

logging:
  level:
    com.firstviscount: DEBUG
    org.springframework.web: DEBUG
  pattern:
    console: "%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n"

management:
  endpoints:
    web:
      exposure:
        include: "*"
  endpoint:
    health:
      show-details: always
EOF
done
```

### 3. Infrastructure Setup

Start local infrastructure using Docker Compose:

```bash
# Create docker-compose.local.yml
cat > docker-compose.local.yml << 'EOF'
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    container_name: fv-postgres-local
    environment:
      POSTGRES_USER: fvadmin
      POSTGRES_PASSWORD: localpass123
      POSTGRES_DB: firstviscount
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./scripts/init-db.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U fvadmin"]
      interval: 10s
      timeout: 5s
      retries: 5

  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    container_name: fv-zookeeper-local
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - "2181:2181"

  kafka:
    image: confluentinc/cp-kafka:7.4.0
    container_name: fv-kafka-local
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
      - "9101:9101"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_JMX_PORT: 9101
      KAFKA_JMX_HOSTNAME: localhost

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: fv-kafka-ui-local
    depends_on:
      - kafka
    ports:
      - "8090:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:29092
      KAFKA_CLUSTERS_0_ZOOKEEPER: zookeeper:2181

  redis:
    image: redis:7-alpine
    container_name: fv-redis-local
    command: redis-server --requirepass redislocal123
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data

  mongodb:
    image: mongo:6
    container_name: fv-mongodb-local
    environment:
      MONGO_INITDB_ROOT_USERNAME: mongoadmin
      MONGO_INITDB_ROOT_PASSWORD: mongolocal123
      MONGO_INITDB_DATABASE: notification_db
    ports:
      - "27017:27017"
    volumes:
      - mongodb-data:/data/db

  mailhog:
    image: mailhog/mailhog
    container_name: fv-mailhog-local
    ports:
      - "1025:1025"  # SMTP server
      - "8025:8025"  # Web UI

volumes:
  postgres-data:
  redis-data:
  mongodb-data:
EOF

# Start infrastructure
docker-compose -f docker-compose.local.yml up -d

# Wait for services to be ready
echo "Waiting for services to start..."
sleep 10

# Verify services are running
docker-compose -f docker-compose.local.yml ps
```

### 4. Database Setup

Initialize databases:

```sql
-- scripts/init-db.sql
-- Create databases for each service
CREATE DATABASE platform_coordination_db;
CREATE DATABASE product_db;
CREATE DATABASE inventory_db;
CREATE DATABASE order_db;
CREATE DATABASE delivery_db;

-- Grant privileges
GRANT ALL PRIVILEGES ON DATABASE platform_coordination_db TO fvadmin;
GRANT ALL PRIVILEGES ON DATABASE product_db TO fvadmin;
GRANT ALL PRIVILEGES ON DATABASE inventory_db TO fvadmin;
GRANT ALL PRIVILEGES ON DATABASE order_db TO fvadmin;
GRANT ALL PRIVILEGES ON DATABASE delivery_db TO fvadmin;
```

```bash
# Run database initialization
docker exec -i fv-postgres-local psql -U fvadmin -d firstviscount < scripts/init-db.sql

# Create Kafka topics
docker exec fv-kafka-local kafka-topics --create --topic product-events --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1
docker exec fv-kafka-local kafka-topics --create --topic order-events --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1
docker exec fv-kafka-local kafka-topics --create --topic inventory-events --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1
docker exec fv-kafka-local kafka-topics --create --topic delivery-events --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1
docker exec fv-kafka-local kafka-topics --create --topic notification-events --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1
```

## Running Services

### 1. Build All Services

```bash
# Build all services
mvn clean install -DskipTests

# Or build with tests
mvn clean install
```

### 2. Run Individual Services

#### Option A: Using Maven Spring Boot Plugin

```bash
# Terminal 1: Platform Coordination Service
cd platform-coordination
mvn spring-boot:run -Dspring-boot.run.profiles=local

# Terminal 2: Product Service
cd product-service
mvn spring-boot:run -Dspring-boot.run.profiles=local

# Terminal 3: Inventory Service
cd inventory-service
mvn spring-boot:run -Dspring-boot.run.profiles=local

# Terminal 4: Order Service
cd order-service
mvn spring-boot:run -Dspring-boot.run.profiles=local

# Terminal 5: Delivery Service
cd delivery-service
mvn spring-boot:run -Dspring-boot.run.profiles=local

# Terminal 6: Notification Service
cd notification-service
mvn spring-boot:run -Dspring-boot.run.profiles=local
```

#### Option B: Using JAR Files

```bash
# Build JARs
mvn clean package

# Run services
java -jar platform-coordination/target/platform-coordination-1.0.0.jar --spring.profiles.active=local
java -jar product-service/target/product-service-1.0.0.jar --spring.profiles.active=local
java -jar inventory-service/target/inventory-service-1.0.0.jar --spring.profiles.active=local
java -jar order-service/target/order-service-1.0.0.jar --spring.profiles.active=local
java -jar delivery-service/target/delivery-service-1.0.0.jar --spring.profiles.active=local
java -jar notification-service/target/notification-service-1.0.0.jar --spring.profiles.active=local
```

#### Option C: Using IDE

**IntelliJ IDEA:**
1. Import project as Maven project
2. Enable annotation processing for Lombok
3. Create Run Configurations for each service
4. Set Active Profile: `local`
5. Set Environment Variables from `.env.local`
6. Run services using the green play button

**VS Code:**
1. Open project folder
2. Install Java Extension Pack
3. Open Java Projects view
4. Right-click on main class → Run Java
5. Add VM arguments: `-Dspring.profiles.active=local`

### 3. Run All Services with Script

```bash
#!/bin/bash
# scripts/start-all-services.sh

# Array of services
services=(
  "platform-coordination:8080"
  "product-service:8082"
  "inventory-service:8083"
  "order-service:8084"
  "delivery-service:8085"
  "notification-service:8086"
)

# Function to start a service
start_service() {
  local service_name="${1%:*}"
  local service_port="${1#*:}"
  
  echo "Starting $service_name on port $service_port..."
  cd "$service_name" || exit
  mvn spring-boot:run -Dspring-boot.run.profiles=local -Dspring-boot.run.jvmArguments="-Xmx512m" > "../logs/$service_name.log" 2>&1 &
  echo $! > "../pids/$service_name.pid"
  cd ..
}

# Create directories
mkdir -p logs pids

# Start all services
for service in "${services[@]}"; do
  start_service "$service"
done

echo "All services started. Check logs/ directory for output."
echo "To stop all services, run: ./scripts/stop-all-services.sh"
```

```bash
#!/bin/bash
# scripts/stop-all-services.sh

# Stop all services
for pid_file in pids/*.pid; do
  if [ -f "$pid_file" ]; then
    pid=$(cat "$pid_file")
    echo "Stopping process $pid..."
    kill "$pid" 2>/dev/null
    rm "$pid_file"
  fi
done

echo "All services stopped."
```

## Development Workflow

### 1. Hot Reload Configuration

Enable Spring DevTools for automatic restart:

```xml
<!-- Add to each service's pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <scope>runtime</scope>
    <optional>true</optional>
</dependency>
```

### 2. Database Migrations

Using Liquibase:

```yaml
# src/main/resources/db/changelog/db.changelog-master.yaml
databaseChangeLog:
  - include:
      file: db/changelog/changes/001-initial-schema.sql
  - include:
      file: db/changelog/changes/002-add-indexes.sql
```

### 3. API Testing

```bash
# Test product service
http GET localhost:8082/api/v1/products

# Create a product
http POST localhost:8082/api/v1/products \
  name="Test Product" \
  price=99.99 \
  categoryId=1

# Test with authentication
http GET localhost:8082/api/v1/products \
  Authorization:"Bearer $TOKEN"
```

### 4. Debugging

#### IntelliJ IDEA Debug Configuration:
1. Click on the line number to add breakpoint
2. Right-click on main class → Debug
3. Use Debug tool window to step through code

#### VS Code Debug Configuration:
```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "java",
      "name": "Product Service",
      "request": "launch",
      "mainClass": "com.firstviscount.product.ProductServiceApplication",
      "projectName": "product-service",
      "args": "--spring.profiles.active=local",
      "envFile": "${workspaceFolder}/.env.local"
    }
  ]
}
```

### 5. Testing

```bash
# Run unit tests for a specific service
cd product-service
mvn test

# Run integration tests
mvn verify -P integration-test

# Run specific test class
mvn test -Dtest=ProductServiceTest

# Run with coverage
mvn test jacoco:report
# Open target/site/jacoco/index.html
```

## Monitoring and Debugging

### 1. Service Health Checks

```bash
# Check all services health
for port in 8080 8082 8083 8084 8085 8086; do
  echo "Service on port $port:"
  curl -s http://localhost:$port/actuator/health | jq .
done
```

### 2. Logs

```bash
# View logs for a specific service
tail -f logs/product-service.log

# View all logs
tail -f logs/*.log

# Filter logs by level
grep "ERROR" logs/product-service.log

# View Docker logs
docker-compose -f docker-compose.local.yml logs -f postgres
```

### 3. Actuator Endpoints

```bash
# View all actuator endpoints
http GET localhost:8082/actuator

# View metrics
http GET localhost:8082/actuator/metrics

# View specific metric
http GET localhost:8082/actuator/metrics/http.server.requests

# View environment
http GET localhost:8082/actuator/env

# View thread dump
http GET localhost:8082/actuator/threaddump
```

### 4. Kafka Monitoring

Access Kafka UI at http://localhost:8090

```bash
# List topics
docker exec fv-kafka-local kafka-topics --list --bootstrap-server localhost:9092

# View messages in a topic
docker exec fv-kafka-local kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic product-events \
  --from-beginning

# Check consumer groups
docker exec fv-kafka-local kafka-consumer-groups \
  --bootstrap-server localhost:9092 \
  --list
```

## Troubleshooting

### Common Issues

1. **Port Already in Use**
```bash
# Find process using port
lsof -i :8082  # macOS/Linux
netstat -ano | findstr :8082  # Windows

# Kill process
kill -9 <PID>  # macOS/Linux
taskkill /PID <PID> /F  # Windows
```

2. **Database Connection Issues**
```bash
# Check PostgreSQL is running
docker ps | grep postgres

# Test connection
docker exec -it fv-postgres-local psql -U fvadmin -d firstviscount -c "\l"

# Check logs
docker logs fv-postgres-local
```

3. **Kafka Connection Issues**
```bash
# Check Kafka is running
docker ps | grep kafka

# Test Kafka connectivity
docker exec fv-kafka-local kafka-broker-api-versions --bootstrap-server localhost:9092
```

4. **Memory Issues**
```bash
# Increase JVM memory
export MAVEN_OPTS="-Xmx1024m -XX:MaxPermSize=256m"

# Or in application
java -Xmx1g -Xms512m -jar service.jar
```

### Reset Environment

```bash
#!/bin/bash
# scripts/reset-local-env.sh

echo "Stopping all services..."
./scripts/stop-all-services.sh

echo "Stopping Docker containers..."
docker-compose -f docker-compose.local.yml down -v

echo "Cleaning build artifacts..."
mvn clean

echo "Removing logs and PIDs..."
rm -rf logs/ pids/

echo "Starting fresh environment..."
docker-compose -f docker-compose.local.yml up -d

echo "Environment reset complete!"
```

## IDE Tips and Tricks

### IntelliJ IDEA

1. **Enable Spring Boot Dashboard**
   - View → Tool Windows → Services
   - Add Spring Boot services

2. **Live Templates**
   - Settings → Editor → Live Templates
   - Add custom templates for common code patterns

3. **Database Tools**
   - View → Tool Windows → Database
   - Add PostgreSQL data source

4. **HTTP Client**
   - Tools → HTTP Client → Create Request in HTTP Client
   - Save requests in `http/` directory

### VS Code

1. **Spring Boot Dashboard**
   - Install Spring Boot Dashboard extension
   - View all services in sidebar

2. **REST Client**
   - Install REST Client extension
   - Create `.http` files for API testing

3. **Java Debugging**
   - Set breakpoints by clicking line numbers
   - Use Debug view for step-through debugging

## Best Practices

1. **Use Profiles**: Always use `local` profile for development
2. **Version Control**: Never commit local configuration files
3. **Database Migrations**: Always create migrations for schema changes
4. **API Documentation**: Update OpenAPI specs when changing endpoints
5. **Testing**: Write tests for new features before committing
6. **Code Quality**: Run linters and code analysis before pushing
7. **Environment Isolation**: Keep local environment separate from others
8. **Regular Cleanup**: Periodically clean Docker volumes and build artifacts

---
*Last Updated: January 2025*