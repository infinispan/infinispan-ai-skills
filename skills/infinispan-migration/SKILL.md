---
name: infinispan-migration
description: Use when user asks about upgrading Infinispan versions, migrating configuration formats (XML/YAML/JSON), handling deprecated features, moving between embedded and server deployments, or migrating cache store data.
---

# Infinispan Migration Guide

Guides version upgrades, configuration migrations, and deployment model transitions.

## Workflow

1. Identify the source and target versions (or deployment models)
2. Check the version-specific migration guide for breaking changes
3. Apply configuration changes — update deprecated/removed elements
4. Migrate data if store formats changed
5. Verify the upgrade — check logs for parsing errors or deprecation warnings

## Diagnostic Workflow with MCP

When a live Infinispan server is available with MCP enabled:

1. **Identify current version**: Use `getClusterInfo` to check the running version.
2. **Read current config**: Use `getServerConfiguration` and `getCacheConfiguration`.
3. **Check schemas**: Use `getSchemas` to identify Protobuf schema changes needed.
4. After upgrade, check `infinispan+logs://server` for parsing errors or deprecation warnings.

## Version Migration Guides

### 15.x to 16.0

#### Breaking Changes

| Area | Change | Action |
|------|--------|--------|
| Marshalling | JBoss Marshalling removed as default | Migrate to ProtoStream. Register `.proto` schemas for all cached types. |
| Configuration | `<compatibility>` element removed | Use `<encoding>` to configure media types instead. |
| Hot Rod | Protocol version 4.0 is minimum | Upgrade Hot Rod clients to 16.x. Older clients will fail to connect. |
| REST | `/rest/v2/` paths restructured | Update REST client URLs. `/v2/caches` remains stable. |
| Security | `ServerIdentitiesConfiguration` simplified | Review TLS/SSL configuration. Keystores now configured under `<ssl>` directly. |
| Persistence | `SingleFileStore` removed | Migrate to `SoftIndexFileStore` (default) or `RocksDBStore`. Use the store migrator. |
| Queries | Ickle `HAVING` clause semantics changed | Review aggregation queries. `HAVING` now filters after `GROUP BY` consistently. |
| JGroups | Minimum JGroups version 5.4 | Upgrade JGroups configuration if using custom stacks. |

#### Configuration Element Changes

| Removed/Changed | Replacement |
|-----------------|-------------|
| `<single-file-store>` | `<file-store>` (uses soft-index implementation) |
| `<compatibility enabled="true">` | `<encoding media-type="..."/>` |
| `<store-as-binary>` | `<encoding media-type="application/x-java-serialized-object"/>` |
| `<lazy-deserialization>` | Removed. Encoding handles this automatically. |
| `<eviction>` | `<memory max-count="..." when-full="REMOVE"/>` |

### 14.x to 15.0

#### Breaking Changes

| Area | Change | Action |
|------|--------|--------|
| Java | Minimum Java 17 | Upgrade JDK. Java 11 no longer supported. |
| Marshalling | ProtoStream is the only default marshaller | Remove `<jboss-marshalling>` configuration. Migrate types to ProtoStream. |
| Persistence | Store migrator required for store format changes | Run the store migrator before upgrading. |
| REST | REST v1 API removed | Migrate all REST calls to `/rest/v2/`. |
| Hot Rod | Client 14.x compatible but deprecated features removed | Test client compatibility before upgrading. |
| Cross-site | Relay configuration restructured | Review RELAY2 configuration in JGroups stack. |

### 13.x to 14.0

#### Breaking Changes

| Area | Change | Action |
|------|--------|--------|
| Java | Minimum Java 11 | Upgrade from Java 8. |
| Configuration | Namespace versioning changed | Update `xmlns` in XML configuration files. |
| Persistence | `LevelDBStore` removed | Migrate to `RocksDBStore`. |
| Counters | Counter configuration syntax changed | Update counter definitions in server config. |

