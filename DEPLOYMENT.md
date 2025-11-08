# Deployment Guide

This guide covers deploying RL-Arena to various environments, from local development to production Kubernetes clusters.

## Table of Contents

- [Local Development](#local-development)
- [Docker Compose Deployment](#docker-compose-deployment)
- [Kubernetes Deployment](#kubernetes-deployment)
- [Production Checklist](#production-checklist)
- [Monitoring and Maintenance](#monitoring-and-maintenance)
- [Troubleshooting](#troubleshooting)

## Local Development

### Prerequisites

- **Go 1.21+** for backend
- **Python 3.10+** for executor
- **Node.js 18+** for frontend
- **PostgreSQL 15+**
- **Docker** and **Docker Compose**
- **kubectl** for Kubernetes
- **Minikube** or other local Kubernetes cluster

### Quick Start

1. **Clone all repositories**:

```bash
# Create workspace
mkdir rl-arena && cd rl-arena

# Clone repositories
git clone https://github.com/rl-arena/rl-arena-backend.git
git clone https://github.com/rl-arena/rl-arena-executor.git
git clone https://github.com/rl-arena/rl-arena-web.git
git clone https://github.com/rl-arena/rl-arena-env.git
```

2. **Start backend with Docker Compose**:

```bash
cd rl-arena-backend
cp .env.example .env
docker-compose up -d
```

This starts:
- Backend API server (port 8080)
- PostgreSQL database (port 5433)

3. **Start executor**:

```bash
cd rl-arena-executor
pip install -r requirements.txt
python -m executor.server
```

Executor gRPC server runs on port 50051.

4. **Start frontend**:

```bash
cd rl-arena-web
npm install
npm run dev
```

Frontend runs on port 5173.

5. **Verify setup**:

```bash
# Check backend
curl http://localhost:8080/health

# Check frontend
open http://localhost:5173

# Check API docs
open http://localhost:8080/swagger/index.html
```

## Docker Compose Deployment

### Full Stack with Docker Compose

Create `docker-compose.yml` in your workspace root:

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: rl_arena
    ports:
      - "5433:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  backend:
    build:
      context: ./rl-arena-backend
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_USER=postgres
      - DB_PASSWORD=postgres
      - DB_NAME=rl_arena
      - DB_SSL_MODE=disable
      - JWT_SECRET=your-secret-key-change-in-production
      - EXECUTOR_GRPC_ADDRESS=executor:50051
      - CORS_ALLOWED_ORIGINS=http://localhost:5173
    depends_on:
      postgres:
        condition: service_healthy
    restart: unless-stopped

  executor:
    build:
      context: ./rl-arena-executor
      dockerfile: Dockerfile
    ports:
      - "50051:50051"
    environment:
      - EXECUTOR_HOST=0.0.0.0
      - EXECUTOR_PORT=50051
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - executor_replays:/tmp/replays
    restart: unless-stopped

  web:
    build:
      context: ./rl-arena-web
      dockerfile: Dockerfile
    ports:
      - "80:80"
    environment:
      - VITE_API_URL=http://localhost:8080/api/v1
      - VITE_WS_URL=ws://localhost:8080/api/v1/ws
    depends_on:
      - backend
    restart: unless-stopped

volumes:
  postgres_data:
  executor_replays:
```

**Start the stack**:

```bash
docker-compose up -d
```

**Check status**:

```bash
docker-compose ps
docker-compose logs -f
```

## Kubernetes Deployment

### Prerequisites

- Kubernetes cluster (1.28+)
- kubectl configured
- Container registry (Docker Hub, GCR, ECR, ACR)
- Persistent storage provider

### 1. Build and Push Images

```bash
# Set your registry
export REGISTRY=your-registry.io/rl-arena

# Backend
cd rl-arena-backend
docker build -t $REGISTRY/backend:latest .
docker push $REGISTRY/backend:latest

# Executor
cd rl-arena-executor
docker build -t $REGISTRY/executor:latest .
docker push $REGISTRY/executor:latest

# Orchestrator
docker build -f Dockerfile.orchestrator -t $REGISTRY/orchestrator:latest .
docker push $REGISTRY/orchestrator:latest

# Web
cd rl-arena-web
docker build -t $REGISTRY/web:latest .
docker push $REGISTRY/web:latest
```

### 2. Create Namespace

```bash
kubectl create namespace rl-arena
```

### 3. Create Secrets

```bash
# Database credentials
kubectl create secret generic postgres-secret \
  --from-literal=postgres-password=YourSecurePassword \
  -n rl-arena

# JWT secret
kubectl create secret generic jwt-secret \
  --from-literal=jwt-secret=$(openssl rand -base64 32) \
  -n rl-arena

# Container registry (if private)
kubectl create secret docker-registry regcred \
  --docker-server=$REGISTRY \
  --docker-username=your-username \
  --docker-password=your-password \
  --docker-email=your-email \
  -n rl-arena
```

### 4. Deploy PostgreSQL

```yaml
# postgres-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: rl-arena
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
        env:
        - name: POSTGRES_DB
          value: rl_arena
        - name: POSTGRES_USER
          value: postgres
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: postgres-password
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
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
  namespace: rl-arena
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
  clusterIP: None
```

```bash
kubectl apply -f postgres-statefulset.yaml
```

### 5. Deploy Backend

```yaml
# backend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rl-arena-backend
  namespace: rl-arena
spec:
  replicas: 3
  selector:
    matchLabels:
      app: rl-arena-backend
  template:
    metadata:
      labels:
        app: rl-arena-backend
    spec:
      imagePullSecrets:
      - name: regcred
      containers:
      - name: backend
        image: your-registry.io/rl-arena/backend:latest
        ports:
        - containerPort: 8080
        env:
        - name: DB_HOST
          value: postgres.rl-arena.svc.cluster.local
        - name: DB_PORT
          value: "5432"
        - name: DB_USER
          value: postgres
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: postgres-password
        - name: DB_NAME
          value: rl_arena
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: jwt-secret
              key: jwt-secret
        - name: EXECUTOR_GRPC_ADDRESS
          value: rl-arena-executor.rl-arena.svc.cluster.local:50051
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: rl-arena-backend
  namespace: rl-arena
spec:
  selector:
    app: rl-arena-backend
  ports:
  - port: 8080
    targetPort: 8080
  type: ClusterIP
```

```bash
kubectl apply -f backend-deployment.yaml
```

### 6. Deploy Executor

Use the existing `k8s/deployment.yaml` from rl-arena-executor repository:

```bash
cd rl-arena-executor
kubectl apply -f k8s/deployment.yaml
```

### 7. Deploy Web Frontend

```yaml
# web-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rl-arena-web
  namespace: rl-arena
spec:
  replicas: 2
  selector:
    matchLabels:
      app: rl-arena-web
  template:
    metadata:
      labels:
        app: rl-arena-web
    spec:
      imagePullSecrets:
      - name: regcred
      containers:
      - name: web
        image: your-registry.io/rl-arena/web:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  name: rl-arena-web
  namespace: rl-arena
spec:
  selector:
    app: rl-arena-web
  ports:
  - port: 80
    targetPort: 80
  type: LoadBalancer
```

```bash
kubectl apply -f web-deployment.yaml
```

### 8. Configure Ingress (Optional)

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rl-arena-ingress
  namespace: rl-arena
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
  - hosts:
    - rl-arena.yourdomain.com
    - api.rl-arena.yourdomain.com
    secretName: rl-arena-tls
  rules:
  - host: rl-arena.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: rl-arena-web
            port:
              number: 80
  - host: api.rl-arena.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: rl-arena-backend
            port:
              number: 8080
```

```bash
kubectl apply -f ingress.yaml
```

## Production Checklist

### Security

- [ ] Change default passwords and secrets
- [ ] Use strong JWT secret (256-bit minimum)
- [ ] Enable HTTPS/TLS with valid certificates
- [ ] Configure CORS for production domains
- [ ] Enable PostgreSQL SSL mode
- [ ] Use private container registry
- [ ] Scan Docker images for vulnerabilities
- [ ] Enable Kubernetes RBAC
- [ ] Use network policies for pod isolation
- [ ] Enable pod security policies

### Performance

- [ ] Configure resource limits for all pods
- [ ] Enable horizontal pod autoscaling
- [ ] Set up database connection pooling
- [ ] Configure caching (leaderboard, stats)
- [ ] Use CDN for static assets
- [ ] Optimize database indexes
- [ ] Enable gRPC keepalive
- [ ] Configure appropriate replica counts

### Reliability

- [ ] Set up liveness and readiness probes
- [ ] Configure pod disruption budgets
- [ ] Enable automatic restarts
- [ ] Use StatefulSet for database
- [ ] Configure persistent volumes
- [ ] Set up database backups
- [ ] Enable log aggregation
- [ ] Configure monitoring and alerts

### Monitoring

- [ ] Set up Prometheus metrics
- [ ] Configure Grafana dashboards
- [ ] Enable application logging
- [ ] Set up error tracking (Sentry)
- [ ] Configure uptime monitoring
- [ ] Set up alerting rules
- [ ] Monitor resource usage
- [ ] Track API response times

## Monitoring and Maintenance

### Kubernetes Monitoring

```bash
# Check pod status
kubectl get pods -n rl-arena

# View logs
kubectl logs -n rl-arena -l app=rl-arena-backend -f

# Check resource usage
kubectl top pods -n rl-arena
kubectl top nodes

# View events
kubectl get events -n rl-arena --sort-by='.lastTimestamp'
```

### Database Maintenance

```bash
# Backup database
kubectl exec -n rl-arena postgres-0 -- \
  pg_dump -U postgres rl_arena > backup.sql

# Restore database
kubectl exec -i -n rl-arena postgres-0 -- \
  psql -U postgres rl_arena < backup.sql

# Run migrations
kubectl exec -it -n rl-arena deployment/rl-arena-backend -- \
  /app/migrate up
```

### Scaling

```bash
# Scale backend
kubectl scale deployment rl-arena-backend -n rl-arena --replicas=5

# Scale executor
kubectl scale deployment rl-arena-executor -n rl-arena --replicas=3

# Enable autoscaling
kubectl autoscale deployment rl-arena-backend -n rl-arena \
  --min=3 --max=10 --cpu-percent=80
```

## Troubleshooting

### Common Issues

#### Pods Not Starting

```bash
# Check pod status
kubectl describe pod -n rl-arena <pod-name>

# Common fixes:
# - Verify image exists and is accessible
# - Check resource limits
# - Verify secrets exist
# - Check PVC binding
```

#### Database Connection Issues

```bash
# Test database connection
kubectl exec -it -n rl-arena postgres-0 -- psql -U postgres -d rl_arena

# Check connection string
kubectl exec -it -n rl-arena deployment/rl-arena-backend -- env | grep DB_
```

#### Executor Can't Create Jobs

```bash
# Check RBAC permissions
kubectl get rolebinding -n rl-arena

# Verify ServiceAccount
kubectl get serviceaccount rl-arena-executor -n rl-arena

# Check executor logs
kubectl logs -n rl-arena -l app=rl-arena-executor
```

#### High Memory Usage

```bash
# Check resource usage
kubectl top pods -n rl-arena

# Increase limits if needed
kubectl set resources deployment rl-arena-backend -n rl-arena \
  --limits=memory=1Gi --requests=memory=512Mi
```

### Health Checks

```bash
# Check all services
kubectl get all -n rl-arena

# Test backend health
kubectl exec -it -n rl-arena deployment/rl-arena-backend -- \
  curl http://localhost:8080/health

# Test database
kubectl exec -it -n rl-arena postgres-0 -- \
  pg_isready -U postgres
```

### Logs

```bash
# Backend logs
kubectl logs -n rl-arena -l app=rl-arena-backend --tail=100 -f

# Executor logs
kubectl logs -n rl-arena -l app=rl-arena-executor --tail=100 -f

# Match execution logs
kubectl logs -n rl-arena job/match-<match-id> --all-containers

# Database logs
kubectl logs -n rl-arena postgres-0 --tail=100 -f
```

---

For more information:
- [Architecture](ARCHITECTURE.md) - System architecture
- [API Reference](API_REFERENCE.md) - API documentation
- [Development Guide](DEVELOPMENT.md) - Development setup
