# Kubernetes Deployment Guide

This document provides comprehensive instructions for deploying the First Viscount microservices platform to Kubernetes, covering both local (k3s) and production environments.

## Prerequisites

### System Requirements
- Kubernetes 1.28+ or k3s 1.28+
- kubectl configured
- Helm 3.12+
- Docker registry access
- Minimum 16GB RAM for local deployment
- 50GB available disk space

### Required Tools
```bash
# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Install Helm
curl https://get.helm.sh/helm-v3.12.0-linux-amd64.tar.gz | tar xz
sudo mv linux-amd64/helm /usr/local/bin/

# Install k3s (for local development)
curl -sfL https://get.k3s.io | sh -
```

## Namespace Setup

### Create Namespaces
```yaml
# namespaces.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: firstviscount-prod
  labels:
    name: firstviscount-prod
    environment: production
---
apiVersion: v1
kind: Namespace
metadata:
  name: firstviscount-staging
  labels:
    name: firstviscount-staging
    environment: staging
---
apiVersion: v1
kind: Namespace
metadata:
  name: firstviscount-dev
  labels:
    name: firstviscount-dev
    environment: development
```

```bash
kubectl apply -f namespaces.yaml
```

## Infrastructure Components

### 1. PostgreSQL Deployment

```yaml
# postgres/postgres-deployment.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: firstviscount-prod
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: fast-ssd
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
  namespace: firstviscount-prod
data:
  POSTGRES_DB: firstviscount
  POSTGRES_USER: fvadmin
  postgresql.conf: |
    max_connections = 200
    shared_buffers = 256MB
    effective_cache_size = 1GB
    maintenance_work_mem = 64MB
    checkpoint_completion_target = 0.9
    wal_buffers = 16MB
    default_statistics_target = 100
    random_page_cost = 1.1
    effective_io_concurrency = 200
    work_mem = 4MB
    min_wal_size = 1GB
    max_wal_size = 4GB
---
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
  namespace: firstviscount-prod
type: Opaque
stringData:
  POSTGRES_PASSWORD: "your-secure-password-here"
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: firstviscount-prod
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine
        ports:
        - containerPort: 5432
          name: postgres
        envFrom:
        - configMapRef:
            name: postgres-config
        - secretRef:
            name: postgres-secret
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        - name: postgres-config-volume
          mountPath: /etc/postgresql/postgresql.conf
          subPath: postgresql.conf
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          exec:
            command:
              - pg_isready
              - -U
              - fvadmin
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
              - pg_isready
              - -U
              - fvadmin
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc
      - name: postgres-config-volume
        configMap:
          name: postgres-config
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: firstviscount-prod
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
  clusterIP: None
```

### 2. Kafka Deployment

```yaml
# kafka/kafka-deployment.yaml
apiVersion: v1
kind: Service
metadata:
  name: kafka-headless
  namespace: firstviscount-prod
spec:
  clusterIP: None
  selector:
    app: kafka
  ports:
  - name: kafka
    port: 9092
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kafka
  namespace: firstviscount-prod
spec:
  serviceName: kafka-headless
  replicas: 3
  selector:
    matchLabels:
      app: kafka
  template:
    metadata:
      labels:
        app: kafka
    spec:
      containers:
      - name: kafka
        image: confluentinc/cp-kafka:7.4.0
        ports:
        - containerPort: 9092
          name: kafka
        - containerPort: 9093
          name: controller
        env:
        - name: KAFKA_NODE_ID
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: KAFKA_PROCESS_ROLES
          value: "broker,controller"
        - name: KAFKA_LISTENERS
          value: "PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093"
        - name: KAFKA_ADVERTISED_LISTENERS
          value: "PLAINTEXT://$(KAFKA_NODE_ID).kafka-headless.firstviscount-prod.svc.cluster.local:9092"
        - name: KAFKA_CONTROLLER_LISTENER_NAMES
          value: "CONTROLLER"
        - name: KAFKA_LISTENER_SECURITY_PROTOCOL_MAP
          value: "PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT"
        - name: KAFKA_CONTROLLER_QUORUM_VOTERS
          value: "0@kafka-0.kafka-headless.firstviscount-prod.svc.cluster.local:9093,1@kafka-1.kafka-headless.firstviscount-prod.svc.cluster.local:9093,2@kafka-2.kafka-headless.firstviscount-prod.svc.cluster.local:9093"
        - name: KAFKA_LOG_DIRS
          value: "/var/lib/kafka/data"
        - name: KAFKA_NUM_PARTITIONS
          value: "3"
        - name: KAFKA_DEFAULT_REPLICATION_FACTOR
          value: "3"
        - name: KAFKA_MIN_INSYNC_REPLICAS
          value: "2"
        - name: KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR
          value: "3"
        volumeMounts:
        - name: kafka-data
          mountPath: /var/lib/kafka/data
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
  volumeClaimTemplates:
  - metadata:
      name: kafka-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

### 3. Redis Deployment

```yaml
# redis/redis-deployment.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-config
  namespace: firstviscount-prod
