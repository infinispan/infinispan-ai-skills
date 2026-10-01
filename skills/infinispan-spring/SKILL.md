---
name: infinispan-spring
description: Use when user needs help with Infinispan Spring Boot integration - Spring Cache, Spring Session, remote/embedded starters, reactive support, per-cache configuration, serialization, actuator metrics.
---

# Infinispan Spring Integration Guide

Help users integrate Infinispan with Spring Boot, Spring Cache, and Spring Session. Covers both remote (Hot Rod client to Infinispan Server) and embedded (in-process cache) modes, Spring Boot 3.x and 4.x starters, reactive support, serialization, and per-cache configuration.

## Workflow

1. Determine the deployment mode: **remote** (client/server via Hot Rod) or **embedded** (in-process)
2. Determine the Spring Boot version: **3.x** (Spring 6) or **4.x** (Spring 7)
3. Identify the integration: Spring Cache, Spring Session, or direct cache access
4. Provide the correct Maven dependency, configuration, and code examples
5. Warn about common pitfalls (serialization, missing caches, upgrade breaks)

## Maven Dependencies

### Spring Boot Starters (recommended)

Use starters for auto-configuration of the cache manager bean:

| Spring Boot Version | Remote | Embedded |
|---------------------|--------|----------|
| **3.x** (Spring 6) | `infinispan-spring-boot3-starter-remote` | `infinispan-spring-boot3-starter-embedded` |
| **4.x** (Spring 7) | `infinispan-spring-boot4-starter-remote` | `infinispan-spring-boot4-starter-embedded` |

```xml
<!-- Spring Boot 3.x + Remote Infinispan Server -->
<dependency>
    <groupId>org.infinispan</groupId>
    <artifactId>infinispan-spring-boot3-starter-remote</artifactId>
</dependency>

<!-- Spring Boot 4.x + Embedded -->
<dependency>
    <groupId>org.infinispan</groupId>
    <artifactId>infinispan-spring-boot4-starter-embedded</artifactId>
</dependency>
```

### Spring Integration without Boot (manual bean config)

| Spring Version | Remote | Embedded |
|----------------|--------|----------|
| **6** | `infinispan-spring6-remote` | `infinispan-spring6-embedded` |
| **7** | `infinispan-spring7-remote` | `infinispan-spring7-embedded` |

```xml
<!-- Spring 7 + Remote -->
<dependency>
    <groupId>org.infinispan</groupId>
    <artifactId>infinispan-spring7-remote</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
</dependency>
```

## Setting Up the Cache Manager

### Remote Mode (with Spring Boot Starter)

The starter auto-creates a `RemoteCacheManager` bean. Configure the server address in `application.properties`:

```properties
infinispan.remote.server-list=127.0.0.1:11222
```

Inject it in your application:

```java
@Autowired
private RemoteCacheManager cacheManager;
```

### Embedded Mode (with Spring Boot Starter)

The starter auto-creates an `EmbeddedCacheManager` bean. Optionally point to a configuration file:

```properties
infinispan.embedded.config-xml=infinispan.xml
infinispan.embedded.cluster-name=my-cluster
```

Inject and use it:

```java
@Autowired
private EmbeddedCacheManager cacheManager;

public void useCache() {
    Cache<String, String> cache = cacheManager.getCache("myCache");
    cache.put("key", "value");
}
```

## Spring Cache Provider

Infinispan implements the Spring Cache SPI (`CacheManager` / `Cache` interfaces).

### Enabling Spring Cache

Add `@EnableCaching` to your configuration class. The starter automatically detects the cache manager type:

- If `EmbeddedCacheManager` bean exists, instantiates `SpringEmbeddedCacheManager`
- If `RemoteCacheManager` bean exists, instantiates `SpringRemoteCacheManager`

```java
@SpringBootApplication
@EnableCaching
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
```

### Using Cache Annotations

