# Infinispan Hibernate 2LC - References

## Official Documentation

- [Infinispan Hibernate Cache Guide](https://infinispan.org/docs/stable/titles/hibernate/hibernate.html)
- [Hibernate ORM Second-Level Caching](https://docs.jboss.org/hibernate/orm/6.6/introduction/html_single/Hibernate_Introduction.html#second-level-cache)

## Default Configuration Files (on classpath)

These files are included in the Infinispan Hibernate cache provider JAR and can be referenced in `hibernate.cache.infinispan.cfg`:

- `org/infinispan/hibernate/cache/commons/builder/infinispan-configs.xml` — default clustered configuration
- `org/infinispan/hibernate/cache/commons/builder/infinispan-configs-local.xml` — local (single-node) configuration

## Tutorials

- [Standalone local Hibernate cache](https://github.com/infinispan/infinispan-simple-tutorials/tree/main/hibernate-cache/local)
- [Spring local Hibernate cache](https://github.com/infinispan/infinispan-simple-tutorials/tree/main/hibernate-cache/spring-local)
- [WildFly local Hibernate cache](https://github.com/infinispan/infinispan-simple-tutorials/tree/main/hibernate-cache/wildfly-local)

## Version Compatibility Matrix

| Hibernate ORM Version | Infinispan Artifact | Infinispan Version | Namespace |
|----------------------|--------------------|--------------------|-----------|
| 7.1+ | `infinispan-hibernate-cache-v66` | 16.x | `jakarta` |
| 6.6 | `infinispan-hibernate-cache-v66` | 16.x | `jakarta` |
| 6.2 - 6.5 | `infinispan-hibernate-cache-v62` | 15.x | `jakarta` |
| 6.0 | `infinispan-hibernate-cache-v60` | 14.x | `jakarta` |
| 5.3 | `infinispan-hibernate-cache-v53` | 13.x | `javax` |

## Key Configuration Properties

| Property | Description |
|----------|-------------|
| `hibernate.cache.use_second_level_cache` | Enable/disable 2LC |
| `hibernate.cache.region.factory_class` | Set to `infinispan` |
| `hibernate.cache.use_query_cache` | Enable/disable query caching |
| `hibernate.cache.infinispan.cfg` | Path to custom Infinispan XML configuration |
| `hibernate.cache.infinispan.<type>.cfg` | Cache template for a data type (entity, collection, query, timestamps) |
| `hibernate.cache.infinispan.<type>.eviction.strategy` | Eviction algorithm: NONE, LRU, LIRS |
| `hibernate.cache.infinispan.<type>.eviction.max_entries` | Max cache entries |
| `hibernate.cache.infinispan.<type>.expiration.lifespan` | Entry lifespan in ms |
| `hibernate.cache.infinispan.<type>.expiration.max_idle` | Max idle time in ms |
| `hibernate.cache.infinispan.statistics` | Enable Infinispan statistics and JMX |
| `hibernate.cache.use_minimal_puts` | Skip cache update if entry exists (off by default) |

## Provenance

This skill targets **Infinispan 16.x** and **Hibernate ORM 6.6+**. Content was derived from the official Infinispan documentation and verified against the source repository.