data:
  redis.conf: |
    maxmemory 2gb
    maxmemory-policy allkeys-lru
    save 900 1
    save 300 10
    save 60 10000
    appendonly yes
    appendfsync everysec
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: firstviscount-prod
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        command:
          - redis-server
          - /etc/redis/redis.conf
        ports:
        - containerPort: 6379
        volumeMounts:
        - name: redis-config
          mountPath: /etc/redis
        - name: redis-data
          mountPath: /data
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
      volumes:
      - name: redis-config
        configMap:
          name: redis-config
      - name: redis-data
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: firstviscount-prod
spec:
  selector:
    app: redis
  ports:
  - port: 6379
    targetPort: 6379
```

## Microservices Deployment

### 1. Common ConfigMaps and Secrets

```yaml
# config/common-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: common-config
  namespace: firstviscount-prod
data:
  SPRING_PROFILES_ACTIVE: "production"
  KAFKA_BOOTSTRAP_SERVERS: "kafka-0.kafka-headless:9092,kafka-1.kafka-headless:9092,kafka-2.kafka-headless:9092"
  REDIS_HOST: "redis"
  REDIS_PORT: "6379"
  LOG_LEVEL: "INFO"
  MANAGEMENT_ENDPOINTS_WEB_EXPOSURE_INCLUDE: "health,info,metrics,prometheus"
---
apiVersion: v1
kind: Secret
metadata:
  name: common-secret
  namespace: firstviscount-prod
type: Opaque
stringData:
  JWT_SECRET: "your-jwt-secret-here"
  DB_PASSWORD: "your-db-password-here"
```

### 2. Product Service Deployment

```yaml
# services/product-service.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: product-service-config
  namespace: firstviscount-prod
data:
  APPLICATION_NAME: "product-service"
  SERVER_PORT: "8080"
  DB_HOST: "postgres"
  DB_PORT: "5432"
  DB_NAME: "product_db"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service
  namespace: firstviscount-prod
  labels:
    app: product-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: product-service
  template:
    metadata:
      labels:
        app: product-service
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/actuator/prometheus"
    spec:
      containers:
      - name: product-service
        image: firstviscount/product-service:latest
        ports:
        - containerPort: 8080
          name: http
        envFrom:
        - configMapRef:
            name: common-config
        - configMapRef:
            name: product-service-config
        - secretRef:
            name: common-secret
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 5
        volumeMounts:
        - name: app-logs
          mountPath: /var/log/app
      volumes:
      - name: app-logs
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: product-service
  namespace: firstviscount-prod
spec:
  selector:
    app: product-service
  ports:
  - port: 8080
    targetPort: 8080
    name: http
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: product-service-hpa
  namespace: firstviscount-prod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: product-service
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### 3. Order Service Deployment

```yaml
# services/order-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: firstviscount-prod
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
      - name: order-service
        image: firstviscount/order-service:latest
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_DATASOURCE_URL
          value: "jdbc:postgresql://postgres:5432/order_db"
        - name: SPRING_DATASOURCE_USERNAME
          value: "fvadmin"
        - name: SPRING_DATASOURCE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: common-secret
              key: DB_PASSWORD
        envFrom:
        - configMapRef:
            name: common-config
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: firstviscount-prod
spec:
  selector:
    app: order-service
  ports:
  - port: 8080
    targetPort: 8080
```

## Ingress Configuration

### 1. Nginx Ingress Controller

```bash
# Install Nginx Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml
```

### 2. Ingress Rules

```yaml
# ingress/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: firstviscount-ingress
  namespace: firstviscount-prod
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "300"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.firstviscount.com
    secretName: firstviscount-tls
  rules:
  - host: api.firstviscount.com
    http:
      paths:
      - path: /api/v1/products
        pathType: Prefix
        backend:
          service:
            name: product-service
            port:
              number: 8080
      - path: /api/v1/orders
        pathType: Prefix
        backend:
          service:
            name: order-service
            port:
              number: 8080
      - path: /api/v1/inventory
        pathType: Prefix
        backend:
          service:
            name: inventory-service
            port:
              number: 8080
      - path: /api/v1/delivery
        pathType: Prefix
        backend:
          service:
            name: delivery-service
            port:
              number: 8080
```

## Service Mesh (Istio)

### 1. Install Istio

```bash
# Download and install Istio
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH=$PWD/bin:$PATH

# Install Istio with production profile
istioctl install --set profile=production -y

# Enable automatic sidecar injection
kubectl label namespace firstviscount-prod istio-injection=enabled
```

### 2. Istio Configuration

```yaml
# istio/virtual-service.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: product-service
  namespace: firstviscount-prod
spec:
  hosts:
  - product-service
  http:
  - match:
    - headers:
        version:
          exact: v2
    route:
    - destination:
        host: product-service
        subset: v2
  - route:
    - destination:
        host: product-service
        subset: v1
      weight: 90
    - destination:
        host: product-service
        subset: v2
      weight: 10
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: product-service
  namespace: firstviscount-prod
spec:
  host: product-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 100
        http2MaxRequests: 100
    loadBalancer:
      simple: LEAST_REQUEST
    outlierDetection:
      consecutiveErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

## Monitoring and Observability

### 1. Prometheus Stack

```bash
# Install kube-prometheus-stack
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=20Gi \
  --set alertmanager.alertmanagerSpec.storage.volumeClaimTemplate.spec.resources.requests.storage=10Gi
