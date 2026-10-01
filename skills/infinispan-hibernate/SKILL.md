---
name: infinispan-hibernate
description: Use when user needs help with Infinispan as Hibernate second-level cache - entity cache, collection cache, query cache, configuration, eviction strategies, JPA integration.
---

# Infinispan as Hibernate Second-Level Cache

Help users integrate Infinispan as the Hibernate ORM second-level cache (2LC) provider. Covers entity/collection/query caching, concurrency strategies, cluster configuration, and performance tuning.

## Workflow

1. Determine the user's Hibernate ORM version and deployment scenario (standalone, Spring, WildFly, single-node vs. clustered)
2. Guide them through dependency setup and configuration
3. Explain cache region types and concurrency strategies
4. Provide ready-to-use configuration and annotation examples
5. Warn about common pitfalls (stale reads, eviction in replicated mode, timestamps cache)

## Version Compatibility

| Hibernate ORM | Infinispan Artifact | Infinispan Version |
|---------------|--------------------|--------------------|
| 6.6, 7.1+ | `infinispan-hibernate-cache-v66` | 16.x (latest) |
| 6.2 - 6.5 | `infinispan-hibernate-cache-v62` | 15.x |
| 6.0 | `infinispan-hibernate-cache-v60` | 14.x |
| 5.3 | `infinispan-hibernate-cache-v53` | 13.x |

Default to **Infinispan 16.x** with **Hibernate 6.6+** unless the user specifies otherwise. Hibernate 6.x uses the `jakarta` namespace; Hibernate 5.3 uses the `javax` namespace.

## Maven Dependency

For Hibernate ORM 6.6 / 7.1+ with Infinispan 16.x:

```xml
<dependency>
    <groupId>org.infinispan</groupId>
    <artifactId>infinispan-hibernate-cache-v66</artifactId>
    <version>${version.infinispan}</version>
</dependency>
```

## Enabling the Second-Level Cache

### JPA (persistence.xml)

```xml
<!-- Enable 2LC -->
<property name="hibernate.cache.use_second_level_cache" value="true"/>

<!-- Set Infinispan as region factory -->
<property name="hibernate.cache.region.factory_class" value="infinispan"/>

<!-- Select which entities to cache -->
<shared-cache-mode>ENABLE_SELECTIVE</shared-cache-mode>

<!-- Optional: enable query cache -->
<property name="hibernate.cache.use_query_cache" value="true"/>

<!-- Optional: enable statistics for verification -->
<property name="hibernate.generate_statistics" value="true"/>
```

### Spring Boot (application.properties)

```properties
# Enable 2LC
spring.jpa.properties.hibernate.cache.use_second_level_cache=true

# Set Infinispan as region factory
spring.jpa.properties.hibernate.cache.region.factory_class=infinispan

# Select which entities to cache
spring.jpa.properties.jakarta.persistence.sharedCache.mode=ENABLE_SELECTIVE

# Optional: enable query cache
spring.jpa.properties.hibernate.cache.use_query_cache=true

# Optional: enable statistics
spring.jpa.properties.hibernate.generate_statistics=true
```

## Single-Node vs. Clustered Configuration

By default, the Infinispan 2LC provider loads a **clustered** configuration. For single-node deployments, explicitly set the local configuration:

### Single-Node (Local)

```xml
<!-- JPA persistence.xml -->
<property name="hibernate.cache.infinispan.cfg"
    value="org/infinispan/hibernate/cache/commons/builder/infinispan-configs-local.xml"/>
```

```properties
# Spring application.properties
spring.jpa.properties.hibernate.cache.infinispan.cfg=org/infinispan/hibernate/cache/commons/builder/infinispan-configs-local.xml
```

### Multi-Node (Clustered)

No extra configuration needed -- the default Infinispan configuration is designed for clustered environments. Just set the region factory:

```xml
<property name="hibernate.cache.region.factory_class" value="infinispan"/>
```

### WildFly

In WildFly, Infinispan is the **default** 2LC provider. Do **not** set `hibernate.cache.infinispan.cfg` -- cache configuration comes from WildFly's `standalone.xml` or `standalone-ha.xml`.

## Cache Region Types

Infinispan manages four types of cache regions:

