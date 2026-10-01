# Verification Checklist for Infinispan Listeners

Use this checklist to verify listener implementations are correct and production-ready.

## Embedded Cache Listener Checks

- [ ] Listener class is annotated with `@Listener`
- [ ] Listener methods are annotated with the correct event annotation (`@CacheEntryCreated`, `@CacheEntryModified`, `@CacheEntryRemoved`, `@CacheEntryExpired`)
- [ ] Listener method parameter type matches the annotation (e.g., `CacheEntryCreatedEvent` for `@CacheEntryCreated`)
- [ ] Listener is registered on the correct cache or cache manager instance
- [ ] For non-blocking listeners: method returns `CompletionStage<Void>`, not `void`
- [ ] For async listeners: `@Listener(sync = false)` is set
- [ ] Listener methods handle exceptions internally (sync listeners that throw can abort cache operations)

## Clustered Listener Checks

- [ ] `@Listener(clustered = true)` is set
- [ ] Only supported event types are used: `@CacheEntryCreated`, `@CacheEntryModified`, `@CacheEntryRemoved`, `@CacheEntryExpired`
- [ ] Code does not depend on pre-events (only post-events are delivered for clustered listeners)
- [ ] If `includeCurrentState = true` is used, the listener handles the initial flood of `CacheEntryCreated` events correctly

## Cache Manager Listener Checks

- [ ] Listener is registered on the `CacheManager`, not on a `Cache`
- [ ] Correct event annotations are used: `@CacheStarted`, `@CacheStopped`, `@ViewChanged`, `@Merged`

## Hot Rod Client Listener Checks

- [ ] Listener class is annotated with `@ClientListener` (not `@Listener`)
- [ ] Correct client event annotations are used: `@ClientCacheEntryCreated`, `@ClientCacheEntryModified`, `@ClientCacheEntryRemoved`
- [ ] Listener is registered via `cache.addClientListener(listener)` on a `RemoteCache`
- [ ] Listener is removed when no longer needed via `cache.removeClientListener(listener)`
- [ ] If using `includeCurrentState = true`, the listener handles initial state replay and failover correctly
- [ ] `@ClientCacheFailover` handler is implemented if listener state needs to survive server failover

## Server-Side Filter/Converter Checks

- [ ] Filter/converter factory is annotated with `@NamedFactory(name = "...")` with a unique name
- [ ] Filter/converter classes implement `Serializable` (required for clustered deployment)
- [ ] For ProtoStream: custom event types are annotated with `@Proto`
- [ ] `META-INF/services/` file is created with the correct factory interface and fully qualified class name
- [ ] JAR is deployed to `server/lib` directory
- [ ] `@ClientListener` annotation references the correct `filterFactoryName` and/or `converterFactoryName`
- [ ] Combined filter/converter uses the same name for both `filterFactoryName` and `converterFactoryName`

## Performance Checks

- [ ] Sync listeners do not perform heavy computation or blocking I/O (use non-blocking or async instead)
- [ ] Server-side filters are deployed to reduce network traffic (rather than filtering on the client)
- [ ] Duplicate events are handled idempotently (check `isCommandRetried()` in non-transactional caches)
- [ ] Number of registered client listeners is reasonable (each listener has a server-side event queue)
- [ ] Backpressure thresholds are reviewed if high event throughput is expected
- [ ] Continuous queries are considered as an alternative when indexed caches are already in use

## CDI Integration Checks

- [ ] Infinispan CDI module is on the classpath
- [ ] Observer methods use `@Observes` with the correct event type
- [ ] Observer method parameter matches the exact event class (e.g., `CacheEntryCreatedEvent`, `CacheStartedEvent`)