```

### 2. Service Monitors

```yaml
# monitoring/service-monitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: firstviscount-services
  namespace: firstviscount-prod
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: firstviscount
  endpoints:
  - port: http
    path: /actuator/prometheus
    interval: 30s
```

### 3. Grafana Dashboards

```yaml
# monitoring/grafana-dashboard.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: firstviscount-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  firstviscount-dashboard.json: |
    {
      "dashboard": {
        "title": "First Viscount Microservices",
        "panels": [
          {
            "title": "Request Rate",
            "targets": [
              {
                "expr": "rate(http_server_requests_seconds_count[5m])"
              }
            ]
          },
          {
            "title": "Error Rate",
            "targets": [
              {
                "expr": "rate(http_server_requests_seconds_count{status=~\"5..\"}[5m])"
              }
            ]
          }
        ]
      }
    }
```

## Security Configuration

### 1. Network Policies

```yaml
# security/network-policies.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-to-services
  namespace: firstviscount-prod
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    - podSelector:
        matchLabels:
          app: firstviscount-gateway
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector: {}
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: TCP
      port: 53
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
```

### 2. Pod Security Policies

```yaml
# security/pod-security-policy.yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: firstviscount-psp
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
    - ALL
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    - 'persistentVolumeClaim'
  runAsUser:
    rule: 'MustRunAsNonRoot'
  seLinux:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
  readOnlyRootFilesystem: true
```

## Deployment Scripts

### 1. Full Deployment Script

```bash
#!/bin/bash
# deploy.sh

set -e

NAMESPACE="firstviscount-prod"
ENVIRONMENT="production"

echo "Deploying First Viscount to $ENVIRONMENT..."

# Create namespace
kubectl apply -f namespaces.yaml

# Deploy infrastructure
echo "Deploying infrastructure components..."
kubectl apply -f postgres/
kubectl apply -f kafka/
kubectl apply -f redis/

# Wait for infrastructure
echo "Waiting for infrastructure to be ready..."
kubectl wait --for=condition=ready pod -l app=postgres -n $NAMESPACE --timeout=300s
kubectl wait --for=condition=ready pod -l app=kafka -n $NAMESPACE --timeout=300s
kubectl wait --for=condition=ready pod -l app=redis -n $NAMESPACE --timeout=300s

# Deploy configurations
echo "Deploying configurations..."
kubectl apply -f config/

# Deploy services
echo "Deploying microservices..."
kubectl apply -f services/

# Deploy ingress
echo "Deploying ingress rules..."
kubectl apply -f ingress/

# Deploy monitoring
echo "Deploying monitoring..."
kubectl apply -f monitoring/

echo "Deployment completed!"
kubectl get pods -n $NAMESPACE
```

### 2. Rolling Update Script

```bash
#!/bin/bash
# rolling-update.sh

SERVICE=$1
VERSION=$2

if [ -z "$SERVICE" ] || [ -z "$VERSION" ]; then
    echo "Usage: ./rolling-update.sh <service-name> <version>"
    exit 1
fi

echo "Performing rolling update for $SERVICE to version $VERSION..."

# Update image
kubectl set image deployment/$SERVICE $SERVICE=firstviscount/$SERVICE:$VERSION \
  -n firstviscount-prod

# Check rollout status
kubectl rollout status deployment/$SERVICE -n firstviscount-prod

echo "Rolling update completed!"
```

## Backup and Disaster Recovery

### 1. Database Backup CronJob

```yaml
# backup/postgres-backup.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: firstviscount-prod
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: postgres-backup
            image: postgres:15-alpine
            env:
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_PASSWORD
            command:
            - sh
            - -c
            - |
              DATE=$(date +%Y%m%d_%H%M%S)
              pg_dump -h postgres -U fvadmin -d firstviscount > /backup/firstviscount_$DATE.sql
              # Upload to S3 or other storage
              aws s3 cp /backup/firstviscount_$DATE.sql s3://firstviscount-backups/
            volumeMounts:
            - name: backup
              mountPath: /backup
          volumes:
          - name: backup
            emptyDir: {}
          restartPolicy: OnFailure
```

## Production Checklist

- [ ] All secrets are properly configured and not hardcoded
- [ ] Resource limits and requests are set for all containers
- [ ] Health checks (liveness and readiness probes) are configured
- [ ] HPA is configured for auto-scaling
- [ ] Network policies are in place
- [ ] TLS/SSL certificates are configured
- [ ] Monitoring and alerting is set up
- [ ] Backup strategy is implemented
- [ ] Disaster recovery plan is documented
- [ ] Security scanning is integrated in CI/CD
- [ ] Load testing has been performed
- [ ] Documentation is up to date

---
*Last Updated: January 2025*