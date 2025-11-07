# Node Operations

## Overview

Master the deployment, configuration, and maintenance of BSV blockchain nodes, from legacy monolithic nodes to modern Teranode microservices infrastructure. This module covers everything from basic node setup to production-grade Teranode deployments with Kubernetes, monitoring, security hardening, and operational best practices.

**Estimated Time:** 5-6 hours
**Difficulty:** Advanced
**Prerequisites:** Complete [Network Topology](../network-topology/README.md) and have basic Linux/DevOps knowledge

## Learning Objectives

By the end of this module, you will be able to:

- ✅ Set up and configure legacy BSV nodes
- ✅ Deploy Teranode microservices infrastructure
- ✅ Configure node security and firewalls
- ✅ Implement monitoring and alerting systems
- ✅ Perform node maintenance and upgrades
- ✅ Optimize node performance
- ✅ Deploy Teranode with Docker and Kubernetes
- ✅ Implement backup and disaster recovery strategies
- ✅ Troubleshoot common node issues
- ✅ Scale infrastructure horizontally with Teranode

## SDK Components Used

This course leverages these standardized SDK modules:

- **[ARC](../../../sdk-components/arc/README.md)** - Node communication interface
- **[SPV](../../../sdk-components/spv/README.md)** - Header sync and validation
- **[Transaction](../../../sdk-components/transaction/README.md)** - Node transaction handling
- **[BEEF](../../../sdk-components/beef/README.md)** - Transaction envelope processing

## 1. Node Types and Requirements

### System Requirements Comparison

**Legacy Node (SV Node)**:
```
Minimum:
- CPU: 4 cores
- RAM: 8 GB
- Storage: 1 TB SSD (growing)
- Network: 100 Mbps

Recommended:
- CPU: 8+ cores
- RAM: 16-32 GB
- Storage: 2-4 TB NVMe SSD
- Network: 1 Gbps
```

**Teranode (Microservices)**:
```
Per Service Instance:
- CPU: 2-4 cores
- RAM: 4-8 GB
- Storage: Varies by service
- Network: 1-10 Gbps

Infrastructure:
- Kubernetes cluster (3+ nodes)
- Kafka cluster (3+ brokers)
- PostgreSQL/distributed database
- Object storage (S3-compatible)
- Load balancer
```

### Node Types by Use Case

**1. Mining Node**
- Purpose: Block production
- Requirements: Highest performance
- Best choice: Teranode (scalable block assembly)

**2. Archival Node**
- Purpose: Full blockchain history
- Requirements: Large storage
- Best choice: Legacy or Teranode with full persistence

**3. Transaction Processor**
- Purpose: High-volume tx validation
- Requirements: High throughput
- Best choice: Teranode (1M+ tx/sec)

**4. Development Node**
- Purpose: Testing and development
- Requirements: Moderate resources
- Best choice: Legacy (simpler setup) or Teranode (production parity)

## 2. Legacy Node Setup

### Installing SV Node (Ubuntu/Debian)

```bash
# Update system
sudo apt-get update
sudo apt-get upgrade -y

# Install dependencies
sudo apt-get install -y \
  build-essential \
  libtool \
  autotools-dev \
  automake \
  pkg-config \
  libssl-dev \
  libevent-dev \
  bsdmainutils \
  libboost-all-dev \
  libzmq3-dev

# Download and build Bitcoin SV Node
# (Check https://github.com/bitcoin-sv/bitcoin-sv for latest release)
cd /opt
sudo git clone https://github.com/bitcoin-sv/bitcoin-sv.git
cd bitcoin-sv

# Build from source
sudo ./autogen.sh
sudo ./configure --without-gui
sudo make -j$(nproc)
sudo make install

# Verify installation
bitcoind --version
```

### Basic Configuration

Create configuration file at `~/.bitcoin/bitcoin.conf`:

```conf
# Network
testnet=0  # 0 for mainnet, 1 for testnet
port=8333

# RPC Server
server=1
rpcuser=your_rpc_username
rpcpassword=your_strong_rpc_password
rpcport=8332
rpcallowip=127.0.0.1

# Indexing
txindex=1  # Full transaction index

# Memory Pool
maxmempool=4000  # MB

# Connections
maxconnections=125

# Logging
debug=0
shrinkdebugfile=1

# Data Directory
datadir=/data/bitcoin

# Performance
dbcache=2000  # MB for database cache
par=4  # Number of script verification threads
```

