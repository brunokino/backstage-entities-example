# Deployment

## Overview

The User Service is deployed as a containerized application on Kubernetes, following cloud-native best practices for scalability, reliability, and zero-downtime deployments.

## Deployment Architecture

```mermaid
graph TB
    subgraph "Load Balancer"
        LB[Nginx Ingress / ALB]
    end
    
    subgraph "Kubernetes Cluster"
        subgraph "user-service Namespace"
            Pod1[Pod 1]
            Pod2[Pod 2]
            Pod3[Pod 3]
        end
        
        subgraph "Persistent Services"
            PG[(PostgreSQL)]
            Redis[(Redis Cluster)]
        end
    end
    
    subgraph "External Services"
        Email[Email Service]
        Audit[Audit Service]
    end
    
    LB --> Pod1
    LB --> Pod2
    LB --> Pod3
    
    Pod1 --> PG
    Pod2 --> PG
    Pod3 --> PG
    
    Pod1 --> Redis
    Pod2 --> Redis
    Pod3 --> Redis
    
    Pod1 --> Email
    Pod2 --> Audit
    Pod3 --> Email
    
    style Pod1 fill:#4A90E2
    style Pod2 fill:#4A90E2
    style Pod3 fill:#4A90E2
```

## Environments

| Environment | Purpose | URL | Auto-Deploy |
|-------------|---------|-----|-------------|
| **Development** | Local development | http://localhost:3000 | No |
| **Staging** | Pre-production testing | https://user-service-staging.kinoshita-labs.com | Yes (from `develop` branch) |
| **Production** | Production traffic | https://user-service.kinoshita-labs.com | Yes (from `main` branch, after approval) |

## Container Image

### Dockerfile

```dockerfile
# Multi-stage build for optimized image size

# Stage 1: Build
FROM node:20-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./
COPY prisma ./prisma/

# Install dependencies
RUN npm ci --only=production && \
    npm cache clean --force

# Copy source code
COPY . .

# Generate Prisma client
RUN npx prisma generate

# Build TypeScript
RUN npm run build

# Stage 2: Production
FROM node:20-alpine

WORKDIR /app

# Install dumb-init for proper signal handling
RUN apk add --no-cache dumb-init

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

# Copy built application
COPY --from=builder --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nodejs:nodejs /app/dist ./dist
COPY --from=builder --chown=nodejs:nodejs /app/prisma ./prisma
COPY --chown=nodejs:nodejs package*.json ./

# Switch to non-root user
USER nodejs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000/api/v1/health', (r) => process.exit(r.statusCode === 200 ? 0 : 1))"

# Start application
ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "dist/server.js"]
```

### Building the Image

```bash
# Build image
docker build -t user-service:1.0.0 .

# Tag for registry
docker tag user-service:1.0.0 registry.kinoshita-labs.com/user-service:1.0.0

# Push to registry
docker push registry.kinoshita-labs.com/user-service:1.0.0
```

## Kubernetes Deployment

### Namespace

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: user-service
  labels:
    name: user-service
    environment: production
```

### ConfigMap

```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: user-service-config
  namespace: user-service
data:
  NODE_ENV: "production"
  API_VERSION: "v1"
  PORT: "3000"
  LOG_LEVEL: "info"
  LOG_FORMAT: "json"
  RATE_LIMIT_WINDOW_MS: "900000"
  RATE_LIMIT_MAX_ANONYMOUS: "100"
  RATE_LIMIT_MAX_AUTHENTICATED: "1000"
  CORS_ORIGIN: "https://app.kinoshita-labs.com"
```

### Secret

```yaml
# k8s/secret.yaml (encrypted in production)
apiVersion: v1
kind: Secret
metadata:
  name: user-service-secrets
  namespace: user-service
type: Opaque
stringData:
  DATABASE_URL: "postgresql://user:password@postgres:5432/user_service_prod"
  REDIS_HOST: "redis-cluster"
  REDIS_PORT: "6379"
  REDIS_PASSWORD: "redis-password"
  JWT_SECRET: "super-secret-jwt-key"
  EMAIL_SERVICE_URL: "http://email-service:3001/api/v1"
  AUDIT_SERVICE_URL: "http://audit-service:3002/api/v1"
