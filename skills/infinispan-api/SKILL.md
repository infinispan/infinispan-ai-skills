---
name: infinispan-api
description: Use when user needs help writing code with Infinispan - embedded mode, Hot Rod client, REST client, Ickle queries, counters, transactions, Spring or Quarkus integration.
---

# Infinispan API Usage Guide

Help users write correct Infinispan code, whether embedded or client-server.

## Workflow

1. Determine if the user is embedding Infinispan or using client-server
2. Identify the specific API area (queries, counters, transactions, etc.)
3. Provide Java code examples with explanations
4. Warn about common mistakes

## Embedded Mode

Infinispan runs in the same JVM as your application.

### Setup

```java
// Programmatic configuration
GlobalConfigurationBuilder global = GlobalConfigurationBuilder.defaultClusteredBuilder();
DefaultCacheManager cacheManager = new DefaultCacheManager(global.build());

ConfigurationBuilder config = new ConfigurationBuilder();
config.clustering().cacheMode(CacheMode.DIST_SYNC)
      .encoding().mediaType("application/x-protostream");

cacheManager.defineConfiguration("myCache", config.build());
Cache<String, String> cache = cacheManager.getCache("myCache");
```

```java
// From XML configuration
DefaultCacheManager cacheManager = new DefaultCacheManager("infinispan.xml");
Cache<String, String> cache = cacheManager.getCache("myCache");
```

**Pitfall:** Always call `cacheManager.stop()` on shutdown. Use try-with-resources or a shutdown hook.

### CDI Integration

```java
@Inject
@Remote("myCache")  // for remote caches
Cache<String, String> cache;
```

## Client-Server: Hot Rod

Hot Rod is Infinispan's binary protocol, optimized for Java clients.

### Maven Dependency

```xml
<dependency>
  <groupId>org.infinispan</groupId>
  <artifactId>infinispan-client-hotrod</artifactId>
</dependency>
```

### Connection Setup

```java
ConfigurationBuilder builder = new ConfigurationBuilder();
builder.addServer()
         .host("localhost")
         .port(11222)
       .security()
         .authentication()
           .username("admin")
           .password("password")
           .realm("default")
           .saslMechanism("SCRAM-SHA-512");

RemoteCacheManager cacheManager = new RemoteCacheManager(builder.build());
RemoteCache<String, String> cache = cacheManager.getCache("myCache");
```

### ProtoStream Marshalling

Required for indexed caches and cross-language compatibility.

```java
// 1. Define your entity with annotations
@Proto
public record Book(
    @ProtoField(number = 1) String title,
    @ProtoField(number = 2) String author,
    @ProtoField(number = 3, defaultValue = "0") int year
) {}

// 2. Create a schema initializer
@AutoProtoSchemaBuilder(
    includeClasses = { Book.class },
    schemaPackageName = "org.example"
)
public interface BookSchema extends GeneratedSchema {}

// 3. Register with the client
builder.addContextInitializer(new BookSchemaImpl());

// 4. Register schema on server
RemoteCache<String, String> metadataCache = cacheManager.getCache("___protobuf_metadata");
BookSchemaImpl schema = new BookSchemaImpl();
metadataCache.put(schema.getProtoFileName(), schema.getProtoFile());
```

**Pitfall:** Forgetting to register the schema on the server is the #1 Hot Rod mistake. You'll get `Unknown type` errors.

### Near Caching

```java
builder.remoteCache("myCache")
       .nearCacheMode(NearCacheMode.INVALIDATED)
       .nearCacheMaxEntries(1000);
```

### Async Operations

```java
CompletableFuture<String> future = cache.putAsync("key", "value");
future.thenAccept(prev -> System.out.println("Previous: " + prev));
```

## Client-Server: REST

```bash
# Put an entry
curl -X POST "http://localhost:11222/rest/v2/caches/myCache/key1" \
  -H "Content-Type: text/plain" \
  -d "value1" \
  -u admin:password

# Get an entry
curl "http://localhost:11222/rest/v2/caches/myCache/key1" -u admin:password

# Delete an entry
curl -X DELETE "http://localhost:11222/rest/v2/caches/myCache/key1" -u admin:password

# Query
curl "http://localhost:11222/rest/v2/caches/myCache?action=search&query=FROM%20org.example.Book%20WHERE%20author%3D'Tolkien'" \
  -u admin:password
```

### Java REST Client

```java
RestClient restClient = RestClient.forAddresses(
    new RestClientConfigurationBuilder()
        .addServer().host("localhost").port(11222)
        .security().authentication()
          .username("admin").password("password")
        .build()
);
RestCacheClient cacheClient = restClient.cache("myCache");
```

## Queries (Ickle)

Ickle is Infinispan's query language, similar to JP-QL/HQL.

### Syntax

```sql
-- Basic query
FROM org.example.Book WHERE author = 'Tolkien'

-- Projection
SELECT title, year FROM org.example.Book WHERE year > 2000

-- Full-text search (requires indexing)
FROM org.example.Book WHERE title : 'ring'

-- Aggregation
SELECT author, COUNT(title) FROM org.example.Book GROUP BY author

-- Sorting and pagination
FROM org.example.Book ORDER BY year DESC OFFSET 10 LIMIT 20

-- Delete
DELETE FROM org.example.Book WHERE year < 1950
```

