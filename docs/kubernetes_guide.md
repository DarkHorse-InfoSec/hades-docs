# Kubernetes Deployment Guide

> **ROADMAP, not a shipped deployment option (updated 2026-09-27).** HADES currently ships as a single self-contained binary (the `hades` CLI and `hades-server`). Containerized and distributed deployments (Docker Compose, Kubernetes, Helm, Redis-backed worker queues) are on the Enterprise roadmap. This guide describes that planned architecture. **No throughput, scans-per-day or hardware-sizing figure in it is a measured HADES result**, and none is given; size a deployment by benchmarking on your own hardware and file mix.

This guide describes the planned Helm chart and Kustomize overlays for Kubernetes deployments.

**Related guides:**
- [Observability Guide](observability_guide.md) -- Prometheus metrics, Grafana dashboards, alerting
- [Performance Guide](performance_guide.md) -- Tuning, profiling, benchmarking, deployment patterns
- [Enterprise Deployment Guide](enterprise_deployment_guide.md) -- RBAC, SSO, PostgreSQL, Redis, encryption

## Prerequisites

- Kubernetes 1.24+
- Helm 3.12+ (for Helm chart)
- kubectl (for Kustomize overlays)
- Container registry with the HADES image (build with `docker/hades-full.dockerfile`)

## Helm Chart

### Installation

```bash
# Install with default values
helm install hades kubernetes/helm/hades

# Install with custom values
helm install hades kubernetes/helm/hades \
  --set api.apiKey=your-secret-key \
  --set redis.enabled=true \
  --set api.replicas=3

# Install with a values file
helm install hades kubernetes/helm/hades -f my-values.yaml
```

### Values Reference

#### Image

| Value | Default | Description |
|-------|---------|-------------|
| `image.repository` | `hades-scanner` | Container image name |
| `image.tag` | `0.7.1` | Image tag |
| `image.pullPolicy` | `IfNotPresent` | Pull policy |

#### API Server

| Value | Default | Description |
|-------|---------|-------------|
| `api.replicas` | `2` | API server replica count |
| `api.port` | `8666` | API listen port |
| `api.apiKey` | `""` | API authentication key |
| `api.workers` | `4` | Uvicorn workers per pod |
| `api.resources.requests.cpu` | `500m` | CPU request |
| `api.resources.requests.memory` | `512Mi` | Memory request |
| `api.resources.limits.cpu` | `2000m` | CPU limit |
| `api.resources.limits.memory` | `2Gi` | Memory limit |

#### Workers

| Value | Default | Description |
|-------|---------|-------------|
| `worker.replicas` | `2` | Worker replica count |
| `worker.resources.requests.cpu` | `500m` | CPU request |
| `worker.resources.requests.memory` | `512Mi` | Memory request |
| `worker.resources.limits.cpu` | `2000m` | CPU limit |
| `worker.resources.limits.memory` | `2Gi` | Memory limit |

#### Database

| Value | Default | Description |
|-------|---------|-------------|
| `database.backend` | `sqlite` | Database backend (`sqlite` or `postgresql`) |
| `database.dsn` | `""` | PostgreSQL connection string |

#### Redis

| Value | Default | Description |
|-------|---------|-------------|
| `redis.enabled` | `false` | Enable Redis for caching and queues |
| `redis.url` | `redis://redis:6379/0` | Redis connection URL |

#### Autoscaling (HPA)

| Value | Default | Description |
|-------|---------|-------------|
| `autoscaling.api.enabled` | `true` | Enable API HPA |
| `autoscaling.api.minReplicas` | `2` | Minimum API replicas |
| `autoscaling.api.maxReplicas` | `10` | Maximum API replicas |
| `autoscaling.api.targetCPU` | `70` | CPU utilization target |
| `autoscaling.worker.enabled` | `true` | Enable worker HPA |
| `autoscaling.worker.minReplicas` | `2` | Minimum worker replicas |
| `autoscaling.worker.maxReplicas` | `20` | Maximum worker replicas |

#### Ingress

| Value | Default | Description |
|-------|---------|-------------|
| `ingress.enabled` | `false` | Enable Ingress |
| `ingress.className` | `nginx` | Ingress class |
| `ingress.host` | `hades.example.com` | Hostname |
| `ingress.tls` | `false` | Enable TLS |
| `ingress.tlsSecretName` | `hades-tls` | TLS secret name |

#### Monitoring

| Value | Default | Description |
|-------|---------|-------------|
| `monitoring.serviceMonitor.enabled` | `true` | Create ServiceMonitor for Prometheus |
| `monitoring.serviceMonitor.interval` | `10s` | Scrape interval |
| `monitoring.grafanaDashboard.enabled` | `true` | Deploy dashboard ConfigMap |

### Secrets Management

Create a Kubernetes secret for sensitive values:

```bash
kubectl create secret generic hades-secrets \
  --from-literal=api-key=your-production-key \
  --from-literal=db-dsn=postgresql://user:pass@host/hades \
  --from-literal=encryption-key=your-encryption-key
```

Reference in values:

```yaml
api:
  apiKey: ""  # Set via HADES_API_KEY env from secret
  extraEnv:
    - name: HADES_API_KEY
      valueFrom:
        secretKeyRef:
          name: hades-secrets
          key: api-key
```

### Upgrading

```bash
# Upgrade with new values
helm upgrade hades kubernetes/helm/hades -f my-values.yaml

# Rollback
helm rollback hades 1
```

### Uninstalling

```bash
helm uninstall hades
```

## Kustomize

### Directory Structure

