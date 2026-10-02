---
name: infinispan-resp
description: Use when user asks about Infinispan RESP endpoint, Redis compatibility, migrating from Redis to Infinispan, or using Redis clients (Jedis, Lettuce, Spring Data Redis) with Infinispan.
---

# Infinispan RESP Endpoint Guide

Help users configure and use the Infinispan RESP (Redis-compatible) endpoint, migrate from Redis, and connect with standard Redis clients.

## Workflow

1. Determine if the user wants to connect Redis clients, migrate from Redis, or configure the RESP endpoint
2. Identify their deployment mode (standalone vs. clustered)
3. Provide concrete configuration and client code
4. Warn about behavioral differences from native Redis

## Overview

The RESP endpoint is **enabled by default** on the Infinispan single-port endpoint (port 11222). Redis client connections are automatically detected and routed to the internal RESP connector. No separate port or explicit enablement is needed.

Infinispan supports **RESP3 protocol only**. RESP2 connections will be rejected with an error.

The RESP endpoint works with:
- **Standalone** deployments (like standalone Redis)
- **Clustered** deployments (replicated or distributed data with automatic failover)

### Verification

When Infinispan Server starts, look for this log message:

```
[org.infinispan.SERVER] ISPN080018: Started connector Resp (internal)
```

Quick test with redis-cli:

```bash
redis-cli -p 11222 --user username --pass password --resp3
127.0.0.1:11222> SET k v
OK
127.0.0.1:11222> GET k
"v"
```

## RESP Cache Configuration

The RESP endpoint automatically creates a `respCache` cache with these requirements:
- **Key encoding**: `application/octet-stream` (mandatory)
- **Hash partitioner**: `RESPHashFunctionPartitioner` (mandatory for clustered mode, supports CRC16 hashing)
- Cache type: `local-cache` (standalone) or `distributed-cache` (clustered)

### Default Cache (XML)

```xml
<distributed-cache name="respCache" aliases="0" owners="2"
                   key-partitioner="org.infinispan.distribution.ch.impl.RESPHashFunctionPartitioner"
                   mode="SYNC" remote-timeout="17500" statistics="true">
    <encoding media-type="application/octet-stream"/>
</distributed-cache>
```

### Default Cache (YAML)

```yaml
respCache:
  distributedCache:
    aliases:
      - "0"
    owners: "2"
    keyPartitioner: "org.infinispan.distribution.ch.impl.RESPHashFunctionPartitioner"
    mode: "SYNC"
    statistics: "true"
    encoding:
      mediaType: "application/octet-stream"
```

### Default Cache (JSON)

```json
{
  "respCache": {
    "distributed-cache": {
      "aliases": ["0"],
      "owners": "2",
      "key-partitioner": "org.infinispan.distribution.ch.impl.RESPHashFunctionPartitioner",
      "mode": "SYNC",
      "statistics": true,
      "encoding": {
        "media-type": "application/octet-stream"
      }
    }
  }
}
```

### Custom Cache Constraints

When providing a custom cache configuration, these constraints are enforced (server will fail to start if violated):
- Key encoding **must** be `application/octet-stream`
- Hash partitioner **must** be `org.infinispan.distribution.ch.impl.RESPHashFunctionPartitioner`

To view entries in the Infinispan Console, configure value encoding with Protobuf:

```xml
<encoding>
    <key media-type="application/octet-stream"/>
    <value media-type="application/x-protostream"/>
</encoding>
```

## Explicit RESP Connector Configuration

If the default implicit configuration does not fit, configure the RESP connector explicitly:

### XML

```xml
<endpoints>
  <endpoint socket-binding="default" security-realm="default">
    <resp-connector cache="mycache" />
    <hotrod-connector />
    <rest-connector/>
  </endpoint>
</endpoints>
```

### YAML

```yaml
server:
  endpoints:
    endpoint:
      socketBinding: "default"
      securityRealm: "default"
      respConnector:
        cache: "mycache"
      hotrodConnector: ~
      restConnector: ~
```

### JSON

```json
{
  "server": {
    "endpoints": {
      "endpoint": {
        "socket-binding": "default",
        "security-realm": "default",
        "resp-connector": {
          "cache": "mycache"
        },
        "hotrod-connector": {},
        "rest-connector": {}
      }
    }
  }
}
```

## Database Mapping (SELECT command)

Redis logical databases map to Infinispan cache aliases. The default `respCache` is aliased to database `0`.

Use the cache `aliases` configuration attribute to map additional caches to logical database numbers:

```xml
<distributed-cache name="sessionCache" aliases="1"
                   key-partitioner="org.infinispan.distribution.ch.impl.RESPHashFunctionPartitioner"
                   mode="SYNC">
    <encoding media-type="application/octet-stream"/>
</distributed-cache>
```

