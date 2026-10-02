# RESP Endpoint Verification Checks

Use these checks to verify advice given about the Infinispan RESP endpoint.

## Configuration Checks

- [ ] RESP cache key encoding is `application/octet-stream` (mandatory)
- [ ] Clustered caches use `org.infinispan.distribution.ch.impl.RESPHashFunctionPartitioner`
- [ ] Cache mode matches deployment: `local-cache` for standalone, `distributed-cache` for clustered
- [ ] If transactions needed for MULTI/EXEC, cache has `<transaction mode="NON_XA" locking="PESSIMISTIC"/>`
- [ ] RESP connector references an existing, properly configured cache name
- [ ] Security realm is configured on the endpoint when authentication is required

## Client Configuration Checks

- [ ] Client is configured for **RESP3** protocol (not RESP2)
- [ ] Client connects to port **11222** (Infinispan default), not 6379 (Redis default)
- [ ] Authentication uses Infinispan users (created via CLI or user tool)
- [ ] Jedis uses `DefaultJedisClientConfig.builder().protocol(RESP3)`
- [ ] Lettuce uses `ClientOptions.builder().protocolVersion(ProtocolVersion.RESP3)`
- [ ] Spring Data Redis has a `LettuceClientConfigurationBuilderCustomizer` bean forcing RESP3

## Behavioral Checks

- [ ] Advice mentions RESP3-only requirement when discussing client connections
- [ ] Multi-key atomicity caveats are mentioned when discussing MSET, MGET, or similar
- [ ] Transaction differences (ACID with rollback vs. Redis no-rollback) are noted
- [ ] FLUSHALL scope limitation (current database only) is mentioned if relevant
- [ ] Lua script non-caching behavior is noted when discussing EVAL
- [ ] SELECT command support in clustered mode is highlighted as an advantage

## Migration Checks

- [ ] RESP3 protocol requirement is the first migration step
- [ ] Command compatibility is verified against the supported command list
- [ ] Atomicity model differences are called out
- [ ] Database mapping via cache aliases is explained for SELECT usage
- [ ] Port change from 6379 to 11222 is mentioned