## Configuration Format Migration

Infinispan supports XML, YAML, and JSON configuration formats.
All three are functionally equivalent.

### XML to YAML

**XML:**
```xml
<distributed-cache name="myCache" owners="2">
   <encoding media-type="application/x-protostream"/>
   <memory max-count="10000" when-full="REMOVE"/>
</distributed-cache>
```

**YAML:**
```yaml
distributedCache:
  name: myCache
  owners: 2
  encoding:
    mediaType: application/x-protostream
  memory:
    maxCount: 10000
    whenFull: REMOVE
```

### XML to JSON

```json
{
  "distributed-cache": {
    "name": "myCache",
    "owners": 2,
    "encoding": {
      "media-type": "application/x-protostream"
    },
    "memory": {
      "max-count": 10000,
      "when-full": "REMOVE"
    }
  }
}
```

## Deployment Model Migration

### Embedded to Server

When migrating from embedded (library) mode to Infinispan Server:

1. **Extract cache configuration**: Export `ConfigurationBuilder` code to XML/YAML/JSON.
2. **Register Protobuf schemas**: In embedded mode, annotated Java classes auto-register. In server mode, schemas must be explicitly registered via REST, CLI, or the `registerSchema` MCP tool.
3. **Switch client**: Replace `EmbeddedCacheManager` with `RemoteCacheManager` (Hot Rod).
4. **Handle listeners**: `@Listener` on embedded caches becomes client listeners on Hot Rod, or use the REST SSE endpoint for event streaming.
5. **Handle transactions**: Embedded JTA transactions become Hot Rod transactions. Not all transaction modes are supported over Hot Rod — check compatibility.

### Server to Embedded

Less common, but sometimes needed for testing or edge deployments:

1. **Import cache configuration**: Server XML/YAML/JSON configs can be loaded directly by `DefaultCacheManager`.
2. **Include dependencies**: The server bundles many optional modules. Add only the ones your caches actually use (persistence stores, query, counters, etc.).
3. **Handle security**: Server security realms don't apply in embedded mode. Use programmatic `GlobalConfigurationBuilder.security()` instead.

## Store Migration

When upgrading requires a cache store format change, use the Infinispan Store Migrator:

1. Configure a `source` store pointing to the old format.
2. Configure a `target` store pointing to the new format.
3. Run the migrator: `bin/cli.sh -c https://localhost:11222 migrate store`.
4. Verify data integrity after migration.

For rolling upgrades (zero downtime), use the remote store approach:
1. Set up the new cluster alongside the old one.
2. Configure a `remote-store` on the new cluster pointing to the old cluster.
3. Start the new cluster — it lazily fetches data from the old one.
4. Run `synchronize` to pull all remaining data.
5. Disconnect the remote store.

## Deprecation Tracking

When reviewing a configuration, check for these commonly deprecated patterns:

| Pattern | Status | Replacement |
|---------|--------|-------------|
| `<compatibility>` | Removed in 15.0 | `<encoding>` |
| `<store-as-binary>` | Removed in 15.0 | `<encoding>` |
| `<single-file-store>` | Removed in 16.0 | `<file-store>` |
| `<eviction strategy="...">` | Deprecated since 11.0 | `<memory>` |
| `<lazy-deserialization>` | Removed in 15.0 | Automatic with encoding |
| JBoss Marshalling | Removed in 15.0 | ProtoStream |
| Hot Rod protocol < 3.0 | Unsupported since 14.0 | Upgrade client |
| REST v1 (`/rest/`) | Removed in 15.0 | `/rest/v2/` |

## Official Docs

- Configuration guide: https://infinispan.org/docs/stable/titles/configuring/configuring.html
- Server guide: https://infinispan.org/docs/stable/titles/server/server.html
- Hot Rod client: https://infinispan.org/docs/stable/titles/hotrod_java/hotrod_java.html
