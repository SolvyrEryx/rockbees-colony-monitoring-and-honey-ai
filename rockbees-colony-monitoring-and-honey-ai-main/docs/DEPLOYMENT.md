# 🚀 Deployment Guide
## RockBees - Production Deployment

---

## Table of Contents

1. [Deployment Overview](#deployment-overview)
2. [Docker Containerization](#docker-containerization)
3. [Kubernetes Deployment](#kubernetes-deployment)
4. [Cloud Deployment (AWS/GCP)](#cloud-deployment)
5. [CI/CD Pipeline](#cicd-pipeline)
6. [Production Checklist](#production-checklist)
7. [Monitoring & Logging](#monitoring--logging)
8. [Scaling & Auto-scaling](#scaling--auto-scaling)

---

## Deployment Overview

### Deployment Architecture

```
┌────────────────────────────────────────┐
│   DNS & CDN (CloudFront/Cloudflare)    │
└──────────────┬─────────────────────────┘
               │
    ┌──────────▼──────────┐
    │  Load Balancer      │
    │  (ALB/NLB)          │
    └──────────┬──────────┘
               │
    ┌──────────┴──────────┐
    │                     │
    ▼                     ▼
┌─────────────┐      ┌─────────────┐
│ API Pod 1   │      │ API Pod 2   │
│ (Container) │      │ (Container) │
└──────┬──────┘      └──────┬──────┘
       │                    │
       └──────────┬─────────┘
                  │
         ┌────────▼────────┐
         │  Data Layer     │
         ├─────────────────┤
         │ PostgreSQL (RDS)│
         │ Redis (Cache)   │
         │ S3 (Storage)    │
         └─────────────────┘
```

---

## Docker Containerization

### Dockerfile (Backend)

```dockerfile
# Multi-stage build for optimization
FROM python:3.9-slim as builder

WORKDIR /app

# Install build dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    postgresql-client \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements and install
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# Final stage
FROM python:3.9-slim

WORKDIR /app

# Install runtime dependencies only
RUN apt-get update && apt-get install -y \
    postgresql-client \
    && rm -rf /var/lib/apt/lists/*

# Copy Python packages from builder
COPY --from=builder /root/.local /root/.local
ENV PATH=/root/.local/bin:$PATH

# Copy application code
COPY . .

# Create non-root user
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD python -c "import requests; requests.get('http://localhost:8000/health')" || exit 1

# Run application
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]

EXPOSE 8000
```

### Docker Compose for Production

```yaml
version: '3.8'

services:
  postgres:
    image: postgis/postgis:15-3.3
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_NAME}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./scripts/init_db.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "5432:5432"
    networks:
      - rockbees
    restart: always
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    networks:
      - rockbees
    restart: always
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  api:
    build:
      context: ./backend
      dockerfile: Dockerfile
    environment:
      DATABASE_URL: postgresql://${DB_USER}:${DB_PASSWORD}@postgres:5432/${DB_NAME}
      REDIS_URL: redis://:${REDIS_PASSWORD}@redis:6379
      JWT_SECRET: ${JWT_SECRET}
      AWS_ACCESS_KEY_ID: ${AWS_ACCESS_KEY_ID}
      AWS_SECRET_ACCESS_KEY: ${AWS_SECRET_ACCESS_KEY}
    ports:
      - "8000:8000"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - rockbees
    restart: always
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 2G
        reservations:
          cpus: '1'
          memory: 1G

  celery_worker:
    build:
      context: ./backend
      dockerfile: Dockerfile
    command: celery -A backend.tasks worker --loglevel=info
    environment:
      DATABASE_URL: postgresql://${DB_USER}:${DB_PASSWORD}@postgres:5432/${DB_NAME}
      REDIS_URL: redis://:${REDIS_PASSWORD}@redis:6379
      CELERY_BROKER_URL: redis://:${REDIS_PASSWORD}@redis:6379/1
    depends_on:
      - postgres
      - redis
    networks:
      - rockbees
    restart: always
    deploy:
      resources:
        limits:
          cpus: '4'
          memory: 4G

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./config/nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./config/nginx/ssl:/etc/nginx/ssl:ro
      - ./static:/usr/share/nginx/html:ro
    depends_on:
      - api
    networks:
      - rockbees
    restart: always

volumes:
  postgres_data:
  redis_data:

networks:
  rockbees:
    driver: bridge
```

### Build and Push Docker Images

```bash
# Build images
docker build -t rockbees-api:latest ./backend
docker build -t rockbees-frontend:latest ./frontend/web

# Tag for registry
docker tag rockbees-api:latest your-registry/rockbees-api:latest
docker tag rockbees-frontend:latest your-registry/rockbees-frontend:latest

# Push to registry (Docker Hub, ECR, etc.)
docker push your-registry/rockbees-api:latest
docker push your-registry/rockbees-frontend:latest
```

---

## Kubernetes Deployment

### Kubernetes Manifests

#### Namespace
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: rockbees
```

#### API Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rockbees-api
  namespace: rockbees
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: rockbees-api
  template:
    metadata:
      labels:
        app: rockbees-api
    spec:
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
                        - rockbees-api
                topologyKey: kubernetes.io/hostname
      containers:
        - name: api
          image: your-registry/rockbees-api:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 8000
              name: http
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: rockbees-secrets
                  key: database-url
            - name: REDIS_URL
              valueFrom:
                secretKeyRef:
                  name: rockbees-secrets
                  key: redis-url
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: rockbees-secrets
                  key: jwt-secret
          livenessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
          readinessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 10
            periodSeconds: 5
            timeoutSeconds: 3
          resources:
            requests:
              cpu: 500m
              memory: 512Mi
            limits:
              cpu: 2
              memory: 2Gi
---
apiVersion: v1
kind: Service
metadata:
  name: rockbees-api-service
  namespace: rockbees
spec:
  type: ClusterIP
  selector:
    app: rockbees-api
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000
      name: http
```

#### PostgreSQL StatefulSet
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: rockbees
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
          image: postgis/postgis:15-3.3
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: rockbees-secrets
                  key: db-user
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: rockbees-secrets
                  key: db-password
            - name: POSTGRES_DB
              value: rockbees_db
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
            storage: 50Gi
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: rockbees
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - protocol: TCP
      port: 5432
      targetPort: 5432
```

#### Ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rockbees-ingress
  namespace: rockbees
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts:
        - api.rockbees.app
      secretName: rockbees-tls
  rules:
    - host: api.rockbees.app
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: rockbees-api-service
                port:
                  number: 80
```

### Deploy to Kubernetes

```bash
# Create namespace
kubectl create namespace rockbees

# Create secrets
kubectl create secret generic rockbees-secrets \
  --from-literal=database-url=postgresql://user:pass@postgres:5432/rockbees_db \
  --from-literal=redis-url=redis://redis:6379 \
  --from-literal=jwt-secret=your-secret-key \
  -n rockbees

# Apply manifests
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/postgres-statefulset.yaml
kubectl apply -f k8s/redis-deployment.yaml
kubectl apply -f k8s/api-deployment.yaml
kubectl apply -f k8s/ingress.yaml

# Check status
kubectl get pods -n rockbees
kubectl logs -f deployment/rockbees-api -n rockbees
```

---

## Cloud Deployment

### AWS ECS (Elastic Container Service)

```bash
# Create ECS cluster
aws ecs create-cluster --cluster-name rockbees-cluster

# Register task definition
aws ecs register-task-definition --cli-input-json file://task-definition.json

# Create service
aws ecs create-service \
  --cluster rockbees-cluster \
  --service-name rockbees-api \
  --task-definition rockbees-api:1 \
  --desired-count 3 \
  --load-balancers targetGroupArn=arn:aws:elasticloadbalancing:...,containerName=api,containerPort=8000
```

### Google Cloud Run

```bash
# Build image
gcloud builds submit --tag gcr.io/PROJECT-ID/rockbees-api

# Deploy
gcloud run deploy rockbees-api \
  --image gcr.io/PROJECT-ID/rockbees-api:latest \
  --platform managed \
  --region us-central1 \
  --set-env-vars DATABASE_URL=$DATABASE_URL,REDIS_URL=$REDIS_URL \
  --memory 2Gi \
  --cpu 2
```

### AWS RDS (Database)

```bash
# Create RDS instance
aws rds create-db-instance \
  --db-instance-identifier rockbees-db \
  --db-instance-class db.t3.medium \
  --engine postgres \
  --master-username admin \
  --master-user-password <password> \
  --allocated-storage 100
```

---

## CI/CD Pipeline

### GitHub Actions Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy RockBees

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgis/postgis:15-3.3
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.9'
      
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r backend/requirements.txt
          pip install pytest pytest-cov
      
      - name: Run tests
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/test
          REDIS_URL: redis://localhost:6379
        run: |
          pytest backend/tests/ -v --cov=backend --cov-report=xml
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3

  build:
    needs: test
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      
      - name: Log in to Container Registry
        uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          context: ./backend
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/api:latest

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    
    steps:
      - name: Deploy to Kubernetes
        env:
          KUBE_CONFIG: ${{ secrets.KUBE_CONFIG }}
        run: |
          mkdir ~/.kube
          echo $KUBE_CONFIG | base64 -d > ~/.kube/config
          kubectl rollout restart deployment/rockbees-api -n rockbees
```

---

## Production Checklist

- [ ] All tests passing (100% coverage minimum 80%)
- [ ] Security audit completed
- [ ] Database backups configured
- [ ] SSL/TLS certificates installed
- [ ] Environment variables set securely
- [ ] Monitoring and alerting configured
- [ ] Log aggregation setup
- [ ] Auto-scaling policies defined
- [ ] Disaster recovery plan tested
- [ ] Performance benchmarks verified
- [ ] Documentation updated
- [ ] Team trained on deployment
- [ ] Rollback procedure tested
- [ ] Data retention policies defined
- [ ] Compliance checks completed

---

## Monitoring & Logging

### Prometheus Metrics

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'rockbees-api'
    static_configs:
      - targets: ['localhost:8000']
    metrics_path: '/metrics'
```

### ELK Stack (Elasticsearch, Logstash, Kibana)

```yaml
# logstash.conf
input {
  http {
    port => 5000
  }
}

filter {
  json {
    source => "message"
  }
}

output {
  elasticsearch {
    hosts => ["localhost:9200"]
    index => "rockbees-%{+YYYY.MM.dd}"
  }
}
```

### Application Logging

```python
# backend/config/logging.py
import logging
import logging.handlers

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.handlers.RotatingFileHandler(
            'logs/api.log',
            maxBytes=10485760,  # 10MB
            backupCount=10
        ),
        logging.StreamHandler()
    ]
)
```

---

## Scaling & Auto-scaling

### Kubernetes HPA (Horizontal Pod Autoscaler)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: rockbees-api-hpa
  namespace: rockbees
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: rockbees-api
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
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 50
          periodSeconds: 15
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15
        - type: Pods
          value: 2
          periodSeconds: 60
```

---

**Last Updated**: December 2025  
**Version**: 1.0.0  
**Status**: Active