### Starting the Node

```bash
# Create systemd service
sudo tee /etc/systemd/system/bitcoind.service > /dev/null <<EOF
[Unit]
Description=Bitcoin SV Daemon
After=network.target

[Service]
Type=forking
User=bitcoin
Group=bitcoin
WorkingDirectory=/home/bitcoin
ExecStart=/usr/local/bin/bitcoind -daemon -conf=/home/bitcoin/.bitcoin/bitcoin.conf
ExecStop=/usr/local/bin/bitcoin-cli stop
Restart=on-failure
RestartSec=30

[Install]
WantedBy=multi-user.target
EOF

# Enable and start service
sudo systemctl daemon-reload
sudo systemctl enable bitcoind
sudo systemctl start bitcoind

# Check status
sudo systemctl status bitcoind

# Monitor logs
tail -f ~/.bitcoin/debug.log
```

### Initial Blockchain Sync

```bash
# Check sync progress
bitcoin-cli getblockchaininfo

# Output:
# {
#   "chain": "main",
#   "blocks": 750000,
#   "headers": 800000,
#   "verificationprogress": 0.9375,
#   "chainwork": "...",
#   ...
# }

# Monitor sync
watch -n 10 'bitcoin-cli getblockchaininfo | grep -E "blocks|headers|verificationprogress"'
```

## 3. Teranode Setup

### Prerequisites

**Infrastructure Requirements**:
- Kubernetes cluster (v1.24+)
- Kafka cluster (v3.0+)
- PostgreSQL (v14+) or distributed database
- S3-compatible object storage
- Monitoring stack (Prometheus, Grafana)

### Docker-Based Teranode (Development)

```bash
# Clone Teranode repository
git clone https://github.com/bitcoin-sv/teranode.git
cd teranode

# Configure environment
cp .env.example .env
nano .env

# Example .env configuration:
# KAFKA_BROKERS=kafka:9092
# POSTGRES_HOST=postgres
# POSTGRES_DB=teranode
# POSTGRES_USER=teranode
# POSTGRES_PASSWORD=secure_password
# UTXO_STORE_TYPE=postgres
# BLOB_STORE_TYPE=s3
# S3_ENDPOINT=minio:9000
# S3_BUCKET=teranode-blocks
```

**Docker Compose Setup**:

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Kafka for event messaging
  kafka:
    image: confluentinc/cp-kafka:7.4.0
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    ports:
      - "9092:9092"
    depends_on:
      - zookeeper

  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  # PostgreSQL for UTXO storage
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: teranode
      POSTGRES_USER: teranode
      POSTGRES_PASSWORD: secure_password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  # MinIO for blob storage
  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    volumes:
      - minio_data:/data
    ports:
      - "9000:9000"
      - "9001:9001"

  # Teranode Transaction Validator
  tx-validator:
    build: ./services/tx-validator
    environment:
      KAFKA_BROKERS: kafka:9092
      POSTGRES_HOST: postgres
    depends_on:
      - kafka
      - postgres
    deploy:
      replicas: 3

  # Teranode Block Assembler
  block-assembler:
    build: ./services/block-assembler
    environment:
      KAFKA_BROKERS: kafka:9092
      POSTGRES_HOST: postgres
    depends_on:
      - kafka
      - postgres
    deploy:
      replicas: 2

  # Teranode P2P Service
  p2p-service:
    build: ./services/p2p
    ports:
      - "8333:8333"
    environment:
      KAFKA_BROKERS: kafka:9092
    depends_on:
      - kafka

volumes:
  postgres_data:
  minio_data:
```

**Start Teranode**:

```bash
# Start all services
docker-compose up -d

# Check service health
docker-compose ps

# View logs
docker-compose logs -f tx-validator

# Scale transaction validators
docker-compose up -d --scale tx-validator=10
```

### Kubernetes Deployment (Production)

**Namespace and Configuration**:

```yaml
# teranode-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: teranode

