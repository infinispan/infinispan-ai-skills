# Verification Checklist for Infinispan Hibernate 2LC

Use this checklist to verify that Infinispan second-level cache is correctly configured and functioning.

## Pre-Flight Checks

- [ ] Correct Maven dependency for Hibernate ORM version (`infinispan-hibernate-cache-v66` for Hibernate 6.6+/7.1+)
- [ ] `hibernate.cache.use_second_level_cache` is set to `true`
- [ ] `hibernate.cache.region.factory_class` is set to `infinispan`
- [ ] `shared-cache-mode` is set to `ENABLE_SELECTIVE` (or equivalent)
- [ ] Entities annotated with `@Cacheable` and `@Cache(usage = ...)`
- [ ] If using query cache: `hibernate.cache.use_query_cache` is set to `true`
- [ ] If single-node: `hibernate.cache.infinispan.cfg` points to local config
- [ ] If WildFly: `hibernate.cache.infinispan.cfg` is **not** set

## Runtime Verification

### Enable Statistics

```properties
hibernate.generate_statistics=true
```

### Check Cache Hit/Miss Ratios

```java
Statistics stats = sessionFactory.getStatistics();
CacheRegionStatistics cacheStats = stats.getDomainDataRegionStatistics("com.example.Product");

long hitCount = cacheStats.getHitCount();
long missCount = cacheStats.getMissCount();
long putCount = cacheStats.getPutCount();

System.out.println("Hit ratio: " + (double) hitCount / (hitCount + missCount));
```

### Verify Entity Is Being Cached

1. Load an entity by ID (first access = cache miss + DB query + cache put)
2. Load the same entity again (should be cache hit, no DB query)
3. Check statistics: `hitCount` should increment on second load

### Verify Query Cache Is Working

1. Execute a cacheable query (first execution = DB query + cache put)
2. Execute the same query again (should be cache hit)
3. Modify an entity of the queried type
4. Execute the query again (should be cache miss due to invalidation)

## Concurrency Strategy Checks

- [ ] `NONSTRICT_READ_WRITE` entities have `@Version` field
- [ ] `NONSTRICT_READ_WRITE` is not used on `@NaturalId` attributes
- [ ] If using distributed/replicated caches: eviction is disabled
- [ ] If using distributed/replicated caches: only expiration (with long max-idle) is used

## Cluster Checks

- [ ] Timestamps cache is replicated (not local or invalidation)
- [ ] Wall clocks are synchronized across nodes (NTP)
- [ ] If using `NONSTRICT_READ_WRITE`: stale reads between DB commit and 2LC update are acceptable for the use case
- [ ] Entity/collection caches use invalidation mode unless there is a specific reason for replication/distribution

## Common Issues to Watch For

| Symptom | Likely Cause | Diagnostic |
|---------|-------------|-----------|
| 0% hit ratio | Entity not annotated with `@Cacheable` | Check entity annotations |
| Hit ratio but no performance gain | Cache is too small, frequent eviction | Increase `max_entries`, check eviction stats |
| Stale data in cluster | Async timestamps replication | Consider sync replication for timestamps |
| `ClassNotFoundException` on cache start | Wrong Infinispan cache provider artifact | Verify Maven dependency matches Hibernate version |
| Query cache always misses | Entities modified frequently invalidate all query results | Consider disabling query cache for volatile entity types |
| `LazyInitializationException` | Not a 2LC issue -- session closed before accessing lazy collection | Fetch eagerly or use `@Cache` on the collection with an open session |

## JMX Monitoring

Enable Infinispan JMX statistics:

```xml
<property name="hibernate.cache.infinispan.statistics" value="true"/>
```

This exposes Infinispan cache metrics via JMX MBeans for monitoring with tools like JConsole, VisualVM, or Prometheus JMX exporter.
