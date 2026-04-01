---
name: infinispan-troubleshoot
description: Use when user encounters Infinispan errors, performance issues, cluster problems, serialization failures, or needs help diagnosing issues with caches, cross-site replication, or transactions.
---

# Infinispan Troubleshooting Guide

Structured diagnosis workflows for common Infinispan problems. Optionally connects to a live MCP server for deeper analysis.

## Workflow

1. Ask the user to describe the symptom (error message, unexpected behavior, performance degradation)
2. Classify the problem category
3. Follow the structured diagnosis flow
4. Suggest MCP server connection when live inspection would help
5. Provide concrete fix with config/code

## Problem Classification

| Symptom | Category | Start Here |
|---------|----------|------------|
| Nodes not joining, split-brain, view changes | Cluster | Cluster Issues |
| `Unknown type`, marshalling exceptions, `ClassNotFoundException` | Serialization | Serialization Errors |
| Slow queries, high latency, memory pressure | Performance | Performance Diagnosis |
| Cache store errors, data not persisted | Persistence | Persistence Problems |
| Deadlocks, lock timeouts, `WriteSkewException` | Transactions | Transaction Issues |
| Backup failures, site unreachable, state transfer timeout | Cross-Site | Cross-Site Issues |

## Cluster Issues

### Nodes Not Joining

**Diagnosis flow:**

1. **Check JGroups stack** — Are all nodes using the same stack (TCP/UDP)?
2. **Check cluster name** — Must be identical on all nodes.
3. **Check discovery** — Is the discovery protocol correct for the environment?
   - LAN with multicast: `UDP` stack with `MPING`
   - Kubernetes: `dns.DNS_PING` with correct service name
   - Cloud/no multicast: `JDBC_PING`, `S3_PING`, or static `TCPPING`
4. **Check firewall** — JGroups ports must be open (default 7800 for TCP, 46655 for UDP).
5. **Check logs** — Look for `GMS` and `MERGE` messages in server log.

**Common fixes:**
```xml
<!-- Kubernetes discovery -->
<dns.DNS_PING dns_query="infinispan-ping.namespace.svc.cluster.local"/>

<!-- Static TCP discovery -->
<TCPPING initial_hosts="node1[7800],node2[7800],node3[7800]"/>
```

### Split-Brain / Partition Handling

**Check partition handling strategy:**
```xml
<distributed-cache name="myCache">
  <partition-handling when-split="DENY_READ_WRITES" merge-policy="PREFERRED_NON_NULL"/>
</distributed-cache>
```

- `ALLOW_READ_WRITES` — Available during split, risk of conflicts.
- `DENY_READ_WRITES` — Unavailable during split, no conflicts.
- `ALLOW_READS` — Read-only during split.

## Serialization Errors

### `Unknown type` / Marshalling Failures

**Diagnosis flow:**

1. **Check encoding** — Cache and client must use matching media types.
2. **Check ProtoStream schema** — Is the `.proto` schema registered on the server?
3. **Check `@Proto` annotations** — Are all fields annotated with `@ProtoField`?
4. **Check `GeneratedSchema`** — Is the schema initializer registered with the client?
5. **Check proto.lock** — Proto schema changes must not break backward compatibility.

**Quick fix checklist:**
```java
// 1. Verify schema is registered on server
RemoteCache<String, String> metadataCache = cacheManager.getCache("___protobuf_metadata");
String errors = metadataCache.get(".errors");
if (errors != null) {
    System.err.println("Schema errors: " + errors);
}

// 2. Check registered schemas
metadataCache.keySet().forEach(System.out::println);
```

### ClassNotFoundException (Embedded)

- Ensure the class is available on all cluster nodes' classpath.
- If using custom externalizers, verify they're registered in the `SerializationContext`.

## Performance Diagnosis

### Slow Queries

1. **Check indexing** — Non-indexed queries scan all entries. Add indexing for queried fields.
2. **Check query plan** — Use `EXPLAIN` to see if index is being used.
3. **Check data volume** — Large result sets slow down queries. Use pagination (`OFFSET`/`LIMIT`).
4. **Rebuild index** — Stale index can degrade performance:
   ```java
   Indexer indexer = Search.getIndexer(cache);
   indexer.run();
   ```

### High Latency

