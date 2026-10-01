# Infinispan Spring Integration - References

## Official Documentation

- Spring Boot Starter guide: https://infinispan.org/docs/stable/titles/spring_boot/starter.html
- Spring Cache provider: https://infinispan.org/docs/stable/titles/spring/spring.html
- Hot Rod client configuration: https://infinispan.org/docs/stable/titles/hotrod_java/hotrod_java.html
- Encoding and marshalling: https://infinispan.org/docs/stable/titles/encoding/encoding.html

## Spring Framework References

- Spring Cache Abstraction: https://docs.spring.io/spring-framework/reference/integration/cache.html
- Spring Session: https://docs.spring.io/spring-session/reference/
- Spring Boot Actuator: https://docs.spring.io/spring-boot/reference/actuator/

## Key Interfaces and Classes

- `InfinispanRemoteConfigurer` — Customize the remote `Configuration`
- `InfinispanRemoteCacheCustomizer` — Customize individual remote cache configurations
- `InfinispanGlobalConfigurer` — Customize the embedded `GlobalConfiguration`
- `InfinispanCacheConfigurer` — Customize individual embedded cache configurations
- `InfinispanGlobalConfigurationCustomizer` — Fine-tune embedded global config (Boot 4+)
- `@EnableInfinispanRemoteHttpSession` — Enable remote Spring Session
- `@EnableInfinispanEmbeddedHttpSession` — Enable embedded Spring Session

## Tutorials

- Spring Boot remote: https://github.com/infinispan/infinispan-simple-tutorials/tree/main/spring-boot/remote
- Spring Boot embedded: https://github.com/infinispan/infinispan-simple-tutorials/tree/main/spring-boot/embedded
- Spring Session: https://github.com/infinispan/infinispan-simple-tutorials/tree/main/spring-session

## Provenance

This skill targets **Infinispan 16.x** with Spring Boot 3.x/4.x and Spring 6/7. Content was derived from the official Infinispan documentation and verified against the source repository.
