# Infinispan Spring Integration Verification Checklist

Use this checklist to verify that an Infinispan Spring integration is correctly configured.

## Dependency Check

- [ ] Correct starter artifact matches Spring Boot version (3.x uses `spring-boot3-starter-*`, 4.x uses `spring-boot4-starter-*`)
- [ ] Only one mode is on the classpath (remote OR embedded, not both, unless intentional)
- [ ] Spring Session dependencies include `spring-session-core` and `spring-web` if using session externalization
- [ ] Actuator dependency is present if metrics/statistics are needed

## Configuration Check

- [ ] `infinispan.remote.server-list` is set for remote mode
- [ ] `@EnableCaching` annotation is present on a configuration class
- [ ] `@EnableInfinispanRemoteHttpSession` or `@EnableInfinispanEmbeddedHttpSession` is present if using Spring Session
- [ ] Reactive mode is enabled (`infinispan.remote.reactive=true` or `infinispan.embedded.reactive=true`) if using WebFlux
- [ ] No conflicting properties between `hotrod-client.properties` and `application.properties` (hotrod-client.properties takes priority)

## Serialization Check

- [ ] ProtoStream is used for remote caches (recommended over Java Serialization)
- [ ] Model classes are annotated with `@Proto` and fields with `@ProtoField`
- [ ] A `@ProtoSchema` interface extending `GeneratedSchema` exists
- [ ] Server cache encoding matches client marshaller (both `application/x-protostream` for ProtoStream)
- [ ] If using Java Serialization, classes are added to `infinispan.remote.java-serial-allowlist`

## Cache Setup Check

- [ ] Required caches exist on the server (Infinispan does not auto-create caches for Spring Cache annotations)
- [ ] Session cache exists if using Spring Session (default name: `sessions`)
- [ ] Per-cache configuration uses correct property format (bracket notation for dotted cache names)
- [ ] Cache template exists on server if `template-name` is referenced in per-cache config

## Spring Boot 4 Migration Check

- [ ] `InfinispanConfigurationCustomizer` usage is replaced with `InfinispanCacheConfigurer` (removed in Boot 4)
- [ ] Default cache is explicitly defined via `InfinispanCacheConfigurer` if previously relying on auto-created default

## Runtime Verification

- [ ] Application starts without `CacheNotFoundException` or connection errors
- [ ] `@Cacheable` methods return cached values on subsequent calls (verify with logging or debug)
- [ ] Session data persists across application restarts (for Spring Session)
- [ ] Actuator endpoint `/actuator/caches` shows registered caches (if actuator is enabled)
- [ ] No thread blocking warnings in logs when reactive mode is expected