1. **Check cluster size** — More nodes = more replication overhead in REPL_SYNC.
2. **Check numOwners** — Higher owners = more write overhead in DIST_SYNC.
3. **Consider ASYNC mode** — If consistency can be relaxed.
4. **Enable near caching** — For read-heavy Hot Rod clients.
5. **Check JGroups flow control** — `UFC`/`MFC` buffer sizes may need tuning.

### Memory Pressure

1. **Check eviction** — Is `max-count` or `max-size` configured?
2. **Check off-heap** — Use off-heap storage for large datasets:
   ```xml
   <memory storage="OFF_HEAP" max-size="1GB" when-full="REMOVE"/>
   ```
3. **Check passivation** — Enable to move evicted entries to store instead of discarding.

## Persistence Problems

### Store Not Writing

1. **Check connection** — Verify DB/store connectivity.
2. **Check passivation setting** — With `passivation="true"`, entries only write on eviction.
3. **Check write-behind** — Async writes may lag. Check `modification-queue-size`.

### Data Not Available After Restart

1. **Persistence enabled?** — Without a store, all data is in-memory only.
2. **Check `purge-on-startup`** — If true, store is cleared on restart.
3. **Check `drop-on-exit`** — JDBC table drops on shutdown.
4. **Check shared store** — In clustered mode, `shared="true"` means only one node writes.

## Transaction Issues

### Deadlocks

1. **Check lock ordering** — Access keys in consistent order across transactions.
2. **Check lock timeout** — Default is 10 seconds. Increase if transactions are long:
   ```xml
   <locking acquire-timeout="30000"/>
   ```
3. **Check transaction timeout** — JTA transaction timeout may be too short.

### WriteSkewException (Optimistic Locking)

- Another transaction modified the entry between your read and commit.
- **Fix:** Retry the transaction, or switch to `PESSIMISTIC` locking if conflicts are frequent.

### Recovery

For `FULL_XA` mode, check pending transactions:
```java
cache.getAdvancedCache().getXAResource().recover(XAResource.TMSTARTRSCAN);
```

## Cross-Site Issues

### Backup Failures

1. **Check relay configuration** — Is `RELAY2` configured in JGroups stack?
2. **Check site connectivity** — Can relay nodes reach the remote site?
3. **Check backup strategy** — SYNC backups block until remote site acknowledges.
4. **Check timeout** — Increase backup timeout for high-latency links.

### State Transfer Timeout

```xml
<backup site="NYC">
  <state-transfer chunk-size="256" timeout="1200000" max-retries="30" wait-time="2000"/>
</backup>
```

Increase `chunk-size`, `timeout`, and `max-retries` for large datasets.

## MCP Server Integration

When a live Infinispan server is available with MCP enabled, use it for deeper diagnosis.

### Enable MCP

```bash
bin/server.sh -Dorg.infinispan.feature.mcp=true
```

### Available MCP Tools

| Tool | Use For |
|------|---------|
| `getCacheNames` | Verify which caches exist |
| `getCacheEntry` | Inspect specific entries |
| `getCacheConfiguration` | Review cache configuration |
| `getCacheStats` | Check hit/miss ratio, entry count, latency |
| `getClusterHealth` | Cluster status, members, node count |
| `queryCache` | Run Ickle queries to check data state |
| `getSchemas` | Verify registered protobuf schemas |
| `getCounterNames` / `getCounter` | Check counter state |

### Available MCP Resources

| Resource | Use For |
|----------|---------|
| `infinispan+logs://server` | Check server log for errors, cluster events |
| `infinispan+logs://audit` | Review security/auth events |
| `infinispan+logs://rest-access` | Check REST API access patterns |
| `infinispan+logs://hotrod-access` | Check Hot Rod client activity |
| `infinispan+logs://gc` | Diagnose GC pauses and memory issues |

### Diagnosis with MCP

When troubleshooting with MCP:
1. Read server logs first — `infinispan+logs://server?lines=200`
2. Check cluster health — use `getClusterHealth`
3. Check cache state — use `getCacheNames` then `getCacheStats` for specific caches
4. Verify schemas — use `getSchemas` when seeing serialization errors
5. Query data — use `queryCache` with Ickle to check data integrity

## Official Docs

- Troubleshooting: https://infinispan.org/docs/stable/titles/tuning/tuning.html
- JGroups: https://infinispan.org/docs/stable/titles/server/server.html
- Cross-site: https://infinispan.org/docs/stable/titles/xsite/xsite.html