Then use `SELECT 1` in Redis clients to switch to this cache.

**Key advantage over Redis Cluster**: Infinispan supports `SELECT` with multiple logical databases even in clustered mode, whereas Redis Cluster restricts to database `0` only.

## Supported Redis Commands

Infinispan 16.x supports approximately 290 Redis commands across the following categories:

| Category | Commands | Examples |
|----------|----------|----------|
| **Strings** | 30 | GET, SET, MGET, MSET, INCR, APPEND, GETRANGE, SETRANGE, LCS |
| **Hashes** | 16 | HGET, HSET, HDEL, HGETALL, HINCRBY, HKEYS, HVALS, HSCAN |
| **Lists** | 19 | LPUSH, RPUSH, LPOP, RPOP, LRANGE, LINDEX, LINSERT, LMOVE, LMPOP |
| **Sets** | 17 | SADD, SREM, SMEMBERS, SINTER, SUNION, SDIFF, SPOP, SRANDMEMBER |
| **Sorted Sets** | 33 | ZADD, ZRANGE, ZRANK, ZSCORE, ZINCRBY, ZINTERSTORE, ZUNIONSTORE |
| **Bitmaps** | 9 | SETBIT, GETBIT, BITCOUNT, BITOP, BITFIELD, BITPOS |
| **HyperLogLog** | 3 | PFADD, PFCOUNT, PFMERGE |
| **Geo** | 12 | GEOADD, GEODIST, GEOHASH, GEOPOS, GEOSEARCH, GEOSEARCHSTORE |
| **JSON** | 28 | JSON.SET, JSON.GET, JSON.DEL, JSON.MGET, JSON.ARRAPPEND, JSON.OBJKEYS |
| **Pub/Sub** | 10 | SUBSCRIBE, PUBLISH, PSUBSCRIBE, UNSUBSCRIBE |
| **Transactions** | 6 | MULTI, EXEC, DISCARD, WATCH, UNWATCH |
| **Scripting** | Lua | EVAL, EVAL_RO |
| **Cluster** | 8 | CLUSTER NODES, CLUSTER SLOTS, CLUSTER SHARDS, CLUSTER MYID |
| **Bloom Filter** | 14 | BF.ADD, BF.EXISTS, BF.INSERT, BF.MADD, BF.RESERVE, BF.INFO |
| **Cuckoo Filter** | 17 | CF.ADD, CF.ADDNX, CF.EXISTS, CF.INSERT, CF.RESERVE, CF.DEL |
| **Count-Min Sketch** | 12 | CMS.INCRBY, CMS.QUERY, CMS.MERGE, CMS.INFO |
| **Top-K** | 15 | TOPK.ADD, TOPK.QUERY, TOPK.LIST, TOPK.RESERVE, TOPK.INCRBY |
| **Search** | 1 | FT._LIST |
| **Connection** | 15 | AUTH, PING, ECHO, HELLO, SELECT, CLIENT, QUIT, RESET |
| **Generic** | 23 | DEL, EXISTS, EXPIRE, TTL, TYPE, KEYS, SCAN, RENAME, COPY, SORT |

## Differences from Native Redis

### RESP Protocol Version

Infinispan only supports **RESP3**. Clients must negotiate RESP3 via the HELLO command. Attempting RESP2 results in an error. Most modern Redis clients support RESP3.

### Isolation and Atomicity

Redis uses a single thread for all requests, giving serializable isolation. Infinispan uses a **relaxed isolation level**:

- Multi-key commands like `MSET` may show partial results to concurrent readers
- For atomic behavior, wrap operations in `MULTI...EXEC` blocks
- This requires enabling transactions on the cache

### Transactions (MULTI/EXEC)

Enable transactional capabilities on the RESP cache to use `MULTI...EXEC`:

```xml
<distributed-cache name="respCache" aliases="0"
                   key-partitioner="org.infinispan.distribution.ch.impl.RESPHashFunctionPartitioner"
                   mode="SYNC">
    <encoding media-type="application/octet-stream"/>
    <transaction mode="NON_XA" locking="PESSIMISTIC"/>
</distributed-cache>
```

Key differences from Redis transactions:
- Infinispan provides **ACID transactions with rollback** on failure (Redis does not support rollback)
- Transactions are **distributed** in cluster mode and can operate across many slots
- Recommended: `PESSIMISTIC` locking with transaction mode other than `NONE`
- Infinispan does **not** provide serializable transactions

### Lua Scripting