```

### Deployment

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: user-service
  labels:
    app: user-service
    version: "1.0.0"
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
        version: "1.0.0"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "3000"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: user-service
      containers:
      - name: user-service
        image: registry.kinoshita-labs.com/user-service:1.0.0
        imagePullPolicy: Always
        ports:
        - name: http
          containerPort: 3000
          protocol: TCP
        env:
        - name: NODE_ENV
          valueFrom:
            configMapKeyRef:
              name: user-service-config
              key: NODE_ENV
        - name: PORT
          valueFrom:
            configMapKeyRef:
              name: user-service-config
              key: PORT
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: user-service-secrets
              key: DATABASE_URL
        - name: REDIS_HOST
          valueFrom:
            secretKeyRef:
              name: user-service-secrets
              key: REDIS_HOST
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: user-service-secrets
              key: JWT_SECRET
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /api/v1/health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /api/v1/health
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]
      terminationGracePeriodSeconds: 30
```

### Service

```yaml
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: user-service
  namespace: user-service
  labels:
    app: user-service
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 3000
    protocol: TCP
    name: http
  selector:
    app: user-service
```

### Ingress

```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: user-service
  namespace: user-service
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - user-service.kinoshita-labs.com
    secretName: user-service-tls
  rules:
  - host: user-service.kinoshita-labs.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 80
```

### HorizontalPodAutoscaler

```yaml
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: user-service
  namespace: user-service
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
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
```

## Database Migrations

### Pre-Deployment Migration Job

```yaml
# k8s/migration-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: user-service-migration-{{ .Values.version }}
  namespace: user-service
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: migration
        image: registry.kinoshita-labs.com/user-service:{{ .Values.version }}
        command: ["npm", "run", "migrate:deploy"]
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: user-service-secrets
              key: DATABASE_URL
  backoffLimit: 3
```

### Migration Strategy

```mermaid
sequenceDiagram
    participant CI/CD
    participant K8s
    participant Migration
    participant Database
    participant OldPods
    participant NewPods
    
    CI/CD->>K8s: Create migration job
    K8s->>Migration: Start migration pod
    Migration->>Database: Apply migrations
    Database-->>Migration: Success
    Migration-->>K8s: Job complete
    
    K8s->>NewPods: Start rolling update
    K8s->>NewPods: Deploy new pods
    NewPods->>Database: Connect
    
    K8s->>OldPods: Gradually terminate
    OldPods->>OldPods: Drain connections
    OldPods-->>K8s: Terminated
```

## CI/CD Pipeline

### GitHub Actions Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches:
      - main
      - develop

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Login to Container Registry
        uses: docker/login-action@v2
        with:
          registry: registry.kinoshita-labs.com
          username: ${{ secrets.REGISTRY_USERNAME }}
          password: ${{ secrets.REGISTRY_PASSWORD }}

      - name: Extract version
        id: version
        run: echo "VERSION=$(cat package.json | jq -r .version)" >> $GITHUB_OUTPUT

      - name: Build and push Docker image
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: |
            registry.kinoshita-labs.com/user-service:${{ steps.version.outputs.VERSION }}
            registry.kinoshita-labs.com/user-service:latest
          cache-from: type=registry,ref=registry.kinoshita-labs.com/user-service:buildcache
          cache-to: type=registry,ref=registry.kinoshita-labs.com/user-service:buildcache,mode=max

      - name: Set up kubectl
        uses: azure/setup-kubectl@v3

      - name: Configure kubeconfig
        run: |
          echo "${{ secrets.KUBECONFIG }}" | base64 -d > kubeconfig
          export KUBECONFIG=./kubeconfig

      - name: Run database migrations
        run: |
          kubectl create job user-service-migration-${{ steps.version.outputs.VERSION }} \
            --from=cronjob/user-service-migration \
            --namespace=user-service
          kubectl wait --for=condition=complete --timeout=300s \
            job/user-service-migration-${{ steps.version.outputs.VERSION }} \
            --namespace=user-service

      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/user-service \
            user-service=registry.kinoshita-labs.com/user-service:${{ steps.version.outputs.VERSION }} \
            --namespace=user-service
          kubectl rollout status deployment/user-service \
            --namespace=user-service \
            --timeout=5m

      - name: Verify deployment
        run: |
          kubectl get pods -n user-service
          kubectl get svc -n user-service