---
# teranode-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: teranode-config
  namespace: teranode
data:
  kafka.brokers: "kafka-0.kafka:9092,kafka-1.kafka:9092,kafka-2.kafka:9092"
  postgres.host: "postgres-service"
  postgres.db: "teranode"
  utxo.store.type: "postgres"
  blob.store.type: "s3"
  s3.endpoint: "https://s3.amazonaws.com"
  s3.bucket: "teranode-production"
```

**Transaction Validator Deployment**:

```yaml
# tx-validator-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tx-validator
  namespace: teranode
spec:
  replicas: 10
  selector:
    matchLabels:
      app: tx-validator
  template:
    metadata:
      labels:
        app: tx-validator
    spec:
      containers:
      - name: validator
        image: teranode/tx-validator:latest
        resources:
          requests:
            memory: "4Gi"
            cpu: "2"
          limits:
            memory: "8Gi"
            cpu: "4"
        env:
        - name: KAFKA_BROKERS
          valueFrom:
            configMapKeyRef:
              name: teranode-config
              key: kafka.brokers
        - name: POSTGRES_HOST
          valueFrom:
            configMapKeyRef:
              name: teranode-config
              key: postgres.host
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5

---
apiVersion: v1
kind: Service
metadata:
  name: tx-validator
  namespace: teranode
spec:
  selector:
    app: tx-validator
  ports:
  - port: 8080
    targetPort: 8080
  type: ClusterIP
```

**Horizontal Pod Autoscaler**:

```yaml
# tx-validator-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: tx-validator-hpa
  namespace: teranode
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: tx-validator
  minReplicas: 5
  maxReplicas: 50
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

**Deploy to Kubernetes**:

```bash
# Apply configurations
kubectl apply -f teranode-namespace.yaml
kubectl apply -f teranode-config.yaml
kubectl apply -f postgres-secret.yaml
kubectl apply -f tx-validator-deployment.yaml
kubectl apply -f block-assembler-deployment.yaml
kubectl apply -f p2p-service-deployment.yaml
kubectl apply -f tx-validator-hpa.yaml

# Check deployments
kubectl get deployments -n teranode

# Check pods
kubectl get pods -n teranode

# View logs
kubectl logs -f -n teranode deployment/tx-validator

# Scale manually
kubectl scale deployment tx-validator -n teranode --replicas=20
```

## 4. Node Security

### Firewall Configuration (Legacy Node)

```bash
# UFW firewall setup
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH
sudo ufw allow 22/tcp

# Allow Bitcoin P2P
sudo ufw allow 8333/tcp

# Allow RPC (only from specific IPs)
sudo ufw allow from 10.0.0.0/24 to any port 8332

# Enable firewall
sudo ufw enable

# Check status
sudo ufw status verbose
```

### SSL/TLS for RPC

```bash
# Generate SSL certificate
openssl req -x509 -newkey rsa:4096 \
  -keyout ~/.bitcoin/server.key \
  -out ~/.bitcoin/server.crt \
  -days 365 -nodes

# Update bitcoin.conf
echo "rpcssl=1" >> ~/.bitcoin/bitcoin.conf
echo "rpcsslcertificatechainfile=$HOME/.bitcoin/server.crt" >> ~/.bitcoin/bitcoin.conf
echo "rpcsslprivatekeyfile=$HOME/.bitcoin/server.key" >> ~/.bitcoin/bitcoin.conf

# Restart node
sudo systemctl restart bitcoind
```

### Kubernetes Network Policies (Teranode)

```yaml
# network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: teranode-network-policy
  namespace: teranode
spec:
  podSelector:
    matchLabels:
      app: tx-validator
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: p2p-service
    - podSelector:
        matchLabels:
          app: api-gateway
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: kafka
    ports:
    - protocol: TCP
      port: 9092
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
```

### Secrets Management

```bash
# Create Kubernetes secrets
kubectl create secret generic postgres-secret \
  --from-literal=username=teranode \
  --from-literal=password=your_secure_password \
  -n teranode

kubectl create secret generic s3-credentials \
  --from-literal=access_key=your_access_key \
  --from-literal=secret_key=your_secret_key \
  -n teranode

# Verify secrets
kubectl get secrets -n teranode
```