| Region Type | What It Stores | Default Cluster Mode |
|-------------|---------------|---------------------|
| **Entity** | Entity instances indexed by `@Id` or `@EmbeddedId` | Synchronous invalidation (local storage, invalidate on update) |
| **Collection** | Collection associations | Synchronous invalidation |
| **Query** | Query results (query -> result mapping) | Local only (not replicated by default) |
| **Timestamps** | Entity type -> last modification time | Asynchronous replication (all nodes must have all timestamps) |

Additional internal region types:
- `immutable-entity`: Entities annotated with `@Immutable`
- `naturalid`: Entities indexed by `@NaturalId`
- `pending-puts`: Auxiliary caches for invalidation mode

## JPA Annotations for Caching

### Mark an Entity as Cacheable

```java
import jakarta.persistence.Cacheable;
import jakarta.persistence.Entity;
import org.hibernate.annotations.Cache;
import org.hibernate.annotations.CacheConcurrencyStrategy;

@Entity
@Cacheable
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Product {
    @Id
    private Long id;
    private String name;
    private BigDecimal price;
    // ...
}
```

### Cache a Collection

```java
@Entity
@Cacheable
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Category {
    @Id
    private Long id;

    @OneToMany(mappedBy = "category")
    @Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
    private Set<Product> products;
}
```

### Cache a Query

```java
// JPA query caching (requires hibernate.cache.use_query_cache=true)
List<Product> products = entityManager
    .createQuery("SELECT p FROM Product p WHERE p.price < :maxPrice", Product.class)
    .setParameter("maxPrice", new BigDecimal("100"))
    .setHint("org.hibernate.cacheable", Boolean.TRUE)
    .getResultList();
```

## Cache Concurrency Strategies

| Strategy | Description | Cache Mode | Eviction | Notes |
|----------|-------------|-----------|----------|-------|
| `READ_ONLY` | Immutable data, never updated | Invalidation, replicated, or distributed | Allowed with invalidation | Simplest, best for reference data |
| `NONSTRICT_READ_WRITE` | Rarely updated, slight staleness acceptable | Distributed or replicated (non-transactional) | **Not allowed** | Requires entity versioning (`@Version`). Cannot cache `@NaturalId`. Fewer RPCs, better performance. |
| `READ_WRITE` | Frequently read, sometimes updated | Invalidation (allows eviction) or distributed/replicated (no eviction) | Depends on cache mode | Recommended default. Same guarantees as `TRANSACTIONAL` in Hibernate 6.x. |
| `TRANSACTIONAL` | JTA transactional consistency | Invalidation (Hibernate <= 5.2: transactional cache; >= 5.3: same as `READ_WRITE`) | Allowed with invalidation | Only in JTA environments. In Hibernate 6.x, implementation is identical to `READ_WRITE`. |

### Compatibility Table

| Concurrency Strategy | Cache Transactions | Cache Mode | Eviction Allowed |
|---------------------|-------------------|------------|-----------------|
| `READ_WRITE` | Non-transactional | Invalidation | Yes |
| `READ_WRITE` | Non-transactional | Distributed/Replicated | **No** |
| `NONSTRICT_READ_WRITE` | Non-transactional | Distributed/Replicated | **No** |
| `TRANSACTIONAL` | Non-transactional (6.x) | Invalidation | Yes |

## Per-Region Configuration

Override cache settings for specific data types or individual entities/collections:

### By Data Type

```xml
<!-- Use a custom Infinispan cache template for all entities -->
<property name="hibernate.cache.infinispan.entity.cfg" value="custom-entities"/>

<!-- Use a custom template for query cache -->
<property name="hibernate.cache.infinispan.query.cfg" value="custom-query-cache"/>
```

### By Specific Entity or Collection

```xml
<!-- Custom cache for a specific entity -->
<property name="hibernate.cache.infinispan.com.example.Product.cfg"
    value="product-cache"/>

<!-- Custom cache for a specific collection -->
<property name="hibernate.cache.infinispan.com.example.Category.products.cfg"
    value="category-products-cache"/>
```

### Override Eviction/Expiration via Properties