```java
@Service
public class BookService {

    // Cache the return value in the "books" cache, keyed by bookId
    @Cacheable(value = "books", key = "#bookId")
    public Book findBook(Integer bookId) {
        // expensive lookup
        return bookRepository.findById(bookId);
    }

    // Update the cached value
    @CachePut(value = "books", key = "#book.id")
    public Book updateBook(Book book) {
        return bookRepository.save(book);
    }

    // Evict a single entry from cache
    @CacheEvict(value = "books", key = "#bookId")
    public void deleteBook(Integer bookId) {
        bookRepository.deleteById(bookId);
    }

    // Evict all entries from cache
    @CacheEvict(value = "books", allEntries = true)
    public void clearAllBooks() {
        // cache cleared
    }
}
```

### Cache Operation Timeouts

By default, Spring Cache operations are synchronous with no timeout. Configure timeouts via properties:

| Property | Description | Default |
|----------|-------------|---------|
| `infinispan.spring.operation.read.timeout` | Read timeout in milliseconds | `0` (no timeout) |
| `infinispan.spring.operation.write.timeout` | Write timeout in milliseconds | `0` (no timeout) |

## Spring Session

Infinispan can externalize HTTP sessions using Spring Session.

### Setup

1. Add the starter and Spring Session to your classpath.
2. Enable session support with the appropriate annotation.

**Important:** Infinispan does not provide a default session cache. You must create the cache first.

```java
@EnableCaching
@EnableInfinispanRemoteHttpSession   // For remote caches
// OR @EnableInfinispanEmbeddedHttpSession  // For embedded caches
@Configuration
public class SessionConfig {
}
```

### Session Annotation Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `maxInactiveIntervalInSeconds` | Session expiration time in seconds | `1800` (30 min) |
| `cacheName` | Name of the cache that stores sessions | `sessions` |

### Spring Session Dependencies (without Boot starter)

For remote caches with Spring 6:

```xml
<dependencies>
    <dependency>
        <groupId>org.infinispan</groupId>
        <artifactId>infinispan-core</artifactId>
    </dependency>
    <dependency>
        <groupId>org.infinispan</groupId>
        <artifactId>infinispan-spring6-remote</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.session</groupId>
        <artifactId>spring-session-core</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-web</artifactId>
    </dependency>
</dependencies>
```

## Per-Cache Configuration via Properties

Configure individual remote caches in `application.properties` or `application.yaml` to create caches on first access from the client.

### application.properties

```properties
# Create a cache with an inline XML configuration
infinispan.remote.cache.example.configuration=<distributed-cache/>
infinispan.remote.cache.example.force-return-values=true

# Create a cache using a server-side template
infinispan.remote.cache.mycache.template-name=myTemplate

# Per-cache marshaller override
infinispan.remote.cache.mycache.marshaller=org.infinispan.commons.marshall.JavaSerializationMarshaller

# Wildcard matching for multiple caches
infinispan.remote.cache.[com.myproject.*].template-name=myCustomTemplate
infinispan.remote.cache.[com.myproject.*].transaction.transaction-mode=NON_XA

# Near-cache for a specific cache
infinispan.remote.cache.[com.myproject.mycache].near-cache.mode=INVALIDATED
```

### application.yaml

```yaml
infinispan:
  remote:
    server-list: "127.0.0.1:11222"
    cache:
      example:
        configuration: "<distributed-cache/>"
        force-return-values: true
      mycache:
        template-name: "myTemplate"
      "[com.mycompany.mycache]":
        template-name: "myCustomTemplate"
        near-cache:
          mode: "INVALIDATED"
      "[com.mycompany.*]":
        template-name: "myCustomTemplate"
        transaction:
          transaction-mode: "NON_XA"
```

### hotrod-client.properties (alternative)

Properties in `hotrod-client.properties` take priority over `application.properties`:

```properties
infinispan.client.hotrod.cache.mycache.template_name=mytemplate1
infinispan.client.hotrod.cache.mycache.force_return_values=true
infinispan.client.hotrod.cache.[com.myproject.mycache].near_cache.mode=INVALIDATED
```