### Java API

```java
QueryFactory queryFactory = Search.getQueryFactory(cache);
Query<Book> query = queryFactory.create("FROM org.example.Book WHERE author = :author");
query.setParameter("author", "Tolkien");
List<Book> results = query.execute().list();
```

### Continuous Queries

```java
QueryFactory queryFactory = Search.getQueryFactory(cache);
Query<Book> query = queryFactory.create("FROM org.example.Book WHERE year > 2020");

ContinuousQuery<String, Book> cq = Search.getContinuousQuery(cache);
cq.addContinuousQueryListener(query, new ContinuousQueryListener<String, Book>() {
    @Override public void resultJoining(String key, Book value) { /* new match */ }
    @Override public void resultUpdated(String key, Book value) { /* updated match */ }
    @Override public void resultLeaving(String key) { /* no longer matches */ }
});
```

**Pitfall:** Non-indexed queries scan all entries. For large caches, always index fields you query on.

## Counters

### Strong Counter (linearizable)

```java
CounterManager counterManager = RemoteCounterManagerFactory.asCounterManager(cacheManager);

// Define
counterManager.defineCounter("visits",
    CounterConfiguration.builder(CounterType.BOUNDED_STRONG)
        .initialValue(0)
        .lowerBound(0)
        .storage(Storage.PERSISTENT)
        .build());

// Use
StrongCounter counter = counterManager.getStrongCounter("visits");
CompletableFuture<Long> value = counter.incrementAndGet();
```

### Weak Counter (faster, eventually consistent)

```java
counterManager.defineCounter("approx-count",
    CounterConfiguration.builder(CounterType.WEAK)
        .initialValue(0)
        .concurrencyLevel(4)
        .build());

WeakCounter counter = counterManager.getWeakCounter("approx-count");
counter.increment();
```

## Transactions

### Configuration

```xml
<distributed-cache name="tx-cache">
  <transaction mode="NON_XA" locking="OPTIMISTIC"/>
</distributed-cache>
```

**Transaction modes:**
- `NON_XA` — Simple transactions without XA recovery. Good for most use cases.
- `NON_DURABLE_XA` — XA without recovery logs. Better integration with JTA.
- `FULL_XA` — Full XA with recovery. Use when participating in distributed transactions.

**Locking:**
- `OPTIMISTIC` — Locks acquired at commit time. Better throughput, risk of WriteSkewException.
- `PESSIMISTIC` — Locks acquired on write. No write skew, lower throughput.

### Java API

```java
TransactionManager tm = cache.getAdvancedCache().getTransactionManager();

tm.begin();
try {
    cache.put("key1", "value1");
    cache.put("key2", "value2");
    tm.commit();
} catch (Exception e) {
    tm.rollback();
    throw e;
}
```

### Batching (Embedded)

```java
cache.getAdvancedCache().startBatch();
cache.put("key1", "value1");
cache.put("key2", "value2");
cache.getAdvancedCache().endBatch(true); // true = commit, false = rollback
```

## Spring Integration

### Spring Boot Starter

```xml
<dependency>
  <groupId>org.infinispan</groupId>
  <artifactId>infinispan-spring-boot3-starter-remote</artifactId>
</dependency>
```

```properties
# application.properties
infinispan.remote.server-list=localhost:11222
infinispan.remote.auth-username=admin
infinispan.remote.auth-password=password
```

### Spring Cache Abstraction

```java
@Cacheable(value = "books", key = "#isbn")
public Book findBook(String isbn) { ... }

@CacheEvict(value = "books", key = "#isbn")
public void deleteBook(String isbn) { ... }

@CachePut(value = "books", key = "#isbn")
public Book updateBook(String isbn, Book book) { ... }
```

### Spring Session

```xml
<dependency>
  <groupId>org.infinispan</groupId>
  <artifactId>infinispan-spring6-session</artifactId>
</dependency>
```

## Quarkus Integration

### Extension

```xml
<dependency>
  <groupId>org.infinispan</groupId>
  <artifactId>infinispan-quarkus-client</artifactId>
</dependency>
```

```properties
# application.properties
quarkus.infinispan-client.hosts=localhost:11222
quarkus.infinispan-client.username=admin
quarkus.infinispan-client.password=password
```

### Injection

```java
@Inject
@Remote("myCache")
RemoteCache<String, Book> cache;
```

### Dev Services

Quarkus automatically starts an Infinispan dev server. No manual config needed in dev mode.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| `Unknown type` errors with Hot Rod | Register ProtoStream schema on both client and server |
| CacheManager not stopped on shutdown | Use try-with-resources or shutdown hook |
| Query scans all entries (slow) | Add indexing to the cache for queried fields |
| Stale reads with near cache | Configure near cache invalidation and max entries |
| Transaction timeout | Increase lock timeout or check for deadlocks |

## Official Docs

- Hot Rod client: https://infinispan.org/docs/stable/titles/hotrod_java/hotrod_java.html
- REST API: https://infinispan.org/docs/stable/titles/rest/rest.html
- Querying: https://infinispan.org/docs/stable/titles/query/query.html
- Embedding: https://infinispan.org/docs/stable/titles/embedding/embedding.html