## 5. Monitoring and Observability

### Legacy Node Monitoring

**Install Node Exporter**:

```bash
# Download and install
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
tar xvfz node_exporter-1.6.1.linux-amd64.tar.gz
sudo cp node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/

# Create systemd service
sudo tee /etc/systemd/system/node_exporter.service > /dev/null <<EOF
[Unit]
Description=Node Exporter

[Service]
User=node_exporter
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
```

**Bitcoin Node Monitoring Script**:

```bash
#!/bin/bash
# monitor-node.sh

while true; do
  echo "=== Bitcoin Node Status ==="
  echo "Blockchain Info:"
  bitcoin-cli getblockchaininfo | jq '.blocks, .headers, .verificationprogress'

  echo -e "\nNetwork Info:"
  bitcoin-cli getnetworkinfo | jq '.connections, .localservices'

  echo -e "\nMempool Info:"
  bitcoin-cli getmempoolinfo | jq '.size, .bytes, .usage'

  echo -e "\nUptime:"
  bitcoin-cli uptime

  sleep 60
done
```

### Teranode Monitoring with Prometheus

**Prometheus Configuration**:

```yaml
# prometheus-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: teranode
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s

    scrape_configs:
    - job_name: 'tx-validator'
      kubernetes_sd_configs:
      - role: pod
        namespaces:
          names:
          - teranode
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        action: keep
        regex: tx-validator
      - source_labels: [__meta_kubernetes_pod_ip]
        target_label: __address__
        replacement: ${1}:8080

    - job_name: 'block-assembler'
      kubernetes_sd_configs:
      - role: pod
        namespaces:
          names:
          - teranode
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        action: keep
        regex: block-assembler

    - job_name: 'kafka'
      static_configs:
      - targets: ['kafka-0:9092', 'kafka-1:9092', 'kafka-2:9092']
```

**Grafana Dashboard** (Conceptual queries):

```
# Transaction throughput
rate(teranode_transactions_processed_total[5m])

# Validation latency
histogram_quantile(0.95, rate(teranode_validation_duration_seconds_bucket[5m]))

# Active connections
teranode_p2p_connections_active

# Mempool size
teranode_mempool_size_bytes

# UTXO database size
teranode_utxo_db_size_bytes
```

### Alerting Rules

```yaml
# alerts.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-alerts
  namespace: teranode
data:
  alerts.yml: |
    groups:
    - name: teranode
      rules:
      - alert: HighTransactionLatency
        expr: histogram_quantile(0.95, rate(teranode_validation_duration_seconds_bucket[5m])) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High transaction validation latency"
          description: "95th percentile latency is {{ $value }}s"

      - alert: ServiceDown
        expr: up{job="tx-validator"} == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.instance }} is down"

      - alert: HighMemoryUsage
        expr: container_memory_usage_bytes / container_spec_memory_limit_bytes > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage on {{ $labels.pod }}"
```

## 6. Performance Optimization

### Legacy Node Optimization

**Database Cache Tuning**:

```conf
# bitcoin.conf optimizations
dbcache=8000  # Increase for faster IBD
maxmempool=4000  # Larger mempool
par=8  # More verification threads
rpcthreads=8  # More RPC threads

# Connection tuning
maxconnections=250
maxuploadtarget=5000  # MB/day upload limit
```

**System Optimization**:

```bash
# Increase file descriptors
sudo tee -a /etc/security/limits.conf > /dev/null <<EOF
bitcoin soft nofile 65536
bitcoin hard nofile 65536
EOF

# Optimize disk I/O
echo 'deadline' | sudo tee /sys/block/sda/queue/scheduler

# Disable swap for better performance
sudo swapoff -a
```

### Teranode Performance Tuning

**Resource Allocation**:

```yaml
# Optimized resource requests
resources:
  requests:
    memory: "8Gi"
    cpu: "4"
  limits:
    memory: "16Gi"
    cpu: "8"

# Anti-affinity for high availability
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - tx-validator
        topologyKey: kubernetes.io/hostname
```

**Kafka Tuning**:

```properties
# kafka.properties
# Increase partitions for parallelism
num.partitions=50

# Batch processing
batch.size=16384
linger.ms=10

# Compression
compression.type=snappy

# Replication
min.insync.replicas=2
default.replication.factor=3
```

