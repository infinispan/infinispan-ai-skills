# Infinispan Listeners and Events - References

## Official Documentation

- Listeners and notifications: https://infinispan.org/docs/stable/titles/developing/developing.html#listeners-notifications
- Hot Rod Java client: https://infinispan.org/docs/stable/titles/hotrod_java/hotrod_java.html
- Performance tuning: https://infinispan.org/docs/stable/titles/tuning/tuning.html

## Key API Packages

- `org.infinispan.notifications.Listener` — Main listener annotation
- `org.infinispan.notifications.Listenable` — Interface for registering listeners (implemented by Cache and CacheManager)
- `org.infinispan.notifications.cachelistener.annotation` — Cache-level event annotations (`@CacheEntryCreated`, `@CacheEntryModified`, `@CacheEntryRemoved`, `@CacheEntryExpired`)
- `org.infinispan.notifications.cachemanagerlistener.annotation` — Cache manager event annotations (`@CacheStarted`, `@CacheStopped`, `@ViewChanged`, `@Merged`)
- `org.infinispan.notifications.cachelistener.filter` — `CacheEventFilter`, `CacheEventConverter`, `CacheEventFilterConverter`
- `org.infinispan.filter.KeyFilter` — Simple key-based filtering
- `org.infinispan.filter.NamedFactory` — Annotation for naming filter/converter factories
- `org.infinispan.client.hotrod.annotation` — Hot Rod client listener annotations (`@ClientListener`, `@ClientCacheEntryCreated`, `@ClientCacheEntryModified`, `@ClientCacheEntryRemoved`, `@ClientCacheFailover`)
- `org.infinispan.client.hotrod.event` — Hot Rod client event classes

## Server-Side Deployment (Service Loader Files)

When deploying filters/converters to Infinispan Server, create the appropriate service file under `META-INF/services/`:

| Factory Type | Service File |
|-------------|-------------|
| `CacheEventFilterFactory` | `META-INF/services/org.infinispan.notifications.cachelistener.filter.CacheEventFilterFactory` |
| `CacheEventConverterFactory` | `META-INF/services/org.infinispan.notifications.cachelistener.filter.CacheEventConverterFactory` |
| `CacheEventFilterConverterFactory` | `META-INF/services/org.infinispan.notifications.cachelistener.filter.CacheEventFilterConverterFactory` |

## Provenance

This skill targets **Infinispan 16.x**. Content was derived from the official Infinispan documentation and verified against the source repository.
