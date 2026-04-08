# HADES Scaling Guide

This guide covers horizontal scaling, database scaling, load balancing, and Kubernetes deployment concepts for HADES.

## Docker Compose Scaling

### Adding Workers

Scale background workers to increase scan throughput:

```bash
# Start with 4 workers
docker compose -f docker/docker-compose.scale.yml up -d --scale hades-worker=4

# Scale up to 8 workers without downtime
docker compose -f docker/docker-compose.scale.yml up -d --scale hades-worker=8
```

Set the default replica count via environment variable:

```bash
export HADES_WORKER_REPLICAS=6
docker compose -f docker/docker-compose.scale.yml up -d
```

### Adding API Instances

For the HA deployment, scale API server instances behind the nginx load balancer:

```bash
docker compose -f docker/docker-compose.ha.yml up -d --scale hades-api=4
```

Nginx automatically load-balances across all API instances using least-connections.

### Deployment Files

| File | Use Case | Description |
|------|----------|-------------|
| `docker-compose.yml` | Development | Single API + viewer, optional PostgreSQL/Redis |
| `docker-compose.scale.yml` | Production | PostgreSQL + Redis + API + scalable workers |
| `docker-compose.ha.yml` | High availability | Nginx LB + PgBouncer + Redis Sentinel + multi-API + workers |

## Redis Configuration

### Distributed State

Redis enables state sharing across all HADES instances:

- **Scan cache:** Shared LRU cache avoids duplicate scans across API instances
- **Task queue:** Workers pull scan tasks from a Redis list (`hades:tasks`)
- **Rate limiting:** Sliding window counters shared across API servers
- **Session state:** API key sessions and distributed locks

### Redis Connection

```bash
# Environment variable
export HADES_REDIS_URL=redis://redis-host:6379/0

# With authentication
export HADES_REDIS_URL=redis://:password@redis-host:6379/0
```

### Redis Memory Sizing

| Deployment | maxmemory | Estimated Capacity |
|-----------|-----------|-------------------|
| Small (< 100K scans/day) | 256 MB | ~50K cached results |
| Medium (100K-500K/day) | 512 MB | ~100K cached results |
| Large (500K-2M/day) | 1 GB | ~200K cached results |
| Enterprise (2M+/day) | 2-4 GB | ~500K+ cached results |

### Redis Sentinel (High Availability)

The `docker-compose.ha.yml` includes Redis Sentinel for automatic failover. For custom Sentinel configuration:

```
sentinel monitor hades-master redis-primary 6379 2
sentinel down-after-milliseconds hades-master 5000
sentinel failover-timeout hades-master 10000
sentinel parallel-syncs hades-master 1
```

## PostgreSQL Scaling

### Connection Pooling with PgBouncer

PgBouncer sits between HADES and PostgreSQL, pooling database connections:

```
# PgBouncer settings (in docker-compose.ha.yml)
POOL_MODE=transaction         # Recommended for HADES
MAX_CLIENT_CONN=200           # Max connections from HADES
DEFAULT_POOL_SIZE=25          # Connections to PostgreSQL
```

Benefits:
- Reduces PostgreSQL connection overhead
- Handles connection spikes gracefully
- Required when running multiple API instances

### Read Replicas

For read-heavy workloads (many scan result retrievals vs. new scans):

1. Configure a PostgreSQL streaming replica
2. Direct read queries to the replica via a separate DSN
3. Keep write queries (new scans, audit log) on the primary

### Table Partitioning

For databases exceeding 100M scan records, partition the scans table by date:

```sql
-- Example: monthly partitions
CREATE TABLE scans (
    id SERIAL,
    created_at TIMESTAMP NOT NULL,
    tenant_id TEXT,
    results_json TEXT
) PARTITION BY RANGE (created_at);

CREATE TABLE scans_2026_01 PARTITION OF scans
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
```

### Backup and Recovery

```bash
# Automated backup
pg_dump -h localhost -U hades -Fc hades > hades_backup_$(date +%Y%m%d).dump

# Restore
pg_restore -h localhost -U hades -d hades hades_backup.dump
```

## Load Balancing

### Nginx Configuration

The included `docker/nginx.conf` provides:

