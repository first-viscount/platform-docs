# k3s Development Setup for Ubuntu Workstations

This guide covers setting up k3s on Ubuntu workstations for local microservices development of the First Viscount platform.

## System Requirements

### Minimum Requirements
- **OS**: Ubuntu 22.04 LTS or 24.04 LTS
- **CPU**: 4 cores (Intel i5/AMD Ryzen 5)
- **RAM**: 16GB DDR4
- **Storage**: 50GB free SSD space
- **Architecture**: x86_64

### Recommended Requirements
- **CPU**: 8 cores (Intel i7/AMD Ryzen 7)
- **RAM**: 32GB DDR4
- **Storage**: 100GB+ free NVMe SSD
- **Network**: Stable internet for pulling images

### Resource Allocation Breakdown
```
Component               | RAM Usage
------------------------|----------
k3s control plane       | ~1GB
Kafka                   | ~2GB
PostgreSQL              | ~1GB
MongoDB                 | ~1GB
Redis                   | ~0.5GB
6 Microservices         | ~3-6GB (0.5-1GB each)
Kubernetes overhead     | ~2GB
Operating system        | ~2GB
Development tools       | ~2-4GB
------------------------|----------
Total                   | ~16-20GB
```

## Pre-Installation Setup

### 1. System Preparation

```bash
# Update system packages
sudo apt update && sudo apt upgrade -y

# Install required packages
sudo apt install -y curl wget apt-transport-https ca-certificates software-properties-common

# Disable swap (required for Kubernetes)
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Enable required kernel modules
sudo modprobe overlay
sudo modprobe br_netfilter

# Make kernel modules persistent
cat <<EOF | sudo tee /etc/modules-load.d/k3s.conf
overlay
br_netfilter
EOF

# Configure sysctl for Kubernetes
cat <<EOF | sudo tee /etc/sysctl.d/k3s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

# Apply sysctl parameters
sudo sysctl --system
```

### 2. Firewall Configuration

```bash
# Option 1: Configure UFW (if enabled)
sudo ufw allow 6443/tcp  # Kubernetes API server
sudo ufw allow 10250/tcp # Kubelet metrics
sudo ufw allow 10251/tcp # kube-scheduler
sudo ufw allow 10252/tcp # kube-controller-manager
sudo ufw allow 2379:2380/tcp # etcd

# Option 2: Disable UFW for development (simpler)
sudo ufw disable
```

### 3. DNS Configuration (if needed)

```bash
# If experiencing DNS issues with systemd-resolved
sudo mkdir -p /etc/systemd/resolved.conf.d/
cat <<EOF | sudo tee /etc/systemd/resolved.conf.d/k3s.conf
[Resolve]
DNSStubListener=no
EOF

sudo systemctl restart systemd-resolved
```

## k3s Installation

### 1. Install k3s

```bash
# Install k3s with optimizations for development
curl -sfL https://get.k3s.io | sh -s - \
  --write-kubeconfig-mode 644 \
  --disable traefik \
  --disable metrics-server \
  --disable-cloud-controller \
  --kubelet-arg="eviction-hard=memory.available<1Gi" \
  --kubelet-arg="eviction-soft=memory.available<2Gi" \
  --kubelet-arg="eviction-soft-grace-period=memory.available=2m"

# Wait for k3s to be ready
sudo k3s kubectl wait --for=condition=ready nodes --all --timeout=60s
```

### 2. Configure kubectl

```bash
# Make kubectl accessible for your user
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config

# Install kubectl (if not already installed)
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Verify installation
kubectl version --client
kubectl get nodes
```

### 3. Install Helm (for easier deployments)

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

## Development Optimizations

### 1. Resource Quotas for Development

```yaml
# Create namespace with resource limits
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: firstviscount-dev
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: firstviscount-dev
spec:
  hard:
    requests.memory: "8Gi"
    requests.cpu: "4"
    limits.memory: "16Gi"
    limits.cpu: "8"
    persistentvolumeclaims: "10"
    services: "20"
    pods: "50"
EOF
```

### 2. Local Registry Setup

```bash
# Create local registry for faster image loading
kubectl create namespace registry
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: registry-pvc
  namespace: registry
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: registry
  namespace: registry
spec:
  replicas: 1
  selector:
    matchLabels:
      app: registry
  template:
    metadata:
      labels:
        app: registry
    spec:
      containers:
      - name: registry
        image: registry:2
        ports:
        - containerPort: 5000
        volumeMounts:
        - name: registry-storage
          mountPath: /var/lib/registry
        env:
        - name: REGISTRY_STORAGE_DELETE_ENABLED
          value: "true"
      volumes:
      - name: registry-storage
        persistentVolumeClaim:
          claimName: registry-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: registry
  namespace: registry
spec:
  type: NodePort
  ports:
  - port: 5000
    nodePort: 30500
  selector:
    app: registry
EOF

# Configure k3s to use local registry
cat <<EOF | sudo tee /etc/rancher/k3s/registries.yaml
mirrors:
  "localhost:30500":
    endpoint:
      - "http://localhost:30500"
EOF

sudo systemctl restart k3s
```

### 3. Development Storage Class

```yaml
# Create fast local storage class
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: rancher.io/local-path
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
EOF
```

## Service Deployment

