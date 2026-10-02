---
name: infinispan-deploy
description: Use when user needs help deploying Infinispan on Kubernetes or OpenShift - Infinispan Operator, Helm charts, container images, scaling, upgrades, monitoring.
---

# Infinispan Deployment Guide

Help users deploy and operate Infinispan on Kubernetes/OpenShift using the Infinispan Operator, Helm charts, or container images.

## Workflow

1. Determine target environment (OpenShift, vanilla Kubernetes, local dev)
2. Determine deployment method (Operator, Helm, plain manifests)
3. Provide tailored YAML manifests and configuration
4. Explain operational concerns (scaling, upgrades, monitoring)

## Infinispan Operator

The Operator is the recommended deployment method for Kubernetes/OpenShift.

### Installation

```bash
# OpenShift (OperatorHub)
# Install via OperatorHub UI or:
oc apply -f https://operatorhub.io/install/infinispan.yaml

# Vanilla Kubernetes
kubectl apply -f https://operatorhub.io/install/infinispan.yaml

# Or via OLM
kubectl create -f infinispan-subscription.yaml
```

### Infinispan CR (Custom Resource)

```yaml
apiVersion: infinispan.org/v1
kind: Infinispan
metadata:
  name: infinispan
  namespace: my-namespace
spec:
  replicas: 3
  version: "16.0"
  expose:
    type: Route           # OpenShift
    # type: LoadBalancer   # Kubernetes
    # type: NodePort       # Local dev
  security:
    endpointSecretName: connect-secret
  container:
    cpu: "2000m"
    memory: 2Gi
    extraJvmOpts: "-Xms1g -Xmx1g"
  service:
    type: DataGrid        # Full Infinispan. Use "Cache" for simpler caching only.
    container:
      storage: 10Gi
      storageClassName: standard
  logging:
    categories:
      org.infinispan: info
      org.jgroups: info
```

### Authentication Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: connect-secret
type: Opaque
stringData:
  identities.yaml: |
    credentials:
      - username: admin
        password: changeme
        roles:
          - admin
      - username: developer
        password: changeme
        roles:
          - application
```

### Cache CR

```yaml
apiVersion: infinispan.org/v2alpha1
kind: Cache
metadata:
  name: my-cache
spec:
  clusterName: infinispan
  name: my-cache
  templateName: org.infinispan.DIST_SYNC
  # Or use a full template:
  # template: |
  #   distributedCache:
  #     owners: 2
  #     mode: SYNC
  #     encoding:
  #       mediaType: application/x-protostream
```

### Batch CR

Run admin operations as batch jobs:

```yaml
apiVersion: infinispan.org/v2alpha1
kind: Batch
metadata:
  name: create-caches
spec:
  cluster: infinispan
  config: |
    create cache --template=org.infinispan.DIST_SYNC myCache1
    create cache --template=org.infinispan.DIST_SYNC myCache2
```

## Helm Charts

```bash
# Add repo
helm repo add infinispan https://infinispan.org/charts

# Install
helm install infinispan infinispan/infinispan \
  --set replicas=3 \
  --set security.secretName=connect-secret \
  --set container.resources.requests.memory=2Gi \
  --set container.resources.requests.cpu=1000m

# Custom values
helm install infinispan infinispan/infinispan -f values.yaml
```

### Example values.yaml

```yaml
replicas: 3
security:
  secretName: connect-secret
container:
  resources:
    requests:
      memory: 2Gi
      cpu: "1000m"
    limits:
      memory: 2Gi
      cpu: "2000m"
  extraJvmOpts: "-Xms1g -Xmx1g"
expose:
  type: LoadBalancer
persistence:
  size: 10Gi
  storageClassName: standard
```

## Container Images

### Official Image

```bash
# Pull
podman pull quay.io/infinispan/server:16.0

# Run locally
podman run -p 11222:11222 \
  -e USER="admin" \
  -e PASS="changeme" \
  quay.io/infinispan/server:16.0