```

## Deployment Strategies

### Rolling Update (Default)

- **Zero downtime**: New pods start before old pods terminate
- **Gradual rollout**: MaxSurge: 1, MaxUnavailable: 0
- **Health checks**: Only route traffic to healthy pods

### Blue-Green Deployment

```bash
# Deploy green (new version)
kubectl apply -f k8s/deployment-green.yaml

# Wait for green to be ready
kubectl wait --for=condition=available deployment/user-service-green

# Switch traffic
kubectl patch service user-service -p '{"spec":{"selector":{"version":"green"}}}'

# Cleanup blue (old version)
kubectl delete deployment user-service-blue
```

### Canary Deployment

```yaml
# Deploy canary with 10% traffic
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: user-service
spec:
  hosts:
  - user-service
  http:
  - match:
    - headers:
        canary:
          exact: "true"
    route:
    - destination:
        host: user-service
        subset: canary
  - route:
    - destination:
        host: user-service
        subset: stable
      weight: 90
    - destination:
        host: user-service
        subset: canary
      weight: 10
```

## Rollback Procedures

### Automatic Rollback

Kubernetes automatically rolls back if readiness probes fail.

### Manual Rollback

```bash
# View deployment history
kubectl rollout history deployment/user-service -n user-service

# Rollback to previous version
kubectl rollout undo deployment/user-service -n user-service

# Rollback to specific revision
kubectl rollout undo deployment/user-service -n user-service --to-revision=3

# Check rollback status
kubectl rollout status deployment/user-service -n user-service
```

## Monitoring Deployment

### Check Deployment Status

```bash
# Watch deployment
kubectl rollout status deployment/user-service -n user-service

# Check pods
kubectl get pods -n user-service -w

# Check events
kubectl get events -n user-service --sort-by='.lastTimestamp'

# View logs
kubectl logs -f deployment/user-service -n user-service
```

### Post-Deployment Verification

```bash
# Health check
curl https://user-service.kinoshita-labs.com/api/v1/health

# Smoke tests
npm run test:smoke

# Check metrics
curl https://user-service.kinoshita-labs.com/metrics
```

## Best Practices

### 1. Always Run Migrations First

Run database migrations before deploying new code to avoid compatibility issues.

### 2. Use Health Checks

Configure liveness and readiness probes properly:
- **Liveness**: Restart unhealthy pods
- **Readiness**: Route traffic only to ready pods

### 3. Set Resource Limits

Always define requests and limits to prevent resource exhaustion.

### 4. Use Rolling Updates

Configure `maxSurge` and `maxUnavailable` for zero-downtime deployments.

### 5. Implement Graceful Shutdown

Handle SIGTERM properly:

```typescript
process.on('SIGTERM', async () => {
  console.log('SIGTERM received, shutting down gracefully...');
  
  // Stop accepting new requests
  server.close(() => {
    console.log('HTTP server closed');
  });
  
  // Close database connections
  await prisma.$disconnect();
  
  // Close Redis connections
  await redis.quit();
  
  process.exit(0);
});
```

### 6. Version Everything

- Tag Docker images with version numbers
- Use semantic versioning
- Keep deployment manifests in version control

### 7. Monitor Deployments

- Watch metrics during and after deployment
- Set up alerts for deployment failures
- Keep deployment logs for audit

---

**Last Updated**: 2025-01-07  
**Document Owner**: Platform Team

