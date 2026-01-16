# ADR: Object Storage with MinIO

**Status:** Accepted
**Date:** 2024-04-01

## Context

Need S3-compatible object storage for observability backends (Loki, Tempo, Mimir) and application data.

## Decision

Use MinIO deployed on local SSD with erasure coding.

## Rationale

| Option | Cost | Latency | S3 Compatible |
|--------|------|---------|---------------|
| Cloud S3 | $$ | Network | ✅ |
| **MinIO on SSD** | Included | Local | ✅ | **Selected** |
| Ceph | Infrastructure | Local | ✅ |

**Key Decision Factors:**
- Zero additional cost (uses boot disk)
- Local latency for observability backends
- S3 API compatibility
- Erasure coding for durability

## Configuration

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: minio
  namespace: storage
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: minio
          image: minio/minio:latest
          args:
            - server
            - --console-address=:9001
            - http://minio-{0...2}.minio.storage:9000/data
          volumeMounts:
            - name: data
              mountPath: /data
```

## Bucket Structure

| Bucket | Purpose | Retention |
|--------|---------|-----------|
| `loki-chunks` | Log storage | 7 days |
| `tempo-traces` | Trace storage | 7 days |
| `mimir-blocks` | Metrics storage | 30 days |
| `velero-backups` | K8s backups | 30 days |
| `<tenant>-data` | Application data | Tenant-defined |

## High Availability

- 3 MinIO instances
- Erasure coding (EC:2) for durability
- Survives 1 node failure

## Consequences

**Positive:** Zero cost, local latency, S3 compatible, HA via erasure coding
**Negative:** Uses boot disk space, self-managed

## Related

- [SPEC-MINIO-CONFIGURATION](./SPEC-MINIO-CONFIGURATION.md)
