---
name: infinispan-config
description: Use when user needs help with Infinispan cache configuration - cache modes, persistence, indexing, encoding, security, clustering, cross-site replication, XML/YAML/programmatic setup.
---

# Infinispan Configuration Guide

Help users configure Infinispan by understanding their use case, explaining trade-offs, and providing ready-to-use config snippets.

## Workflow

1. Ask what the user is trying to achieve
2. Determine the config format they prefer (XML, YAML, or programmatic Java)
3. Explain relevant concepts and trade-offs
4. Provide a concrete config snippet
5. Warn about common pitfalls

## Cache Modes

| Mode | Description | Use When |
|------|-------------|----------|
| **Local** | Single-node, no clustering | Development, single-server apps |
| **Invalidation** | Invalidates stale entries across nodes | Shared database with local caching |
| **Replicated** | Full copy on every node | Small datasets, read-heavy, max availability |
| **Distributed** | Partitioned across nodes with numOwners copies | Large datasets, balanced read/write |

### Key Parameters
- `numOwners` (distributed): Number of copies. Default 2. Higher = more resilient, lower throughput.
- `mode`: `SYNC` (consistent, slower) vs `ASYNC` (faster, eventual consistency).
- `capacityFactor`: Weight for data distribution. Set to 0 for zero-capacity nodes.

### Example: Distributed Cache (XML)

```xml
<distributed-cache name="myCache" owners="2" mode="SYNC">
  <encoding media-type="application/x-protostream"/>
  <memory max-count="10000" when-full="REMOVE"/>
  <expiration lifespan="3600000"/>
</distributed-cache>
```

### Example: Distributed Cache (YAML)

```yaml
distributedCache:
  name: myCache
  owners: 2
  mode: SYNC
  encoding:
    mediaType: application/x-protostream
  memory:
    maxCount: 10000
    whenFull: REMOVE
  expiration:
    lifespan: 3600000
```

### Example: Distributed Cache (Programmatic Java)

```java
ConfigurationBuilder builder = new ConfigurationBuilder();
builder.clustering().cacheMode(CacheMode.DIST_SYNC)
       .hash().numOwners(2)
       .encoding().mediaType("application/x-protostream")
       .memory().maxCount(10000).whenFull(EvictionStrategy.REMOVE)
       .expiration().lifespan(3600000);
```

## Persistence

| Store | Use When |
|-------|----------|
| **JDBC** | Relational database backend (MySQL, PostgreSQL, Oracle, etc.) |
| **RocksDB** | High-performance local persistence |
| **Remote** | Cascading Infinispan clusters |

### JDBC Store Example (XML)

```xml
<distributed-cache name="persistent">
  <persistence passivation="false">
    <jdbc:string-keyed-jdbc-store>
      <jdbc:connection-pool connection-url="jdbc:postgresql://localhost:5432/ispn"
                            username="user" password="pass" driver="org.postgresql.Driver"/>
      <jdbc:string-keyed-table prefix="ISPN" create-on-start="true" drop-on-exit="false">
        <jdbc:id-column name="ID" type="VARCHAR(255)"/>
        <jdbc:data-column name="DATA" type="BYTEA"/>
        <jdbc:timestamp-column name="TS" type="BIGINT"/>
        <jdbc:segment-column name="SEG" type="INT"/>
      </jdbc:string-keyed-table>
    </jdbc:string-keyed-jdbc-store>
  </persistence>
</distributed-cache>
```

### Key Decisions
- **passivation=true**: Entries evicted from memory go to store. Store is overflow only.
- **passivation=false**: All entries written to store. Store is persistent backup.
- **write-behind**: Async writes to store. Better throughput, risk of data loss.

## Indexing & Querying

Enable indexing for Ickle query performance on large datasets.

```xml
<distributed-cache name="indexed">
  <encoding media-type="application/x-protostream"/>
  <indexing storage="local-heap">
    <indexed-entities>
      <indexed-entity>org.example.MyEntity</indexed-entity>
    </indexed-entities>
  </indexing>
</distributed-cache>
```

**Storage options:**
- `local-heap` — In-memory index. Fast, lost on restart.
- `filesystem` — Persisted index. Survives restart.

**Pitfall:** Encoding MUST be `application/x-protostream` for indexed caches. Forgetting this is the #1 indexing mistake.

