---
title: Prometheus Endpoints
description: Native Prometheus scrape endpoints for RustFS monitoring
---

# Prometheus Endpoints

RustFS exposes native Prometheus scrape endpoints that can be directly scraped by Prometheus without requiring an intermediary collector.

## Authentication

All metrics endpoints require JWT Bearer token authentication. Tokens are generated using the admin configuration endpoint.

### Generating a Token

RustFS provides two methods to generate Prometheus scrape tokens.

#### Simple Method: Query Parameters

Generate a token using query parameters:

```bash
curl -X POST "http://localhost:9000/rustfs/admin/v3/prometheus/token?access_key=$RUSTFS_ACCESS_KEY&secret_key=$RUSTFS_SECRET_KEY"
```

Response:

```json
{"bearer_token": "eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9..."}
```

Extract the token for use in your Prometheus configuration:

```bash
curl -s -X POST "http://localhost:9000/rustfs/admin/v3/prometheus/token?access_key=$RUSTFS_ACCESS_KEY&secret_key=$RUSTFS_SECRET_KEY" | jq -r '.bearer_token'
```

> **Security Note:** Credentials passed via query parameters may appear in server logs. Use HTTPS in production and consider the S3 signature method for stricter security requirements.

#### Advanced Method: S3 Signature with Config

Use S3 signature authentication to get the token along with a complete scrape configuration:

```bash
# Using awscurl
awscurl --service s3 --region us-east-1 \
  --access_key $RUSTFS_ACCESS_KEY \
  --secret_key $RUSTFS_SECRET_KEY \
  "http://localhost:9000/rustfs/admin/v3/prometheus/config"
```

This returns a complete Prometheus scrape configuration YAML with the JWT token embedded, which you can use as a starting point for your Prometheus setup.

### Token Expiration

By default, tokens expire after 30 days. Generate a new token before expiration to maintain uninterrupted monitoring.

### Why Two Authentication Methods?

RustFS provides two ways to generate tokens, each suited to different use cases:

| Endpoint | Method | Authentication | Returns |
|----------|--------|----------------|---------|
| `/rustfs/admin/v3/prometheus/token` | POST | Query Parameters | JWT token only |
| `/rustfs/admin/v3/prometheus/config` | GET | S3 Signature | JWT token + scrape config YAML |

**Simple method (Query Parameters):** Use the `/token` endpoint when you just need a token and already have your Prometheus configuration set up.

**Advanced method (S3 Signature):** Use the `/config` endpoint when setting up Prometheus for the first time or when you want a ready-to-use scrape configuration.

Both endpoints require admin credentials, ensuring only administrators can generate scrape tokens. The separation between token generation (admin credentials) and metrics scraping (JWT Bearer tokens) provides these benefits:

- Only administrators can generate scrape tokens
- Prometheus scrapers use lightweight JWT validation
- Tokens can be rotated without changing admin credentials

## Endpoints

### Cluster Metrics

**Endpoint:** `GET /rustfs/v2/metrics/cluster`

Provides cluster-wide metrics including capacity and usage.

| Metric | Type | Description |
|--------|------|-------------|
| `rustfs_cluster_capacity_raw_total_bytes` | Gauge | Total raw storage capacity |
| `rustfs_cluster_capacity_usable_total_bytes` | Gauge | Usable capacity after erasure coding |
| `rustfs_cluster_capacity_used_bytes` | Gauge | Currently used capacity |
| `rustfs_cluster_capacity_free_bytes` | Gauge | Available capacity |
| `rustfs_cluster_buckets_total` | Gauge | Total number of buckets |
| `rustfs_cluster_objects_total` | Gauge | Total number of objects |

### Bucket Metrics

**Endpoint:** `GET /rustfs/v2/metrics/bucket`

Provides per-bucket metrics with `bucket` label.

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `rustfs_bucket_usage_bytes` | Gauge | bucket | Bytes used by bucket |
| `rustfs_bucket_objects_total` | Gauge | bucket | Object count in bucket |
| `rustfs_bucket_quota_bytes` | Gauge | bucket | Quota limit (if set) |