### 1. Infrastructure Services

```bash
# Create infrastructure namespace
kubectl create namespace infrastructure

# Deploy PostgreSQL
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-dev-config
  namespace: infrastructure
data:
  postgresql.conf: |
    shared_buffers = 256MB
    effective_cache_size = 1GB
    work_mem = 4MB
    maintenance_work_mem = 64MB
    max_connections = 200
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: infrastructure
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
        env:
        - name: POSTGRES_PASSWORD
          value: devpassword
        - name: POSTGRES_USER
          value: postgres
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: infrastructure
spec:
  ports:
  - port: 5432
  selector:
    app: postgres
EOF
```

### 2. Deploy First Viscount Services

```bash
# Create deployment script
cat <<'EOF' > ~/deploy-firstviscount-k3s.sh
#!/bin/bash
set -e

echo "Deploying First Viscount services to k3s..."

# Deploy services in order
kubectl apply -f k8s/namespaces.yaml
kubectl apply -f k8s/infrastructure/
kubectl wait --for=condition=ready pod -l app=postgres -n infrastructure --timeout=300s
kubectl wait --for=condition=ready pod -l app=kafka -n infrastructure --timeout=300s

kubectl apply -f k8s/services/platform-coordination/
kubectl apply -f k8s/services/product-catalog/
kubectl apply -f k8s/services/inventory/
kubectl apply -f k8s/services/order/
kubectl apply -f k8s/services/delivery/
kubectl apply -f k8s/services/notification/

echo "Waiting for all pods to be ready..."
kubectl wait --for=condition=ready pod --all -n firstviscount-dev --timeout=600s

echo "Deployment complete!"
kubectl get pods -n firstviscount-dev
EOF

chmod +x ~/deploy-firstviscount-k3s.sh
```

## Resource Management

### 1. Memory Pressure Handling

```bash
# Monitor memory usage
kubectl top nodes
kubectl top pods -A

# Create memory-conscious pod specs
cat <<EOF > memory-optimized-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-catalog
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: product-catalog
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        env:
        - name: JAVA_OPTS
          value: "-Xmx384m -Xms256m -XX:MaxMetaspaceSize=128m"
EOF
```

### 2. Service Management Scripts

```bash
# Start only core services
cat <<'EOF' > ~/k3s-start-core.sh
#!/bin/bash
kubectl scale deployment platform-coordination product-catalog order-service --replicas=1 -n firstviscount-dev
kubectl scale deployment inventory-service delivery-service notification-service --replicas=0 -n firstviscount-dev
EOF

# Stop all services to save resources
cat <<'EOF' > ~/k3s-stop-services.sh
#!/bin/bash
kubectl scale deployment --all --replicas=0 -n firstviscount-dev
EOF

chmod +x ~/k3s-*.sh
```

## Troubleshooting

### Common Issues and Solutions

#### 1. High Memory Usage
```bash
# Check what's using memory
sudo k3s crictl stats

# Clean up unused images
sudo k3s crictl rmi --prune

# Restart k3s if needed
sudo systemctl restart k3s
```

#### 2. DNS Resolution Issues
```bash
# Check CoreDNS
kubectl get pods -n kube-system | grep coredns
kubectl logs -n kube-system -l k8s-app=kube-dns

# Restart CoreDNS if needed
kubectl rollout restart deployment coredns -n kube-system
```

#### 3. Storage Issues
```bash
# Check PVC status
kubectl get pvc -A

# Clean up unused PVCs
kubectl delete pvc --all -n firstviscount-dev
```

## Development Workflow

### 1. Building and Pushing Images

```bash
# Build and push to local registry
docker build -t localhost:30500/product-catalog:dev .
docker push localhost:30500/product-catalog:dev

# Update deployment
kubectl set image deployment/product-catalog product-catalog=localhost:30500/product-catalog:dev -n firstviscount-dev
```

### 2. Hot Reload with Skaffold

```bash
# Install Skaffold
curl -Lo skaffold https://storage.googleapis.com/skaffold/releases/latest/skaffold-linux-amd64
sudo install skaffold /usr/local/bin/

# Run with hot reload
skaffold dev --port-forward
```

### 3. Viewing Logs

```bash
# Stream logs from all services
kubectl logs -f -l app=product-catalog -n firstviscount-dev

# Use stern for better log viewing
wget https://github.com/stern/stern/releases/download/v1.22.0/stern_1.22.0_linux_amd64.tar.gz
tar -xvf stern_1.22.0_linux_amd64.tar.gz
sudo mv stern /usr/local/bin/

stern ".*" -n firstviscount-dev
```

## Performance Tips

1. **Use SSD Storage**: k3s performs much better on SSD
2. **Limit Replicas**: Use single replicas for development
3. **Resource Limits**: Set appropriate limits to prevent OOM
4. **Selective Deployment**: Only run services you're working on
5. **Regular Cleanup**: Remove unused images and volumes

## Next Steps

- Review [Hybrid Development Strategy](./hybrid-development-strategy.md) for when to use k3s vs Docker Compose
- Set up monitoring with [Monitoring and Observability](./monitoring-observability.md)
- Configure CI/CD with [CI/CD Pipeline](./ci-cd-pipeline.md)

---
*Last Updated: January 2025*