## Serialization / Marshalling

### ProtoStream (Recommended)

The Spring Boot starter automatically discovers `GeneratedSchema` implementations on the classpath, registers them with the client, and uploads schemas to the server. No manual configuration needed.

1. Annotate your model classes with `@Proto`:

```java
@Proto
public record Book(
    @ProtoField(number = 1) String isbn,
    @ProtoField(number = 2) String title,
    @ProtoField(number = 3) String author
) {}
```

2. Create a `@ProtoSchema` interface:

```java
@ProtoSchema(includeClasses = { Book.class },
             schemaPackageName = "com.example")
public interface BookSchema extends GeneratedSchema {
}
```

The starter handles the rest. To disable auto-registration:

```properties
infinispan.remote.use-schema-registration=false
```

### Java Serialization (Legacy)

If using Java Serialization instead of ProtoStream, add classes to the allow list:

```properties
infinispan.remote.java-serial-allowlist=com.example.model.*
```

## Reactive Support

Starting with Spring 6.1, reactive caching is supported for use in WebFlux applications.

**Important:** If you use `spring-boot-starter-webflux` without enabling reactive mode, your application may block.

### Enable Reactive Mode

```properties
# Remote
infinispan.remote.reactive=true

# Embedded
infinispan.embedded.reactive=true
```

## Cache Manager Configuration Beans

### Remote Mode Customization

```java
// Option 1: InfinispanRemoteConfigurer (only one allowed)
@Bean
public InfinispanRemoteConfigurer infinispanRemoteConfigurer() {
    return () -> new ConfigurationBuilder()
        .addServer().host("127.0.0.1").port(11222)
        .security().authentication()
            .username("admin").password("password")
        .build();
}

// Option 2: InfinispanRemoteCacheCustomizer (multiple allowed, use @Ordered)
@Bean
@Order(1)
public InfinispanRemoteCacheCustomizer customizer() {
    return builder -> builder.statistics().enable();
}
```

### Embedded Mode Customization

```java
// Option 1: InfinispanGlobalConfigurer (only one allowed)
@Bean
public InfinispanGlobalConfigurer globalConfigurer() {
    return () -> GlobalConfigurationBuilder.defaultClusteredBuilder()
        .transport().clusterName("my-cluster")
        .build();
}

// Option 2: InfinispanCacheConfigurer (multiple allowed)
@Bean
public InfinispanCacheConfigurer cacheConfigurer() {
    return manager -> {
        Configuration config = new ConfigurationBuilder()
            .clustering().cacheMode(CacheMode.DIST_SYNC)
            .memory().maxCount(1000)
            .build();
        manager.defineConfiguration("myCache", config);
    };
}

// Option 3: InfinispanGlobalConfigurationCustomizer (multiple allowed)
@Bean
public InfinispanGlobalConfigurationCustomizer globalCustomizer() {
    return builder -> builder.metrics().gauges(true);
}
```

## Spring Boot Actuator Integration

Expose Infinispan cache statistics as metrics:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

Enable statistics on cache instances and bind dynamic caches with `CacheMetricsRegistrar`:

```java
@Autowired
private CacheMetricsRegistrar cacheMetricsRegistrar;

public void registerDynamicCache(Cache cache) {
    cacheMetricsRegistrar.bindCacheToRegistry(cache);
}
```

## Application Properties Reference

### Remote Starter Properties

| Property | Default | Description |
|----------|---------|-------------|
| `infinispan.remote.enabled` | `true` | Enable the remote cache starter |
| `infinispan.remote.server-list` | | Comma-separated list of servers (`host1[:port],host2[:port]`) |
| `infinispan.remote.client-properties` | `classpath:hotrod-client.properties` | Location of the Hot Rod client properties file |
| `infinispan.remote.use-schema-registration` | `true` | Auto-discover and register Protobuf `GeneratedSchema` implementations |
| `infinispan.remote.reactive` | `false` | Enable reactive support for caching |
| `infinispan.remote.read-timeout` | `0` | Read timeout in milliseconds (`0` = no timeout) |
| `infinispan.remote.write-timeout` | `0` | Write timeout in milliseconds (`0` = no timeout) |
| `infinispan.remote.marshaller` | | Fully qualified marshaller class name |
| `infinispan.remote.java-serial-allowlist` | | Comma-separated list or regex for Java serialization allow list |

