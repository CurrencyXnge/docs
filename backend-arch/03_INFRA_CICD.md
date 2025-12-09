<!-- Author: Ratheesh G Kumar, Software Engineer, Team CurrencyXnge_fintech SaaS Product -->
# 🚀 CI/CD, Docker & Kubernetes Strategy

This document outlines the DevOps pipeline, containerization strategy, and orchestration for CurrencyEx.

## 1. Containerization (Multi-Stage Docker)

We use **Multi-Stage Builds** to keep production images tiny (typically < 20MB) and secure (no shell access).

**Dockerfile Template (`cmd/api/Dockerfile`)**:
```dockerfile
# --- Stage 1: Builder ---
FROM golang:1.21-alpine AS builder

# Install build dependencies
RUN apk add --no-cache git make

WORKDIR /app

# Pre-cache deps definition
COPY go.mod go.sum ./
RUN go mod download

# Build the binary
COPY . .
# CGO_ENABLED=0 creates a statically linked binary (no external C deps)
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-w -s" -o /go-bin/app ./cmd/api

# --- Stage 2: Runner ---
# Use Distroless 'static' for maximum security (contains only CA roots and tzdata)
FROM gcr.io/distroless/static-debian11

COPY --from=builder /go-bin/app /
COPY --from=builder /app/config /config  

# Use non-root user (Standard in Distroless)
USER nonroot:nonroot

EXPOSE 8080
CMD ["/app"]
```

---

## 2. Kubernetes (K8s) Cluster Design

We deploy to a Managed K8s Service (EKS / GKE / DigitalOcean K8s).

### Namespaces
- `prod`: Production workloads
- `staging`: Pre-prod testing
- `dev`: Feature branch deployments
- `monitoring`: Prometheus/Grafana stack

### Deployment Manifest (`deployments/k8s/deployment.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
  namespace: prod
spec:
  replicas: 3 # High Availability
  selector:
    matchLabels:
      app: api-gateway
  template:
    metadata:
      labels:
        app: api-gateway
    spec:
      containers:
        - name: app
          image: registry.gitlab.com/currencyex/api:v1.2.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m" # Throttling cap
              memory: "512Mi" # OOM Kill cap
          livenessProbe:
             httpGet:
               path: /healthz
               port: 8080
             initialDelaySeconds: 10
             periodSeconds: 15
          envFrom:
            - configMapRef:
                name: app-config
            - secretRef:
                name: db-credentials
```

---

## 3. CI/CD Pipeline (GitLab CI / GitHub Actions)

We practice **GitOps**. The pipeline automates testing, building, and deployment.

### Pipeline Stages
1.  **Test**: Runs Unit Tests and Linters on every Pull Request.
2.  **Build**: Builds Docker image and checks for vulnerabilities (Trivy).
3.  **Deploy Staging**: Auto-deploys `main` branch to Staging env.
4.  **Integration**: Runs E2E tests against Staging.
5.  **Release**: Creates a release tag (SemVer `v1.2.0`).
6.  **Deploy Prod**: Manual Trigger to deploy tag to Production.

**Example `.gitlab-ci.yml`**:
```yaml
stages:
  - test
  - build
  - deploy

unit_test:
  stage: test
  image: golang:1.21
  script:
    - go test -v -cover ./...

build_image:
  stage: build
  script:
    - docker build -t $REGISTRY_URL:$CI_COMMIT_SHA .
    - docker push $REGISTRY_URL:$CI_COMMIT_SHA

deploy_prod:
  stage: deploy
  when: manual
  script:
    - helm upgrade --install api-gateway ./charts/api --set image.tag=$CI_COMMIT_SHA -n prod
```

---

## 4. Environment Configuration

We strictly separate config from code using **The Twelve-Factor App** methodology.

| Env Var | Description | Prod Example |
| :--- | :--- | :--- |
| `APP_ENV` | Environment context | `production` |
| `DB_HOST` | Database endpoint | `db-cluster-rw.prod.svc.cluster.local` |
| `NATS_URL` | Message Bus | `nats://nats-cluster:4222` |
| `JWT_SECRET` | Signing Key | (K8s Secret) |

---

## 5. Observability (Metrics & Logs)

- **Logs**: JSON Structured Logs (Zap). Collected by **Fluentd** -> **OpenSearch**.
- **Metrics**: Prometheus scrapes `/metrics` endpoint (Go Collector). 
- **Tracing**: OpenTelemetry (Jaeger) for distributed traces gRPC/HTTP.