```xml
<!-- Global entity eviction settings -->
<property name="hibernate.cache.infinispan.entity.eviction.strategy" value="LRU"/>
<property name="hibernate.cache.infinispan.entity.eviction.max_entries" value="5000"/>
<property name="hibernate.cache.infinispan.entity.eviction.wake_up_interval" value="2000"/>
<property name="hibernate.cache.infinispan.entity.expiration.lifespan" value="60000"/>
<property name="hibernate.cache.infinispan.entity.expiration.max_idle" value="30000"/>

<!-- Per-entity eviction override -->
<property name="hibernate.cache.infinispan.com.example.Product.eviction.strategy" value="LIRS"/>
```

### Configuration Property Reference

| Property Suffix | Values | Description |
|----------------|--------|-------------|
| `.eviction.strategy` | `NONE`, `LRU`, `LIRS` | Eviction algorithm |
| `.eviction.max_entries` | integer | Maximum cache entries |
| `.eviction.wake_up_interval` | milliseconds | Eviction thread check interval |
| `.expiration.lifespan` | milliseconds | Time from insert until expiry |
| `.expiration.max_idle` | milliseconds | Time from last access until expiry |
| `hibernate.cache.infinispan.statistics` | `true`/`false` | Enable Infinispan statistics and JMX exposure |

### Default Eviction Settings

Entities, collections, and queries share these defaults:
- Eviction wake-up interval: 5 seconds
- Max entries: 10,000
- Max idle: 100 seconds
- Algorithm: LRU

**Timestamps cache has no eviction/expiration -- this is by design and must not be changed.**

## Custom Infinispan Configuration File

Provide your own Infinispan XML configuration:

```xml
<property name="hibernate.cache.infinispan.cfg" value="my-infinispan-config.xml"/>
```

Caches not defined in your custom file will fall back to the built-in defaults (clustered or local depending on the configuration type).

## Stale Read Prevention

### Timestamps Cache and Query Consistency

The timestamps cache maps entity types to their last modification time. After loading a cached query result, Hibernate compares the result timestamp against the timestamps of all referenced entity types. If any entity type was modified more recently, the cached query result is discarded and the query re-executes against the database.

**This requires synchronized wall clocks across cluster nodes.**

### Stale Read Scenario with `NONSTRICT_READ_WRITE`

Between DB commit and transaction commit completion, a stale read can occur:

```
A=0 (non-cached), B=0 (cached in 2LC)
TX1: write A = 1, write B = 1
TX1: start commit
TX1: commit A, B in DB
TX2: read A = 1 (from DB), read B = 0 (from 2LC)  // stale!
TX1: update A, B in 2LC
TX1: end commit
TX3: read A = 1, B = 1  // consistent after TX1 completes
```

**Mitigation**: Use `READ_WRITE` strategy with invalidation caches for strict consistency. Only use `NONSTRICT_READ_WRITE` when brief staleness is acceptable and you need the performance benefit (fewer RPCs).

### Asynchronous Replication and Timestamps

The default timestamps cache uses asynchronous replication for performance. This means stale query results can briefly appear even on the **same node** that performed an update. If this is unacceptable, configure synchronous replication for the timestamps cache (at a performance cost).

## Cluster Mode Considerations

### Invalidation (Default for Entities/Collections)

- On read from DB: data cached **locally only** (reduces intra-cluster traffic)
- On update: invalidation message sent to all nodes (synchronous)
- Nodes remove invalidated entries from their local cache
- **Best for**: most use cases, especially with shared database

### Replicated / Distributed Caches

- Can be configured for entities/collections instead of invalidation
- On read from DB: data propagated to other nodes **asynchronously**
- Concurrent database loads: one succeeds, others fail silently (all loading same data)
- Cache may briefly be out of date after a JPA call that triggers DB load
- **Eviction must not be used** -- can cause consistency issues
- Expiration (with reasonably long max-idle) is acceptable

### Minimal Puts Optimization

```xml
<property name="hibernate.cache.use_minimal_puts" value="true"/>
```

- **Off by default** in Infinispan implementation
- With **invalidation caches**: keep off (put-from-load is local and silently fails if locked)
- With **replicated/distributed caches**: consider enabling to avoid redundant remote updates

## Performance Tuning

1. **Enable statistics first** to verify cache is working:
   ```properties
   hibernate.generate_statistics=true
   ```
   Check hit/miss ratios before tuning.

2. **Right-size max entries** per entity type based on working set size. The default 10,000 may be too small or too large.