```
kubernetes/kustomize/
  base/            # Base manifests
  overlays/
    dev/           # Single replica, SQLite
    production/    # Multi-replica, PostgreSQL, Redis, HPA, TLS
```

### Development

```bash
kubectl apply -k kubernetes/kustomize/overlays/dev
```

### Production

```bash
kubectl apply -k kubernetes/kustomize/overlays/production
```

### Customization

Create a new overlay directory referencing the base:

```yaml
# kubernetes/kustomize/overlays/staging/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
patches:
  - patch: |-
      - op: replace
        path: /spec/replicas
        value: 2
    target:
      kind: Deployment
      name: hades-api
```

## Scaling

### Horizontal Pod Autoscaler

The Helm chart includes HPA definitions for both API and worker deployments:

- **API pods**: Scale on CPU utilization (default target: 70%)
- **Worker pods**: Scale on CPU utilization and custom queue-depth metrics

### Manual Scaling

```bash
# Scale API replicas
kubectl scale deployment hades-api --replicas=5

# Scale workers
kubectl scale deployment hades-worker --replicas=10
```

## Monitoring with ServiceMonitor

The Helm chart creates a `ServiceMonitor` resource for Prometheus Operator:

```yaml
monitoring:
  serviceMonitor:
    enabled: true
    interval: 10s
```

This automatically configures Prometheus to scrape the `/metrics` endpoint on all HADES API pods.

## Namespace and Security

### Namespace Isolation

Deploy HADES in a dedicated namespace for resource isolation and RBAC scoping:

```bash
kubectl create namespace hades
helm install hades kubernetes/helm/hades -n hades
```

### Kubernetes RBAC

Create a minimal service account with only the permissions HADES needs:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: hades-sa
  namespace: hades
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: hades-role
  namespace: hades
rules:
  - apiGroups: [""]
    resources: ["configmaps", "secrets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list"]
```

### Network Policies

Restrict traffic to only what HADES requires:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: hades-api-policy
  namespace: hades
spec:
  podSelector:
    matchLabels:
      app: hades-api
  ingress:
    - ports:
        - port: 8666
  egress:
    - ports:
        - port: 5432   # PostgreSQL
        - port: 6379   # Redis
```

---

## Resource Tuning

### Sizing

HADES publishes no measured per-pod throughput, so there is no sizing table here. Start from the chart defaults above, benchmark with your own file mix, and scale from the measured result.

### Tuning Tips

- **API pods** are I/O-bound (network requests, database queries). Scale by replica count rather than per-pod resources.
- **Worker pods** are CPU-bound (YARA matching, ML inference, metadata extraction). Increase CPU limits for faster per-file processing.
- **Memory** requirements depend on ML model size and cache configuration. Monitor with `kubectl top pods` and adjust limits.
- Use the pipeline profiler (`--profile`) to identify whether your bottleneck is metadata extraction, YARA, heuristic analysis, or ML -- then allocate resources accordingly.

For detailed performance tuning, see the **[Performance Guide](performance_guide.md)**.

---

## Rolling Updates

### Zero-Downtime Upgrade Strategy

The Helm chart supports rolling updates by default. Kubernetes replaces pods one at a time, ensuring the service remains available during upgrades.

```bash
# Update the image tag
helm upgrade hades kubernetes/helm/hades \
  --set image.tag=0.7.2

# Monitor the rollout
kubectl rollout status deployment/hades-api -n hades
kubectl rollout status deployment/hades-worker -n hades
```

### Rollback

```bash
# Rollback to previous revision
helm rollback hades 1

# Or rollback a specific deployment
kubectl rollout undo deployment/hades-api -n hades
```

### Pre-Upgrade Checklist

1. Back up PostgreSQL database if using external storage
2. Verify the new image is available in your container registry
3. Review release notes for any configuration changes
4. Test the upgrade in a dev/staging namespace first

---

## Docker Compose Alternative

For teams not yet on Kubernetes, Docker Compose provides similar scaling capabilities with less operational complexity:

```bash
# Scalable deployment (PostgreSQL + Redis + workers)
docker compose -f docker/docker-compose.scale.yml up -d --scale hades-worker=4

# High-availability deployment (nginx + multi-instance API)
docker compose -f docker/docker-compose.ha.yml up -d --scale hades-api=4

# Full stack with observability (PostgreSQL + Redis + API + workers + Prometheus + Grafana)
docker compose -f docker/docker-compose.full-stack.yml up -d
```

Docker Compose is a good stepping stone before migrating to Kubernetes, as the same container images and environment variables are used in both environments.

---

## Troubleshooting

### Pods stuck in CrashLoopBackOff

Check logs:

```bash
kubectl logs deployment/hades-api --tail=50
```

Common causes:
- Missing or incorrect database connection string
- Redis connection refused (if `redis.enabled: true` but no Redis)
- Invalid API key format

### Health check failures

HADES provides a `/api/v1/health` endpoint used by Kubernetes liveness and readiness probes. Run a manual check:

```bash
kubectl exec deployment/hades-api -- curl -f http://localhost:8666/api/v1/health
```

### Workers not processing tasks

Verify Redis connectivity:

```bash
kubectl exec deployment/hades-worker -- python -c "import redis; r = redis.from_url('redis://redis:6379/0'); print(r.ping())"
```

Check worker logs for connection errors or task failures.

### HPA not scaling

Verify metrics-server is installed:

```bash
kubectl top pods
```

Check HPA status:

```bash
kubectl get hpa
kubectl describe hpa hades-api-hpa
```