### Node Metrics

**Endpoint:** `GET /rustfs/v2/metrics/node`

Provides per-node and per-disk metrics with `server` and `drive` labels.

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `rustfs_node_disk_total_bytes` | Gauge | server, drive | Total disk capacity |
| `rustfs_node_disk_used_bytes` | Gauge | server, drive | Used disk space |
| `rustfs_node_disk_free_bytes` | Gauge | server, drive | Free disk space |

### Resource Metrics

**Endpoint:** `GET /rustfs/v2/metrics/resource`

Provides system resource metrics for the RustFS process.

| Metric | Type | Description |
|--------|------|-------------|
| `rustfs_process_cpu_percent` | Gauge | CPU usage percentage |
| `rustfs_process_memory_bytes` | Gauge | Memory usage in bytes |
| `rustfs_process_uptime_seconds` | Gauge | Process uptime |

## Configuration

### Cache Settings

Metrics responses are cached to reduce CPU overhead during frequent scrapes.

| Environment Variable | Default | Description |
|---------------------|---------|-------------|
| `RUSTFS_PROMETHEUS_CACHE_TTL` | `10` | Cache TTL in seconds. Set to `0` to disable caching. |

The default 10-second cache is shorter than typical Prometheus scrape intervals (15-60s), ensuring fresh data while avoiding redundant collection.

```bash
# Use 30-second cache for high-load environments
export RUSTFS_PROMETHEUS_CACHE_TTL=30

# Disable caching entirely (fresh data on every scrape)
export RUSTFS_PROMETHEUS_CACHE_TTL=0
```

## Prometheus Configuration

### Example Configuration

```yaml
global:
  scrape_interval: 60s

scrape_configs:
  - job_name: 'rustfs-cluster'
    bearer_token: '<your-jwt-token>'
    metrics_path: /rustfs/v2/metrics/cluster
    scheme: http
    static_configs:
      - targets: ['rustfs-1:9000', 'rustfs-2:9000', 'rustfs-3:9000']

  - job_name: 'rustfs-bucket'
    bearer_token: '<your-jwt-token>'
    metrics_path: /rustfs/v2/metrics/bucket
    scheme: http
    static_configs:
      - targets: ['rustfs-1:9000']

  - job_name: 'rustfs-node'
    bearer_token: '<your-jwt-token>'
    metrics_path: /rustfs/v2/metrics/node
    scheme: http
    static_configs:
      - targets: ['rustfs-1:9000', 'rustfs-2:9000', 'rustfs-3:9000']

  - job_name: 'rustfs-resource'
    bearer_token: '<your-jwt-token>'
    metrics_path: /rustfs/v2/metrics/resource
    scheme: http
    static_configs:
      - targets: ['rustfs-1:9000', 'rustfs-2:9000', 'rustfs-3:9000']
```

### Scraping Multiple Nodes

For distributed deployments, configure targets to scrape each node:

```yaml
static_configs:
  - targets:
    - 'node1.rustfs.local:9000'
    - 'node2.rustfs.local:9000'
    - 'node3.rustfs.local:9000'
    - 'node4.rustfs.local:9000'
```

## Grafana Dashboard

Import the RustFS Grafana dashboard for visualization:

1. Open Grafana and navigate to Dashboards, then Import
2. Use dashboard ID: `<coming-soon>` or import JSON from the RustFS repository
3. Select your Prometheus data source
4. View cluster health, capacity trends, and performance metrics

## Troubleshooting

### 401 Unauthorized

- Verify the Bearer token is valid and not expired
- Regenerate the token using the `/token` endpoint (Basic Auth) or `/config` endpoint (S3 Signature)
- Ensure the Authorization header format is `Bearer <token>`

### Empty Metrics

- Verify RustFS is running and healthy
- Check that buckets exist (bucket metrics require buckets)
- Ensure the endpoint path is correct

### Connection Refused

- Verify RustFS is listening on the expected address
- Check firewall rules allow access to port 9000
- Verify the scheme (http vs https) matches your configuration