3. **Tune expiration** based on data volatility:
   - Reference data (countries, currencies): long lifespan, `READ_ONLY`
   - Session data: short max-idle
   - Frequently updated: short lifespan or rely on invalidation

4. **Choose the right concurrency strategy**:
   - `READ_ONLY` for immutable reference data
   - `READ_WRITE` for general mutable entities (recommended default)
   - `NONSTRICT_READ_WRITE` only when staleness is acceptable and performance is critical

5. **Query cache**: enable selectively. Only cache queries that are expensive and whose results are rarely invalidated. Remember that any modification to an entity type invalidates **all** query results referencing that type.

6. **Avoid eviction with replicated/distributed caches** -- use expiration with long max-idle instead.

## Common Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| Forgetting `@Cacheable` on entity | Entity not cached despite 2LC being enabled | Add `@Cacheable` and set `ENABLE_SELECTIVE` shared cache mode |
| Setting `hibernate.cache.infinispan.cfg` in WildFly | Conflicts with WildFly's built-in Infinispan configuration | Remove the property; configure caches in `standalone.xml` |
| Using eviction with distributed/replicated caches | Consistency issues -- evicted entries may not be reloaded correctly | Disable eviction; use expiration with long max-idle instead |
| Using `NONSTRICT_READ_WRITE` without `@Version` | Required for optimistic concurrency in this mode | Add `@Version` field to entity |
| Using `NONSTRICT_READ_WRITE` on `@NaturalId` | Natural IDs are never versioned | Use `READ_WRITE` for natural ID caching |
| Evicting/expiring timestamps cache | Breaks query cache invalidation | Never configure eviction or expiration on timestamps cache |
| Not enabling query cache at persistence unit level | `setHint("org.hibernate.cacheable", true)` has no effect | Set `hibernate.cache.use_query_cache=true` |
| Using local/invalidation mode for timestamps cache | All nodes must have all timestamps | Use replicated mode (default is async replication) |
| Unsynchronized clocks in cluster | Query cache returns stale results | Synchronize wall clocks (NTP) across all cluster nodes |
| Using default clustered config for single-node | Unnecessary overhead | Set `hibernate.cache.infinispan.cfg` to `infinispan-configs-local.xml` |

## Complete Example: Spring Boot + Infinispan 2LC

### application.properties

```properties
# Datasource
spring.datasource.url=jdbc:postgresql://localhost:5432/mydb
spring.datasource.username=user
spring.datasource.password=pass

# Hibernate
spring.jpa.hibernate.ddl-auto=validate

# Infinispan 2LC
spring.jpa.properties.hibernate.cache.use_second_level_cache=true
spring.jpa.properties.hibernate.cache.region.factory_class=infinispan
spring.jpa.properties.jakarta.persistence.sharedCache.mode=ENABLE_SELECTIVE
spring.jpa.properties.hibernate.cache.use_query_cache=true
spring.jpa.properties.hibernate.generate_statistics=true

# Single-node: use local config
spring.jpa.properties.hibernate.cache.infinispan.cfg=org/infinispan/hibernate/cache/commons/builder/infinispan-configs-local.xml
```

### Entity

```java
@Entity
@Cacheable
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Version
    private int version;

    private String name;
    private BigDecimal price;

    @ManyToOne
    @JoinColumn(name = "category_id")
    private Category category;
}
```

### Repository with Cached Query

```java
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {

    @QueryHints(@QueryHint(name = "org.hibernate.cacheable", value = "true"))
    List<Product> findByPriceLessThan(BigDecimal maxPrice);
}
```

## Tutorials and References

- [Standalone local tutorial](https://github.com/infinispan/infinispan-simple-tutorials/tree/main/hibernate-cache/local)
- [Spring local tutorial](https://github.com/infinispan/infinispan-simple-tutorials/tree/main/hibernate-cache/spring-local)
- [WildFly local tutorial](https://github.com/infinispan/infinispan-simple-tutorials/tree/main/hibernate-cache/wildfly-local)
- [Hibernate ORM 2LC documentation](https://docs.jboss.org/hibernate/orm/6.6/introduction/html_single/Hibernate_Introduction.html#second-level-cache)
- [Infinispan Hibernate Cache documentation](https://infinispan.org/docs/stable/titles/hibernate/hibernate.html)
