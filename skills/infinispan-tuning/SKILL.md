---
name: infinispan-tuning
description: Use when user reports Infinispan performance problems - slow operations, high latency, GC pressure, memory issues, throughput bottlenecks - or asks about JVM sizing, capacity planning, or monitoring metrics.
---

# Infinispan Tuning Guide

Performance tuning guidance for Infinispan server and embedded deployments.
Always gather data before recommending changes.

## Workflow

1. Gather current state (logs, metrics, JVM info, cache stats)
2. Identify the bottleneck category (CPU, memory/GC, persistence I/O, network)
3. Apply one change at a time from the relevant section below
4. Measure the impact before making further changes

## Diagnostic Workflow with MCP

When a live Infinispan server is available with MCP enabled, gather data before tuning:

1. **JVM memory**: Use `getJvmMemory` — check heap usage, GC stats, memory pools.
2. **JVM threads**: Use `getJvmThreads` — check thread counts and deadlocks.
3. **JVM info**: Use `getJvmInfo` — check JVM version, uptime, input arguments.
4. **Cache stats**: Use `getCacheStats` for each relevant cache — hit/miss ratios, eviction counts, operation timings.
5. **Server config**: Use `getServerConfiguration` — thread pool sizes, JGroups settings, endpoint configuration.
6. **Logs**: Read `infinispan+logs://server` for warnings and `infinispan+logs://gc` for GC behavior.

## JVM Tuning

### Heap Sizing

| Guideline | Detail |
|-----------|--------|
| Minimum heap | Set `-Xms` equal to `-Xmx` to avoid resize pauses |
| Maximum heap | Leave 30-40% of physical memory for off-heap, OS, and JGroups buffers |
| Off-heap | Infinispan stores data off-heap by default; this memory is NOT counted in heap |
| Compressed oops | Keep heap under 32 GB to benefit from compressed ordinary object pointers |

### GC Selection

| GC | When to use |
|----|-------------|
| G1GC | Default. Good for heaps 4-32 GB. Balanced throughput and latency. |
| ZGC | Heaps > 16 GB where low pause times are critical. Use `-XX:+UseZGC`. |
| Shenandoah | Alternative low-pause GC. Use `-XX:+UseShenandoahGC`. |

### GC Tuning Flags

For G1GC (default):
```
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:InitiatingHeapOccupancyPercent=45
-XX:G1HeapRegionSize=16m
```

For ZGC (latency-sensitive):
```
-XX:+UseZGC
-XX:+ZGenerational
```

### Diagnostic Checklist

1. Check heap used percentage. If consistently > 80%, increase heap or enable eviction.
2. Check GC collection time. If GC time is > 5% of uptime, tune GC or reduce heap pressure.
3. Check GC log for long pauses (> 500ms). Consider switching to ZGC or Shenandoah.
4. In containers, set `-Xmx` to ~60-70% of the container memory limit. The rest is needed for off-heap, metaspace, and OS.

## Cache Tuning

### Number of Segments

Segments control data distribution granularity. Default: 256.

| Adjustment | When |
|------------|------|
| Decrease (64-128) | Small caches (< 10,000 entries), fewer nodes (2-3) |
| Keep default (256) | Most deployments |
| Increase (512-1024) | Very large caches (> 10M entries), many nodes (> 10) |

More segments = better distribution but more metadata overhead.

### Eviction and Memory

| Strategy | Configuration | Use case |
|----------|---------------|----------|
| Count-based | `<memory max-count="10000"/>` | Limit by number of entries |
| Memory-based | `<memory max-size="500MB"/>` | Limit by memory consumption (requires size estimation) |
| No eviction | (default) | Data always fits in memory, or persistence handles overflow |

Eviction strategies: `REMOVE` (default, evict oldest) or `EXCEPTION` (reject new entries when full).

### Near-Caching (Hot Rod Clients)

Enable near-caching on the client to avoid network round-trips for hot reads:

```java
builder.remoteCache("myCache")
   .nearCacheMode(NearCacheMode.INVALIDATED)
   .nearCacheMaxEntries(1000);
```

### Batching and Bulk Operations

- Use `putAll` instead of individual `put` calls for bulk inserts.
- Use `getAll` for bulk reads.
- For Hot Rod, these reduce round-trips significantly.

## Persistence Tuning

### Write-Behind (Async)

Decouple cache writes from store writes to reduce latency:

```xml
<persistence>
   <file-store>
      <write-behind modification-queue-size="1024" />
   </file-store>
</persistence>
```

### JDBC Connection Pool

For `jdbc-string-store`, ensure the connection pool is sized for the write throughput:

| Parameter | Guideline |
|-----------|-----------|
| `max-pool-size` | At least equal to the number of write threads |
| `min-pool-size` | 10-20% of max for steady-state |
| `idle-removal` | Enable to reclaim connections during low activity |

### Store Segmentation

Segmented stores (default for file stores) parallelize reads/writes across segments.
Ensure your underlying storage can handle parallel I/O.

## Network Tuning

### JGroups Thread Pools

| Pool | Default | Adjust when |
|------|---------|-------------|
| `max-threads` (default pool) | 200 | High concurrent operations across nodes |
| `max-threads` (internal pool) | 4 | Rarely needs adjustment |

### TCP Buffer Sizes

For high-throughput clusters, increase TCP send/receive buffers:

```xml
<TCP recv_buf_size="5000000"
     send_buf_size="5000000" />
```

### Bundler Configuration

JGroups bundles small messages to reduce syscalls. The default `transfer-queue` bundler
works well for most cases. For latency-sensitive deployments:

```xml
<TCP bundler_type="no-bundler" />
```

## Monitoring — Key Metrics

| Metric | What it tells you | Alert threshold |
|--------|-------------------|-----------------|
| `hits / (hits + misses)` | Cache hit ratio | < 80% — review access patterns or increase cache size |
| `evictions` | Entries evicted per second | Sustained high rate — increase memory or reduce dataset |
| `averageWriteTime` | Mean write latency | > 10ms — check persistence, network, or contention |
| `averageReadTime` | Mean read latency | > 1ms (local), > 5ms (clustered) — check near-caching |
| GC pause time | Max stop-the-world pause | > 500ms — tune GC |
| Heap used % | Memory pressure | > 85% sustained — increase heap or enable eviction |

## Official Docs

- Planning and tuning: https://infinispan.org/docs/stable/titles/tuning/tuning.html
- JGroups: https://infinispan.org/docs/stable/titles/server/server.html