## 7. Backup and Disaster Recovery

### Legacy Node Backup

```bash
#!/bin/bash
# backup-node.sh

BACKUP_DIR="/backup/bitcoin"
DATE=$(date +%Y%m%d)

# Stop node
bitcoin-cli stop
sleep 10

# Backup blockchain data
tar -czf "$BACKUP_DIR/blocks-$DATE.tar.gz" ~/.bitcoin/blocks
tar -czf "$BACKUP_DIR/chainstate-$DATE.tar.gz" ~/.bitcoin/chainstate

# Backup wallet (if applicable)
if [ -f ~/.bitcoin/wallet.dat ]; then
  cp ~/.bitcoin/wallet.dat "$BACKUP_DIR/wallet-$DATE.dat"
fi

# Restart node
bitcoind -daemon

# Clean old backups (keep last 7 days)
find "$BACKUP_DIR" -name "*.tar.gz" -mtime +7 -delete
```

### Teranode Backup Strategy

**PostgreSQL Backup**:

```bash
#!/bin/bash
# backup-postgres.sh

BACKUP_DIR="/backup/postgres"
DATE=$(date +%Y%m%d)

# Backup UTXO database
kubectl exec -n teranode postgres-0 -- \
  pg_dump -U teranode teranode | \
  gzip > "$BACKUP_DIR/utxo-$DATE.sql.gz"

# Backup to S3
aws s3 cp "$BACKUP_DIR/utxo-$DATE.sql.gz" \
  s3://teranode-backups/postgres/
```

**Kafka Topic Backup**:

```bash
# Backup Kafka topics
kubectl exec -n teranode kafka-0 -- \
  kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic transactions \
  --from-beginning \
  --max-messages 1000000 > transactions-backup.json
```

### Disaster Recovery Plan

**Recovery Steps**:

1. **Restore Infrastructure**:
```bash
# Restore Kubernetes cluster
kubectl apply -f teranode-infrastructure/

# Restore persistent volumes
kubectl apply -f pv-restore.yaml
```

2. **Restore Data**:
```bash
# Restore PostgreSQL
gunzip < utxo-backup.sql.gz | \
  kubectl exec -i -n teranode postgres-0 -- \
  psql -U teranode teranode

# Restore Kafka topics
# Kafka will rebuild from existing data
```

3. **Validate Recovery**:
```bash
# Check service health
kubectl get pods -n teranode

# Verify data integrity
kubectl exec -n teranode postgres-0 -- \
  psql -U teranode teranode -c "SELECT COUNT(*) FROM utxos;"
```

## 8. Upgrades and Maintenance

### Legacy Node Upgrade

```bash
# Backup before upgrade
./backup-node.sh

# Download new version
cd /opt/bitcoin-sv
git fetch
git checkout v1.1.0  # Replace with desired version

# Build
./autogen.sh
./configure --without-gui
make -j$(nproc)

# Stop old version
bitcoin-cli stop
sleep 10

# Install new version
sudo make install

# Start new version
bitcoind -daemon

# Verify
bitcoin-cli getnetworkinfo | grep subversion
```

### Teranode Rolling Update

```yaml
# rolling-update.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tx-validator
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0  # Zero downtime
  template:
    spec:
      containers:
      - name: validator
        image: teranode/tx-validator:v2.0.0  # New version
```

```bash
# Apply rolling update
kubectl apply -f rolling-update.yaml

# Watch rollout
kubectl rollout status deployment/tx-validator -n teranode

# Rollback if needed
kubectl rollout undo deployment/tx-validator -n teranode
```

## 9. Troubleshooting

### Legacy Node Issues

**Sync Stuck**:
```bash
# Check peers
bitcoin-cli getpeerinfo

# Add peers manually
bitcoin-cli addnode "seed.bitcoinsv.io" "onetry"

# Clear peers and restart
rm ~/.bitcoin/peers.dat
sudo systemctl restart bitcoind
```

**High Memory Usage**:
```bash
# Reduce cache
bitcoin-cli stop
# Edit bitcoin.conf: dbcache=2000
bitcoind -daemon
```