### Embedded Starter Properties

| Property | Default | Description |
|----------|---------|-------------|
| `infinispan.embedded.enabled` | `true` | Enable embedded capabilities |
| `infinispan.embedded.config-xml` | | Path to XML configuration file (takes priority over bean config) |
| `infinispan.embedded.machine-id` | | Machine identifier for topology-aware consistent hashing |
| `infinispan.embedded.cluster-name` | | Name of the embedded cluster |
| `infinispan.embedded.reactive` | `false` | Enable reactive support for caching |

## Upgrading

### Spring Boot 3.x to 4.x

1. Update Maven dependencies:
   - `infinispan-spring-boot3-starter-remote` to `infinispan-spring-boot4-starter-remote`
   - `infinispan-spring-boot3-starter-embedded` to `infinispan-spring-boot4-starter-embedded`

2. **Breaking change (embedded):** `InfinispanConfigurationCustomizer` is deprecated in Spring Boot 3 and removed in Spring Boot 4. Use `InfinispanCacheConfigurer` instead and explicitly define a `default` cache if needed:

```java
@Bean
public InfinispanCacheConfigurer defaultCacheConfigurer() {
    return manager -> {
        Configuration config = new ConfigurationBuilder()
            .clustering().cacheMode(CacheMode.LOCAL)
            .build();
        manager.defineConfiguration("default", config);
    };
}
```

### Spring 6 to Spring 7 (without Boot)

Update Maven artifact names:
- `infinispan-spring6-embedded` to `infinispan-spring7-embedded`
- `infinispan-spring6-remote` to `infinispan-spring7-remote`

Backward compatibility is maintained. No functionality changes.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Spring Session fails with "cache not found" | Infinispan does not create a default session cache. Create a cache named `sessions` (or your custom name) on the server before starting the app. |
| WebFlux app blocks on cache operations | Enable reactive mode: `infinispan.remote.reactive=true` or `infinispan.embedded.reactive=true` |
| Serialization errors with remote caches | Use ProtoStream with `@Proto` and `@ProtoSchema`. Ensure the server cache encoding is `application/x-protostream`. |
| `hotrod-client.properties` overrides `application.properties` silently | Properties in `hotrod-client.properties` take priority. Remove `hotrod-client.properties` if you want `application.properties` to be the single source. |
| `InfinispanConfigurationCustomizer` not working in Spring Boot 4 | This bean was removed in Spring Boot 4. Use `InfinispanCacheConfigurer` and define caches explicitly. |
| Only one `InfinispanRemoteConfigurer` bean allowed | Use `InfinispanRemoteCacheCustomizer` (multiple allowed, use `@Order`) for additional customizations. |
| Missing `@EnableCaching` annotation | The starter creates the cache manager but does not enable caching annotations. You must add `@EnableCaching` to a configuration class. |
| Per-cache properties not applied | Cache names containing dots must be wrapped in brackets: `infinispan.remote.cache.[com.myproject.mycache].template-name=...` |
| Auto-schema registration uploads wrong schemas | Set `infinispan.remote.use-schema-registration=false` and register schemas manually if needed. |

## Official Docs

- Spring Boot starter: https://infinispan.org/docs/stable/titles/spring_boot/starter.html
- Spring Cache provider: https://infinispan.org/docs/stable/titles/spring/spring.html
- Hot Rod client configuration: https://infinispan.org/docs/stable/titles/hotrod_java/hotrod_java.html
- Encoding and marshalling: https://infinispan.org/docs/stable/titles/encoding/encoding.html
- Spring Cache reference (Spring Framework): https://docs.spring.io/spring-framework/reference/integration/cache.html