```

### Custom Configuration

```bash
# Mount custom config
podman run -p 11222:11222 \
  -v /path/to/infinispan.xml:/opt/infinispan/server/conf/infinispan.xml:Z \
  -e USER="admin" \
  -e PASS="changeme" \
  quay.io/infinispan/server:16.0
```

### Building Custom Image

```dockerfile
FROM quay.io/infinispan/server:16.0
COPY infinispan.xml /opt/infinispan/server/conf/infinispan.xml
COPY custom-lib.jar /opt/infinispan/server/lib/custom-lib.jar
```

## Scaling

### Scale Up

```bash
# Operator
kubectl patch infinispan infinispan -p '{"spec":{"replicas":5}}' --type=merge

# Check state transfer progress
kubectl logs -f infinispan-0 | grep "STATE_TRANSFER"
```

### Scale Down (Graceful Shutdown)

The Operator handles graceful shutdown automatically:
1. Stops accepting new requests
2. Completes in-flight operations
3. Transfers state to remaining nodes
4. Shuts down the pod

**Pitfall:** Don't scale down too fast. Allow state transfer to complete between each scale-down step.

### Zero-Capacity Nodes

For query/compute nodes that don't hold data, set `capacityFactor` to `0` in the cache configuration (not to be confused with persistent storage size):

```xml
<distributed-cache name="myCache">
  <memory max-count="10000"/>
  <!-- This node won't own any data segments -->
</distributed-cache>
```

Configure `zero-capacity-node=true` on the server to make a node zero-capacity for all caches by default.

## Upgrades

### Rolling Upgrade (Operator)

```yaml
# Update the version in the Infinispan CR
spec:
  version: "16.1"
```

The Operator performs a rolling upgrade:
1. Creates new pod with new version
2. Waits for it to join the cluster
3. Transfers state
4. Shuts down old pod
5. Repeats for each pod

### Migration Between Major Versions

For major version upgrades:
1. Deploy new cluster alongside old
2. Configure remote store pointing to old cluster
3. Migrate data
4. Switch clients to new cluster
5. Decommission old cluster

## Monitoring

### Prometheus Metrics

The Infinispan server exposes metrics at `/metrics` by default.

```yaml
# ServiceMonitor for Prometheus Operator
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: infinispan-monitor
spec:
  selector:
    matchLabels:
      app: infinispan-service
  endpoints:
    - port: infinispan-metrics
      path: /metrics
      interval: 30s
```

### Key Metrics to Watch

| Metric | What It Tells You |
|--------|-------------------|
| `cache_manager_status` | Cluster health |
| `cache_container_stats_number_of_entries` | Cache size |
| `cache_container_stats_hits` / `misses` | Hit ratio |
| `cache_container_stats_average_read_time` | Read latency |
| `cache_container_stats_average_write_time` | Write latency |
| `jvm_memory_used_bytes` | JVM memory usage |

### Health Endpoint

```bash
curl http://infinispan:11222/rest/v2/cache-managers/default/health
```

Returns cluster health, node status, and cache health.

### Grafana

Import the Infinispan Grafana dashboard or create custom dashboards from the Prometheus metrics above.

## Common Deployment Pitfalls

| Pitfall | Fix |
|---------|-----|
| Pods stuck in `Pending` | Check PVC provisioning, resource limits, node capacity |
| Nodes can't discover each other | Verify `dns.DNS_PING` query matches headless service |
| OOM kills | Set `-Xmx` to ~60-70% of container memory limit |
| State transfer timeout on scale | Increase timeout, reduce chunk size |
| Data loss on pod restart | Ensure persistent volumes are configured |
| Slow startup with large datasets | Increase startup probe timeout |

## Official Docs

- Operator guide: https://infinispan.org/docs/infinispan-operator/main/operator.html
- Server guide: https://infinispan.org/docs/stable/titles/server/server.html
- Container image: https://github.com/infinispan/infinispan-images