## Encoding

Controls how entries are serialized in the cache.

| Media Type | Use When |
|------------|----------|
| `application/x-protostream` | Default. Required for indexing/querying. Best interop. |
| `application/x-java-serialized-object` | Java-only clients, legacy apps |
| `application/json` | REST clients, language-agnostic |
| `text/plain` | Simple string key/values |

**Pitfall:** Client and server encoding must match, or you'll get serialization errors. If using Hot Rod with ProtoStream, the server cache must also use `application/x-protostream`.

## Security

### Authentication

```xml
<server>
  <security>
    <security-realms>
      <security-realm name="default">
        <!-- Properties-based auth -->
        <properties-realm groups-attribute="Roles">
          <user-properties path="users.properties"/>
          <group-properties path="groups.properties"/>
        </properties-realm>
        <!-- OR LDAP -->
        <ldap-realm url="ldap://ldap-server:389" principal="cn=admin"
                    credential="password">
          <identity-mapping rdn-identifier="uid" search-dn="ou=People,dc=example,dc=com"/>
        </ldap-realm>
      </security-realm>
    </security-realms>
  </security>
</server>
```

### Authorization

```xml
<cache-container>
  <security>
    <authorization>
      <role name="admin" permissions="ALL"/>
      <role name="reader" permissions="READ BULK_READ"/>
      <role name="writer" permissions="READ WRITE BULK_READ"/>
    </authorization>
  </security>
</cache-container>

<distributed-cache name="secured">
  <security>
    <authorization roles="admin reader writer"/>
  </security>
</distributed-cache>
```

## Clustering

### JGroups Transport

```xml
<cache-container>
  <transport stack="tcp" cluster="my-cluster" node-name="node1"/>
</cache-container>
```

**Stacks:**
- `tcp` — Unicast. Use for WAN, cloud, or when multicast is unavailable.
- `udp` — Multicast. Use for LAN with multicast support.

**Cloud discovery:** Use `dns.DNS_PING` (Kubernetes), `JDBC_PING` (shared DB), or cloud-specific protocols (S3_PING, AZURE_PING, GOOGLE_PING2).

## Cross-Site Replication

Configure backup sites for geographic redundancy.

```xml
<distributed-cache name="xsite-cache">
  <backups>
    <backup site="NYC" strategy="ASYNC" timeout="10000">
      <state-transfer chunk-size="512" timeout="600000"/>
    </backup>
    <backup site="LON" strategy="SYNC" timeout="15000"/>
  </backups>
</distributed-cache>
```

### Key Decisions
- **SYNC backups**: Strong consistency across sites. Higher latency.
- **ASYNC backups**: Low latency. Risk of data loss if site fails.
- **Active-Active**: Both sites accept writes. Requires conflict resolution.
- **Active-Passive**: One site handles writes, other is standby.

### Conflict Resolution (Active-Active)

```xml
<backup site="NYC" strategy="ASYNC">
  <conflict-resolution merge-policy="PREFER_NON_NULL"/>
</backup>
```

Merge policies: `PREFER_NON_NULL`, `PREFER_ORIGIN_SITE`, `REMOVE_ALL`, or custom `ConflictManager` implementation.

### Relay Configuration

Cross-site requires JGroups RELAY2 protocol. Each site needs a relay node.

```xml
<jgroups>
  <stack name="xsite" extends="tcp">
    <relay.RELAY2 site="LON" max_site_masters="2"/>
    <remote-sites default-stack="tcp">
      <remote-site name="NYC"/>
    </remote-sites>
  </stack>
</jgroups>
```

## Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Queries fail with "unknown type" | Set encoding to `application/x-protostream` and register protobuf schema |
| Cluster nodes don't discover each other | Check JGroups stack, firewall rules, and discovery protocol |
| Data loss after restart | Enable persistence with `passivation="false"` |
| Cross-site state transfer timeout | Increase `chunk-size` and `timeout` values |
| Encoding mismatch between client/server | Ensure both use the same media type |

## Official Docs

- Configuration guide: https://infinispan.org/docs/stable/titles/configuring/configuring.html
- Cross-site replication: https://infinispan.org/docs/stable/titles/xsite/xsite.html
- Security: https://infinispan.org/docs/stable/titles/security/security.html