- `EVAL` and `EVAL_RO` are supported
- Scripts use **RESP3 serialization** (equivalent to `redis.setresp(3)` in every script)
- Unlike Redis, Infinispan does **not** cache scripts -- they are discarded immediately after execution

### Other Notable Differences

- `HELLO`: Only RESP3 (protocol version 3) is supported
- `FLUSHALL`: Behaves like `FLUSHDB`, flushing only the current database
- `INFO`: Returns all standard Redis attributes, but most values are `0` (not applicable to Infinispan)
- `SCAN`: Cursors are reaped after 5 minutes of inactivity
- `LINSERT`: O(N) time complexity (N = list size)
- `LMOVE`: Atomic for same-list rotation; relaxed consistency for different lists unless transactions are configured
- `SMOVE`: Not atomic -- element may briefly disappear from both source and destination
- `MEMORY USAGE`: Does not include metadata overhead
- `MODULE LIST`: Always returns empty list
- `JSON.GET` with RegEx: Pattern can use Redis style `(i)pattern` or Java style; dynamic filters with regex are not supported

## Clustered Mode

Infinispan provides a **horizontally scalable RESP-compatible server** with built-in clustering:

- Automatic failover detection and membership discovery
- Consistent hashing compatible with Redis hash-slot algorithm
- Automatic request routing to the correct entry owner
- **No need for hash tags** or `-MOVED` error handling, even for multi-key operations across different owners
- Adding/removing nodes automatically redistributes data without downtime

## Authentication and Security

Authentication uses the Infinispan security realm. Clients authenticate using the `AUTH` command:

```bash
redis-cli -p 11222 --user myuser --pass mypassword
```

Or programmatically:

```
AUTH myuser mypassword
```

Users must be created via the Infinispan CLI or user tool before connecting:

```bash
bin/cli.sh user create myuser -p mypassword
```

The RESP endpoint inherits the security realm configured for the endpoint. Configure it in the server configuration:

```xml
<endpoints>
  <endpoint socket-binding="default" security-realm="default">
    <resp-connector />
  </endpoint>
</endpoints>
```

## Redis Client Integration

### Jedis (Java)

```xml
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>5.2.0</version>
</dependency>
```

```java
import redis.clients.jedis.DefaultJedisClientConfig;
import redis.clients.jedis.Jedis;

// Infinispan requires RESP3
DefaultJedisClientConfig config = DefaultJedisClientConfig.builder()
    .user("admin")
    .password("password")
    .protocol(redis.clients.jedis.RedisProtocol.RESP3)
    .build();

try (Jedis jedis = new Jedis("localhost", 11222, config)) {
    jedis.set("key", "value");
    String value = jedis.get("key");
    System.out.println(value); // "value"

    // Hash operations
    jedis.hset("user:1", "name", "Alice");
    jedis.hset("user:1", "email", "alice@example.com");
    Map<String, String> user = jedis.hgetAll("user:1");

    // Sorted set
    jedis.zadd("leaderboard", 100, "player1");
    jedis.zadd("leaderboard", 200, "player2");
    List<String> top = jedis.zrevrange("leaderboard", 0, 9);
}
```

### Lettuce (Java)

```xml
<dependency>
    <groupId>io.lettuce</groupId>
    <artifactId>lettuce-core</artifactId>
    <version>6.4.0.RELEASE</version>
</dependency>
```

```java
import io.lettuce.core.RedisClient;
import io.lettuce.core.RedisURI;
import io.lettuce.core.api.StatefulRedisConnection;
import io.lettuce.core.api.sync.RedisCommands;
import io.lettuce.core.protocol.ProtocolVersion;

RedisURI uri = RedisURI.builder()
    .withHost("localhost")
    .withPort(11222)
    .withAuthentication("admin", "password".toCharArray())
    .build();

RedisClient client = RedisClient.create(uri);
client.setOptions(io.lettuce.core.ClientOptions.builder()
    .protocolVersion(ProtocolVersion.RESP3)
    .build());

try (StatefulRedisConnection<String, String> connection = client.connect()) {
    RedisCommands<String, String> commands = connection.sync();
    commands.set("key", "value");
    String value = commands.get("key");

    // JSON operations (if using RedisJSON-compatible commands)
    commands.dispatch(io.lettuce.core.protocol.CommandType.valueOf("JSON.SET"),
        new io.lettuce.core.output.StatusOutput<>(StringCodec.UTF8),
        new io.lettuce.core.protocol.CommandArgs<>(StringCodec.UTF8)
            .addKey("doc:1").add("$").add("{\"name\":\"Alice\",\"age\":30}"));
}
client.shutdown();
```

### Spring Data Redis

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

**application.properties**:

```properties
spring.data.redis.host=localhost
spring.data.redis.port=11222
spring.data.redis.username=admin
spring.data.redis.password=password
```

For Lettuce (default in Spring Boot), force RESP3 via a custom `LettuceClientConfigurationBuilderCustomizer`:

```java
import io.lettuce.core.ClientOptions;
import io.lettuce.core.protocol.ProtocolVersion;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.boot.autoconfigure.data.redis.LettuceClientConfigurationBuilderCustomizer;

@Configuration
public class RedisConfig {

    @Bean
    public LettuceClientConfigurationBuilderCustomizer lettuceCustomizer() {
        return builder -> builder.clientOptions(
            ClientOptions.builder()
                .protocolVersion(ProtocolVersion.RESP3)
                .build()
        );
    }
}
```

Then use `StringRedisTemplate` or `RedisTemplate` normally:

```java
@Service
public class CacheService {

    private final StringRedisTemplate redisTemplate;

    public CacheService(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    public void cacheValue(String key, String value) {
        redisTemplate.opsForValue().set(key, value);
    }

    public String getCachedValue(String key) {
        return redisTemplate.opsForValue().get(key);
    }
}
```

## Migration from Redis to Infinispan

### Why Migrate

| Feature | Redis | Infinispan |
|---------|-------|------------|
| **Clustering** | Requires Redis Cluster setup, hash slots, client awareness | Built-in, transparent to clients |
| **Multiple databases in cluster** | Only database 0 | Full SELECT support in clustered mode |
| **Transactions** | No rollback | ACID with rollback |
| **Data distribution** | Manual sharding or Redis Cluster | Automatic with consistent hashing |
| **Failover** | Sentinel or Cluster | Built-in, automatic |
| **Persistence** | RDB/AOF | Pluggable stores (JDBC, RocksDB, remote) |
| **Kubernetes** | Manual or third-party operators | Official Infinispan Operator |
| **Multi-protocol** | Redis protocol only | RESP + Hot Rod + REST on same port |
| **Java ecosystem** | External dependency | Embedded mode available, native Quarkus/Spring integration |

### Migration Steps

1. **Install Infinispan Server** and create users
2. **Update client configuration**: Change host/port to Infinispan (default port 11222)
3. **Force RESP3**: All clients must use RESP3 protocol (critical difference)
4. **Test supported commands**: Verify your application uses only supported commands
5. **Review atomicity requirements**: If relying on Redis single-threaded atomicity, add `MULTI...EXEC` blocks or enable transactions
6. **Map databases**: Configure cache aliases for any `SELECT` usage beyond database 0
7. **Update monitoring**: Switch from Redis INFO metrics to Infinispan Console/REST metrics

### Migration Checklist

- [ ] Client configured for RESP3 (not RESP2)
- [ ] Authentication configured with Infinispan users
- [ ] Port updated to 11222 (or custom)
- [ ] Verified all Redis commands used are in the supported list
- [ ] Reviewed MULTI/EXEC usage for transaction behavior differences
- [ ] Lua scripts tested (no script caching, RESP3 serialization)
- [ ] Database SELECT usage mapped to cache aliases
- [ ] Cluster topology handling reviewed (no MOVED errors with Infinispan)

## Common Pitfalls

1. **RESP2 clients fail to connect**: Infinispan only supports RESP3. Always configure `protocolVersion(ProtocolVersion.RESP3)` or equivalent.

2. **Missing RESPHashFunctionPartitioner**: Custom cache configurations must use `org.infinispan.distribution.ch.impl.RESPHashFunctionPartitioner` as the key partitioner. Server will refuse to start otherwise.

3. **Key encoding must be octet-stream**: Using any other key encoding (like `application/x-protostream`) will cause the server to reject the cache configuration.

4. **Assuming single-threaded atomicity**: Unlike Redis, multi-key operations are not atomic by default. Use `MULTI...EXEC` with transactions enabled for atomic behavior.

5. **FLUSHALL scope**: `FLUSHALL` only flushes the current database (behaves like `FLUSHDB`), not all databases.

6. **SMOVE is not atomic**: An element may briefly disappear from both source and destination sets during concurrent operations.

7. **Lua script caching**: Scripts are not cached between invocations. If performance-sensitive, consider alternative approaches.

8. **SCAN cursor timeout**: Cursors expire after 5 minutes of inactivity. Long-running scans may lose their cursor.

9. **Port confusion**: Infinispan uses port 11222 by default (not 6379). All protocols (REST, Hot Rod, RESP) share this single port.

10. **JSON RegEx differences**: JSON filter regex uses Java-style patterns. Redis-style dynamic filters like `$..book[?(@.name =~ $.pattern)]` are not supported.
