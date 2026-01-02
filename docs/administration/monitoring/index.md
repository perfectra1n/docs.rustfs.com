---
title: Monitoring
description: Monitor your RustFS deployment with Prometheus and other observability tools
---

# Monitoring RustFS

RustFS provides comprehensive monitoring capabilities through native Prometheus endpoints and OpenTelemetry integration.

## Monitoring Options

RustFS supports two methods for exposing metrics:

| Method | Description | Use Case |
|--------|-------------|----------|
| **Native Prometheus Endpoints** | Direct HTTP scraping with JWT auth | Simple setups, direct Prometheus integration |
| **OpenTelemetry Collector** | OTLP export to collector | Complex observability pipelines, multiple backends |

## Quick Start

### Native Prometheus Scraping

1. Generate a Prometheus configuration with JWT token:

```bash
# Using awscurl with S3 signature authentication
awscurl --service s3 --region us-east-1 \
  --access_key $RUSTFS_ACCESS_KEY \
  --secret_key $RUSTFS_SECRET_KEY \
  http://localhost:9000/rustfs/admin/v3/prometheus/config
```

2. Add the generated configuration to your `prometheus.yml`

3. Start scraping metrics from your RustFS cluster

### OpenTelemetry Integration

Configure RustFS to export metrics via OTLP:

```bash
export RUSTFS_OBS_ENDPOINT=http://otel-collector:4318
./rustfs server /data
```

## Available Metrics

RustFS exposes metrics across four categories:

- **Cluster Metrics** - Capacity, bucket count, object count
- **Bucket Metrics** - Per-bucket usage and quotas
- **Node Metrics** - Per-disk capacity and health
- **Resource Metrics** - CPU, memory, process uptime

See [Prometheus Endpoints](./prometheus) for detailed endpoint documentation.

## Related Topics

- [Prometheus Endpoints](./prometheus) - Native Prometheus scrape endpoints
- [OpenTelemetry](./opentelemetry) - OTLP-based observability (coming soon)