- **Least-connections** upstream balancing
- **WebSocket upgrade** support for `/ws` endpoint
- **Rate limiting:** 100 req/min general, 30 req/min for scan endpoints
- **Health check routing:** `/api/v1/health` bypasses rate limiting
- **Static caching:** Dashboard assets cached for 10 minutes
- **Upload limits:** 100 MB max file size

### Custom Nginx Configuration

Mount a custom `nginx.conf`:

```yaml
# In docker-compose override
services:
  nginx:
    volumes:
      - ./my-nginx.conf:/etc/nginx/nginx.conf:ro
```

### SSL/TLS Termination

Add SSL certificates to the nginx configuration:

```nginx
server {
    listen 443 ssl;
    ssl_certificate     /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;
    # ... proxy_pass directives unchanged
}
```

### Health Checks

Nginx health checks ensure traffic only routes to healthy instances:

```bash
# Check health through the load balancer
curl http://localhost/api/v1/health

# Direct instance check (bypass LB)
curl http://hades-api-1:8666/api/v1/health
```

## Kubernetes Concepts

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hades-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hades-api
  template:
    metadata:
      labels:
        app: hades-api
    spec:
      containers:
        - name: hades-api
          image: hades-scanner:0.7.1
          ports:
            - containerPort: 8666
          env:
            - name: HADES_DB_BACKEND
              value: postgresql
            - name: HADES_DB_DSN
              valueFrom:
                secretKeyRef:
                  name: hades-secrets
                  key: db-dsn
            - name: HADES_REDIS_URL
              value: redis://redis-svc:6379/0
          resources:
            requests:
              memory: "1Gi"
              cpu: "1"
            limits:
              memory: "4Gi"
              cpu: "4"
          livenessProbe:
            httpGet:
              path: /api/v1/health
              port: 8666
            initialDelaySeconds: 15
            periodSeconds: 30
          readinessProbe:
            httpGet:
              path: /api/v1/health
              port: 8666
            initialDelaySeconds: 5
            periodSeconds: 10
```

### Horizontal Pod Autoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hades-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hades-api
  minReplicas: 2
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

### Persistent Volume Claims

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: hades-scan-data
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 50Gi
  storageClassName: standard
```

### ConfigMap for Rules and Config

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: hades-config
data:
  hades_config.json: |
    {
      "scanning": {
        "max_file_size_mb": 100,
        "threads": 8
      }
    }
```

Mount as a volume in the deployment spec:

```yaml
volumes:
  - name: config
    configMap:
      name: hades-config
volumeMounts:
  - name: config
    mountPath: /data/config
```

## Monitoring and Alerting

### Key Metrics

| Metric | Source | Alert Condition |
|--------|--------|----------------|
| API response time (p95) | nginx access log | > 2 seconds |
| Scan queue depth | Redis `LLEN hades:tasks` | > 1000 |
| Worker count | Docker / Kubernetes | < minimum replicas |
| Error rate | API logs | > 1% of requests |
| PostgreSQL connections | `pg_stat_activity` | > 80% of `max_connections` |
| Redis memory usage | `redis-cli info memory` | > 80% of `maxmemory` |
| Disk usage (scan data) | Host / PVC | > 85% capacity |
| CPU usage per instance | Docker stats / metrics | > 80% sustained |

### Prometheus Metrics

HADES exposes a health endpoint suitable for Prometheus scraping. For custom metrics, configure a Prometheus exporter sidecar or use the SIEM integration to forward metrics.

### Log Aggregation

Collect logs from all instances for centralized analysis:

```bash
# Docker Compose logs
docker compose -f docker/docker-compose.scale.yml logs -f --tail=100

# Structured JSON logs
export HADES_LOG_LEVEL=info
# Logs include: timestamp, level, module, message, scan_id, duration
```

### Alerting Rules

Set up alerts for:

1. **Health check failure:** Any instance failing `/api/v1/health` for > 60 seconds
2. **Queue backup:** Redis task queue depth > 1000 for > 5 minutes
3. **High error rate:** > 1% 5xx responses over 5 minutes
4. **Resource exhaustion:** Memory or CPU > 90% for > 10 minutes
5. **Database connection pool:** > 90% connections used