**RPC Connection Failed**:
```bash
# Check if node is running
ps aux | grep bitcoind

# Check RPC credentials
cat ~/.bitcoin/bitcoin.conf | grep rpc

# Test RPC
curl --user rpcuser:rpcpassword \
  --data-binary '{"jsonrpc":"1.0","id":"test","method":"getblockcount"}' \
  -H 'content-type: text/plain;' \
  http://127.0.0.1:8332/
```

### Teranode Issues

**Pod CrashLoopBackOff**:
```bash
# Check logs
kubectl logs -n teranode pod/tx-validator-xxx

# Describe pod for events
kubectl describe pod -n teranode tx-validator-xxx

# Check resource limits
kubectl top pod -n teranode
```

**Kafka Connection Issues**:
```bash
# Test Kafka connectivity
kubectl run -it --rm kafka-test \
  --image=confluentinc/cp-kafka:7.4.0 \
  --restart=Never \
  -n teranode \
  -- kafka-topics --list --bootstrap-server kafka:9092
```

**Database Connection Failed**:
```bash
# Test PostgreSQL
kubectl exec -it -n teranode postgres-0 -- \
  psql -U teranode -d teranode -c "SELECT 1;"

# Check secrets
kubectl get secret postgres-secret -n teranode -o yaml
```

## Best Practices

1. **Use systemd** for legacy node management (auto-restart, logging)
2. **Implement monitoring** from day one (Prometheus, Grafana)
3. **Automate backups** with tested restore procedures
4. **Use Kubernetes** for Teranode production deployments
5. **Scale horizontally** with Teranode for high throughput
6. **Implement proper security** (firewalls, secrets management, TLS)
7. **Plan for disaster recovery** with documented procedures
8. **Monitor resource usage** and set up alerts
9. **Test upgrades** in staging environment first
10. **Keep infrastructure as code** (Git-tracked configurations)

## Common Pitfalls

1. **Insufficient storage** - Blockchain grows continuously
2. **No backup strategy** - Data loss can be catastrophic
3. **Weak RPC passwords** - Secure your RPC endpoint
4. **Single point of failure** - Use redundancy for production
5. **Ignoring monitoring** - Problems go unnoticed
6. **Manual configuration** - Use infrastructure as code
7. **No disaster recovery plan** - Recovery takes too long

## Hands-On Project: Production Teranode Deployment

Deploy a complete production-grade Teranode infrastructure:

**Requirements**:
- Kubernetes cluster (3+ nodes)
- Helm 3.x
- kubectl configured

**Steps**:

1. **Deploy Infrastructure**:
```bash
# Add Bitnami repo for dependencies
helm repo add bitnami https://charts.bitnami.com/bitnami

# Install Kafka
helm install kafka bitnami/kafka -n teranode

# Install PostgreSQL
helm install postgres bitnami/postgresql -n teranode
```

2. **Deploy Teranode Services**:
```bash
# Deploy all services
kubectl apply -k ./teranode-manifests/

# Verify deployment
kubectl get all -n teranode
```

3. **Configure Monitoring**:
```bash
# Install Prometheus Operator
helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring

# Apply Teranode service monitors
kubectl apply -f teranode-monitoring.yaml
```

4. **Load Test**:
```bash
# Run load test
kubectl apply -f load-test-job.yaml

# Monitor performance
kubectl port-forward -n monitoring svc/prometheus 9090:9090
# Open http://localhost:9090
```

## Next Steps

Continue to:
- **[Advanced Scripting](../advanced-scripting/README.md)** - Complex smart contracts and advanced Script features
- **[Custom Protocols](../custom-protocols/README.md)** - Design application-specific protocols

## Additional Resources

- [Teranode Documentation](https://bsv-blockchain.github.io/teranode/)
- [Bitcoin SV Node GitHub](https://github.com/bitcoin-sv/bitcoin-sv)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Prometheus Best Practices](https://prometheus.io/docs/practices/)
- [Kafka Operations Guide](https://kafka.apache.org/documentation/#operations)
- [BSV Wiki - Running a Node](https://wiki.bitcoinsv.io/)

---

**Status:** ✅ Complete - Ready for learning
