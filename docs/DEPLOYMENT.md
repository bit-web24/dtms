# DTMS Deployment Guide

## Overview

This guide covers various deployment strategies for the DTMS (Distributed Task Management System), from local development to production environments. The system is designed to be container-native and can be deployed using Docker, Kubernetes, or other container orchestration platforms.

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Local Development Deployment](#local-development-deployment)
3. [Staging Environment](#staging-environment)
4. [Production Deployment](#production-deployment)
5. [Kubernetes Deployment](#kubernetes-deployment)
6. [Environment Configuration](#environment-configuration)
7. [Security Considerations](#security-considerations)
8. [Monitoring and Logging](#monitoring-and-logging)
9. [Backup and Recovery](#backup-and-recovery)
10. [Troubleshooting](#troubleshooting)

## Prerequisites

### Infrastructure Requirements

#### Minimum Requirements
- **CPU**: 2 cores per service
- **Memory**: 2GB RAM per service
- **Storage**: 10GB for databases
- **Network**: 1Gbps bandwidth

#### Production Requirements
- **CPU**: 4 cores per service instance
- **Memory**: 4GB RAM per service instance
- **Storage**: 100GB+ SSD for databases
- **Network**: 10Gbps bandwidth
- **High Availability**: Multiple availability zones

### Software Requirements
- Docker 20.10+
- Docker Compose 2.0+
- Kubernetes 1.24+ (for K8s deployment)
- kubectl and helm (for K8s deployment)
- A container registry (Docker Hub, GCR, ECR, etc.)

## Local Development Deployment

### Quick Start

```bash
# Clone the repository
git clone https://github.com/bit-web24/DTMS.git
cd DTMS

# Compile protocol buffers
make

# Start all services
docker-compose up -d --build

# Verify services are running
docker-compose ps
```

### Development Docker Compose

Create `docker-compose.dev.yml`:

```yaml
version: '3.8'

services:
  user_service:
    build:
      context: .
      dockerfile: ./services/user/Dockerfile
    ports:
      - "50051:50051"
      - "8081:8081"
    env_file:
      - ./services/user/.env
    depends_on:
      postgres_user:
        condition: service_healthy
    volumes:
      - ./services/user:/app
      - /app/proto  # Exclude generated files
    networks:
      - user_net
      - service_net
    command: ["air"]  # Use air for hot reload

  # Add hot reload for other services similarly

volumes:
  postgres-user-data:
    driver: local
  postgres-task-data:
    driver: local

networks:
  service_net:
  user_net:
  task_net:
```

### Using Docker Compose Profiles

```yaml
# In docker-compose.yml
services:
  user_service:
    profiles:
      - core
    # ... rest of config

  redis:
    profiles:
      - cache
    image: redis:alpine
    # ... config

  prometheus:
    profiles:
      - monitoring
    image: prom/prometheus
    # ... config
```

Usage:
```bash
# Start core services only
docker-compose --profile core up -d

# Start with cache
docker-compose --profile core --profile cache up -d

# Start full stack
docker-compose --profile core --profile cache --profile monitoring up -d
```

## Staging Environment

### Staging Docker Compose

Create `docker-compose.staging.yml`:

```yaml
version: '3.8'

services:
  user_service:
    image: bit-web24/dtms-user:${STAGING_TAG}
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '1'
          memory: 1G
        reservations:
          cpus: '0.5'
          memory: 512M
    environment:
      - ENV=staging
      - LOG_LEVEL=debug

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.staging.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - dtms

volumes:
  postgres-user-data:
    driver: local
  postgres-task-data:
    driver: local
```

### Environment Configuration

Create `.staging.env`:

```bash
# Common
COMPOSE_PROJECT_NAME=dtms-staging
STAGING_TAG=staging-latest

# User Service
DB_HOST=postgres-user-staging.internal
DB_USER=dtms_user
DB_PASSWORD=${STAGING_DB_PASSWORD}
DB_NAME=dtms_users_staging

# Task Service
DB_HOST=postgres-task-staging.internal
DB_USER=dtms_task
DB_PASSWORD=${STAGING_DB_PASSWORD}
DB_NAME=dtms_tasks_staging
```

### Deployment Script

Create `scripts/deploy-staging.sh`:

```bash
#!/bin/bash

set -e

echo "Deploying to staging environment..."

# Load environment variables
source .staging.env

# Pull latest images
docker-compose -f docker-compose.yml -f docker-compose.staging.yml pull

# Run database migrations
docker-compose -f docker-compose.yml -f docker-compose.staging.yml run --rm user_service migrate
docker-compose -f docker-compose.yml -f docker-compose.staging.yml run --rm task_service migrate

# Deploy services
docker-compose -f docker-compose.yml -f docker-compose.staging.yml up -d

# Run smoke tests
./scripts/smoke-tests.sh staging

echo "Staging deployment completed successfully!"
```

## Production Deployment

### Production Docker Compose

Create `docker-compose.prod.yml`:

```yaml
version: '3.8'

services:
  user_service:
    image: bit-web24/dtms-user:${PROD_TAG}
    deploy:
      replicas: 3
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
      resources:
        limits:
          cpus: '2'
          memory: 2G
        reservations:
          cpus: '1'
          memory: 1G
    environment:
      - ENV=production
      - LOG_LEVEL=info
      - DB_HOST=${DB_HOST}
      - DB_PASSWORD=${DB_PASSWORD}
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:8081/health || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.prod.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl:ro
      - nginx_logs:/var/log/nginx
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 512M

volumes:
  postgres-user-data:
    driver: local
    driver_opts:
      type: nfs
      o: addr=nfs-server,nfsvers=4,rw
      device: ":/mnt/data/postgres-user"
  nginx_logs:
    driver: local
```

### Production Configuration

Create `.prod.env`:

```bash
# Production configuration
COMPOSE_PROJECT_NAME=dtms-prod
PROD_TAG=${BUILD_TAG}

# Database
DB_HOST=database.internal
DB_USER=dtms_prod
DB_PASSWORD=${PROD_DB_PASSWORD}
DB_SSLMODE=require
DB_TIME_ZONE=UTC

# Performance
RPC_MAX_CONNECTIONS=100
DB_MAX_CONNECTIONS=50
DB_MAX_IDLE_CONNECTIONS=10

# Security
JWT_SECRET=${JWT_SECRET}
API_KEY=${API_KEY}
```

### Deployment Pipeline

Create `.github/workflows/production.yml`:

```yaml
name: Production Deployment

on:
  push:
    tags:
      - 'v*'

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    environment: production
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2
    
    - name: Login to Registry
      uses: docker/login-action@v2
      with:
        registry: ${{ secrets.REGISTRY_URL }}
        username: ${{ secrets.REGISTRY_USERNAME }}
        password: ${{ secrets.REGISTRY_PASSWORD }}
    
    - name: Build and push images
      run: |
        export BUILD_TAG=${GITHUB_REF#refs/tags/}
        docker-compose build
        docker-compose push
    
    - name: Deploy to production
      uses: appleboy/ssh-action@master
      with:
        host: ${{ secrets.PROD_HOST }}
        username: ${{ secrets.PROD_USER }}
        key: ${{ secrets.PROD_SSH_KEY }}
        script: |
          cd /opt/dtms
          git pull origin main
          export BUILD_TAG=${GITHUB_REF#refs/tags/}
          docker-compose -f docker-compose.yml -f docker-compose.prod.yml pull
          docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d
          ./scripts/health-check.sh
```

## Kubernetes Deployment

### Namespace and ConfigMaps

Create `k8s/namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dtms
  labels:
    name: dtms
```

Create `k8s/configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: dtms-config
  namespace: dtms
data:
  DB_HOST: "postgres-service"
  DB_PORT: "5432"
  DB_SSLMODE: "require"
  DB_TIME_ZONE: "UTC"
  LOG_LEVEL: "info"
---
apiVersion: v1
kind: Secret
metadata:
  name: dtms-secrets
  namespace: dtms
type: Opaque
data:
  DB_PASSWORD: <base64-encoded-password>
  JWT_SECRET: <base64-encoded-jwt-secret>
```

### Database Deployment

Create `k8s/postgres.yaml`:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-user
  namespace: dtms
spec:
  serviceName: postgres-user
  replicas: 1
  selector:
    matchLabels:
      app: postgres-user
  template:
    metadata:
      labels:
        app: postgres-user
    spec:
      containers:
      - name: postgres
        image: postgres:15
        env:
        - name: POSTGRES_DB
          value: "dtms_users"
        - name: POSTGRES_USER
          value: "dtms_user"
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: dtms-secrets
              key: DB_PASSWORD
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "ssd"
      resources:
        requests:
          storage: 20Gi
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-user-service
  namespace: dtms
spec:
  selector:
    app: postgres-user
  ports:
  - port: 5432
    targetPort: 5432
```

### Application Deployment

Create `k8s/user-service.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: dtms
  labels:
    app: user-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      containers:
      - name: user-service
        image: bit-web24/dtms-user:latest
        ports:
        - containerPort: 50051
          name: grpc
        - containerPort: 8081
          name: health
        env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: dtms-config
              key: DB_HOST
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: dtms-secrets
              key: DB_PASSWORD
        - name: DB_NAME
          value: "dtms_users"
        - name: DB_USER
          value: "dtms_user"
        - name: RPC_PORT
          value: "50051"
        - name: HTTP_PORT
          value: "8081"
        livenessProbe:
          httpGet:
            path: /health
            port: 8081
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 8081
          initialDelaySeconds: 5
          periodSeconds: 5
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: user-service
  namespace: dtms
spec:
  selector:
    app: user-service
  ports:
  - name: grpc
    port: 50051
    targetPort: 50051
  - name: health
    port: 8081
    targetPort: 8081
```

### Ingress Configuration

Create `k8s/ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dtms-ingress
  namespace: dtms
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/grpc-backend: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
spec:
  tls:
  - hosts:
    - api.dtms.example.com
    secretName: dtms-api-tls
  rules:
  - host: api.dtms.example.com
    http:
      paths:
      - path: /v1/users
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 50051
      - path: /v1/tasks
        pathType: Prefix
        backend:
          service:
            name: task-service
            port:
              number: 50052
```

### Horizontal Pod Autoscaler

Create `k8s/hpa.yaml`:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: user-service-hpa
  namespace: dtms
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: user-service
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

### Helm Chart

Create `helm/dtms/Chart.yaml`:

```yaml
apiVersion: v2
name: dtms
description: DTMS Helm Chart
type: application
version: 0.1.0
appVersion: "1.0.0"
dependencies:
  - name: postgresql
    version: 12.x.x
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
```

Create `helm/dtms/values.yaml`:

```yaml
# Default values for DTMS chart

replicaCount: 3

image:
  repository: bit-web24/dtms-user
  pullPolicy: IfNotPresent
  tag: "latest"

service:
  type: ClusterIP
  grpcPort: 50051
  healthPort: 8081

ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: api.dtms.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: dtms-tls
      hosts:
        - api.dtms.example.com

resources:
  limits:
    cpu: 500m
    memory: 1Gi
  requests:
    cpu: 250m
    memory: 512Mi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

postgresql:
  enabled: true
  auth:
    postgresPassword: "changeme"
    database: "dtms"
  primary:
    persistence:
      enabled: true
      size: 20Gi
```

Deploy with Helm:
```bash
# Install
helm install dtms ./helm/dtms -n dtms --create-namespace

# Upgrade
helm upgrade dtms ./helm/dtms -n dtms

# Uninstall
helm uninstall dtms -n dtms
```

## Environment Configuration

### Configuration Management

Create `config/base/configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: dtms-config
  namespace: dtms
data:
  config.yaml: |
    database:
      host: ${DB_HOST}
      port: ${DB_PORT}
      sslmode: ${DB_SSLMODE}
      timezone: ${DB_TIME_ZONE}
    
    server:
      rpc_port: ${RPC_PORT}
      http_port: ${HTTP_PORT}
    
    logging:
      level: ${LOG_LEVEL}
      format: json
```

### Secrets Management

Using Sealed Secrets:
```bash
# Install sealed-secrets controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.19.5/controller.yaml

# Create secret
kubectl create secret generic dtms-secrets --dry-run=client -o yaml > dtms-secrets.yaml

# Seal the secret
kubeseal < dtms-secrets.yaml > dtms-sealed-secret.yaml

# Apply sealed secret
kubectl apply -f dtms-sealed-secret.yaml
```

### Environment-specific Configurations

#### Development
```yaml
# config/development/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: dtms-config-dev
data:
  LOG_LEVEL: "debug"
  DB_SSLMODE: "disable"
  RPC_MAX_CONNECTIONS: "10"
```

#### Production
```yaml
# config/production/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: dtms-config-prod
data:
  LOG_LEVEL: "info"
  DB_SSLMODE: "require"
  RPC_MAX_CONNECTIONS: "100"
  ENABLE_METRICS: "true"
  ENABLE_TRACING: "true"
```

## Security Considerations

### Network Policies

Create `k8s/network-policy.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: dtms-network-policy
  namespace: dtms
spec:
  podSelector: {}
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
          app: user-service
    ports:
    - protocol: TCP
      port: 50051
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
```

### Pod Security Policies

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: dtms-psp
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
```

### RBAC

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dtms-service-account
  namespace: dtms
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dtms
  name: dtms-role
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dtms-role-binding
  namespace: dtms
subjects:
- kind: ServiceAccount
  name: dtms-service-account
  namespace: dtms
roleRef:
  kind: Role
  name: dtms-role
  apiGroup: rbac.authorization.k8s.io
```

## Monitoring and Logging

### Prometheus Monitoring

Create `k8s/monitoring.yaml`:

```yaml
apiVersion: v1
kind: ServiceMonitor
metadata:
  name: dtms-monitor
  namespace: dtms
  labels:
    app: dtms
spec:
  selector:
    matchLabels:
      app: user-service
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
---
apiVersion: v1
kind: Service
metadata:
  name: user-service-metrics
  namespace: dtms
  labels:
    app: user-service
spec:
  selector:
    app: user-service
  ports:
  - name: metrics
    port: 9090
    targetPort: 9090
```

### Grafana Dashboards

Create `k8s/grafana-dashboard.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: dtms-grafana-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  dtms-dashboard.json: |
    {
      "dashboard": {
        "title": "DTMS Dashboard",
        "panels": [
          {
            "title": "Request Rate",
            "type": "graph",
            "targets": [
              {
                "expr": "rate(grpc_server_started_total{service=\"user_service\"}[5m])"
              }
            ]
          }
        ]
      }
    }
```

### ELK Stack for Logging

Create `k8s/filebeat.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
data:
  filebeat.yml: |
    filebeat.inputs:
    - type: container
      paths:
        - /var/log/containers/*dtms*.log
      
    output.elasticsearch:
      hosts: ["elasticsearch:9200"]
      
    setup.kibana:
      host: "kibana:5601"
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: filebeat
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: filebeat
  template:
    metadata:
      labels:
        name: filebeat
    spec:
      serviceAccountName: filebeat
      terminationGracePeriodSeconds: 30
      containers:
      - name: filebeat
        image: docker.elastic.co/beats/filebeat:7.15.0
        args: [
          "-c", "/etc/filebeat.yml",
          "-e",
        ]
        securityContext:
          runAsUser: 0
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 100Mi
        volumeMounts:
        - name: config
          mountPath: /etc/filebeat.yml
          readOnly: true
          subPath: filebeat.yml
        - name: data
          mountPath: /usr/share/filebeat/data
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      volumes:
      - name: config
        configMap:
          defaultMode: 0600
          name: filebeat-config
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: data
        hostPath:
          path: /var/lib/filebeat-data
          type: DirectoryOrCreate
```

## Backup and Recovery

### Database Backup Script

Create `scripts/backup-database.sh`:

```bash
#!/bin/bash

set -e

BACKUP_DIR="/backups"
DATE=$(date +%Y%m%d_%H%M%S)
DB_NAME="$1"
DB_HOST="$2"
DB_USER="$3"
DB_PASS="$4"

if [ -z "$DB_NAME" ] || [ -z "$DB_HOST" ]; then
    echo "Usage: $0 <db_name> <db_host> [db_user] [db_pass]"
    exit 1
fi

# Create backup directory
mkdir -p $BACKUP_DIR/$DATE

# Backup database
docker exec postgres-$DB_NAME pg_dump -U $DB_USER -d $DB_NAME > $BACKUP_DIR/$DATE/$DB_NAME.sql

# Compress backup
gzip $BACKUP_DIR/$DATE/$DB_NAME.sql

# Upload to S3 (if configured)
if [ -n "$S3_BUCKET" ]; then
    aws s3 cp $BACKUP_DIR/$DATE/$DB_NAME.sql.gz s3://$S3_BUCKET/backups/$DATE/
fi

# Clean old backups (keep last 7 days)
find $BACKUP_DIR -type d -mtime +7 -exec rm -rf {} +

echo "Backup completed: $BACKUP_DIR/$DATE/$DB_NAME.sql.gz"
```

### CronJob for Automated Backups

Create `k8s/backup-cronjob.yaml`:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: database-backup
  namespace: dtms
spec:
  schedule: "0 2 * * *"  # Run daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: postgres-backup
            image: postgres:15
            command:
            - /bin/bash
            - -c
            - |
              pg_dump -h $DB_HOST -U $DB_USER -d $DB_NAME | gzip > /backup/$(date +%Y%m%d_%H%M%S).sql.gz
            env:
            - name: DB_HOST
              value: "postgres-service"
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: dtms-secrets
                  key: DB_USER
            - name: DB_NAME
              value: "dtms_users"
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: dtms-secrets
                  key: DB_PASSWORD
            volumeMounts:
            - name: backup-storage
              mountPath: /backup
          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: backup-pvc
          restartPolicy: OnFailure
```

### Disaster Recovery Plan

1. **Database Recovery**:
```bash
# Stop all services
kubectl scale deployment user-service --replicas=0 -n dtms
kubectl scale deployment task-service --replicas=0 -n dtms

# Restore database
kubectl run postgres-restore --image=postgres:15 --rm -i --restart=Never -- \
  psql -h postgres-service -U $DB_USER -d $DB_NAME < backup.sql

# Restart services
kubectl scale deployment user-service --replicas=3 -n dtms
kubectl scale deployment task-service --replicas=3 -n dtms
```

2. **Full Cluster Recovery**:
```bash
# Restore from etcd backup
ETCDCTL_API=3 etcdctl snapshot restore backup.db \
  --data-dir /var/lib/etcd-backup

# Restore persistent volumes
kubectl apply -f backup-pvc.yaml

# Restore all resources
kubectl apply -f k8s/
```

## Troubleshooting

### Common Issues and Solutions

#### 1. Service Not Starting

```bash
# Check pod status
kubectl get pods -n dtms

# Describe pod for errors
kubectl describe pod <pod-name> -n dtms

# Check logs
kubectl logs <pod-name> -n dtms

# Check events
kubectl get events -n dtms --sort-by='.lastTimestamp'
```

#### 2. Database Connection Issues

```bash
# Test database connectivity
kubectl exec -it <postgres-pod> -n dtms -- psql -U $DB_USER -d $DB_NAME

# Check database service
kubectl get svc -n dtms

# Check endpoints
kubectl get endpoints -n dtms

# Test from application pod
kubectl exec -it <app-pod> -n dtms -- nc -zv postgres-service 5432
```

#### 3. Performance Issues

```bash
# Check resource usage
kubectl top pods -n dtms
kubectl top nodes

# Check metrics
kubectl get --raw /metrics | grep grpc

# Analyze with Prometheus
# Query: rate(grpc_server_started_total[5m])
```

#### 4. Network Issues

```bash
# Check network policies
kubectl get networkpolicy -n dtms

# Test connectivity
kubectl exec -it <pod1> -n dtms -- ping <pod2-ip>

# Check DNS
kubectl exec -it <pod> -n dtms -- nslookup postgres-service.dtms.svc.cluster.local
```

### Debug Commands

```bash
# Port forward to local
kubectl port-forward svc/user-service 50051:50051 -n dtms

# Execute shell in container
kubectl exec -it <pod-name> -n dtms -- /bin/sh

# Check environment variables
kubectl exec -it <pod-name> -n dtms -- env | grep DTMS

# Debug with breakpoint (if using delve)
kubectl port-forward <pod-name> 40000:40000 -n dtms
dlv connect localhost:40000
```

### Health Checks

Create `scripts/health-check.sh`:

```bash
#!/bin/bash

NAMESPACE="dtms"
SERVICES=("user-service" "task-service" "api-gateway")

for service in "${SERVICES[@]}"; do
    echo "Checking $service..."
    
    # Check deployment
    kubectl get deployment $service -n $NAMESPACE
    
    # Check pods
    kubectl get pods -l app=$service -n $NAMESPACE
    
    # Check service
    kubectl get svc $service -n $NAMESPACE
    
    # Check health endpoint
    if kubectl get svc $service -n $NAMESPACE | grep -q "808[0-9]"; then
        kubectl exec -it $(kubectl get pods -l app=$service -n $NAMESPACE -o jsonpath='{.items[0].metadata.name}') -n $NAMESPACE -- curl http://localhost:8081/health
    fi
    
    echo "------------------------"
done
```

## Best Practices

### 1. Resource Management
- Set appropriate resource requests and limits
- Use Horizontal Pod Autoscaling
- Monitor resource utilization

### 2. Security
- Use Network Policies
- Implement RBAC
- Encrypt secrets and sensitive data
- Regular security scans

### 3. High Availability
- Deploy across multiple availability zones
- Use readiness and liveness probes
- Implement proper health checks
- Configure pod disruption budgets

### 4. Observability
- Implement structured logging
- Use Prometheus for metrics
- Set up distributed tracing
- Create meaningful alerts

### 5. CI/CD
- Automate testing and deployment
- Use git-based workflows
- Implement canary deployments
- Rollback strategies

This deployment guide provides comprehensive instructions for deploying DTMS in various environments. Adjust configurations based on your specific requirements and infrastructure.