---
name: infinispan-query
description: Use when user needs help with Infinispan queries - Ickle query language, full-text search, vector search, spatial search, indexing configuration, continuous queries, query performance tuning.
---

# Infinispan Query and Search Guide

Help users query and search data in Infinispan caches using the Ickle query language, full-text search, vector search, spatial search, and continuous queries. Covers indexing configuration, entity mapping, analyzers, and performance tuning. Default version: Infinispan 16.x.

## Workflow

1. Determine query type: relational, full-text, vector, spatial, or continuous
2. Check if the cache is indexed (required for full-text, vector, and spatial queries)
3. Verify entity mapping annotations are correct
4. Provide Ickle query examples with Java code
5. Warn about common pitfalls and performance considerations

## Ickle Query Language

Ickle is a subset of JPQL with full-text extensions. It supports SELECT, FROM, WHERE, GROUP BY, HAVING, ORDER BY, DELETE, and UPDATE.

### Basic Syntax

```sql
[SELECT projection [, ...]]
FROM <entityName>
[WHERE condition]
[GROUP BY field [, ...] [HAVING condition]]
[ORDER BY field [ASC | DESC] [, ...]]
```

### Parser Rules

- Whitespace is not significant
- Wildcards are not supported in field names
- A field name or path must always be specified (no default field)
- `&&` and `||` are accepted instead of `AND` or `OR`
- `!` may be used instead of `NOT`
- A missing boolean operator is interpreted as `OR`
- String terms must be enclosed in single or double quotes
- `!=` is accepted instead of `<>`

### Filtering Operators

| Operator | Description | Example |
|----------|-------------|---------|
| `=` | Exact match | `FROM Book WHERE name = 'Java'` |
| `!=` | Not equal | `FROM Book WHERE language != 'English'` |
| `>` | Greater than | `FROM Book WHERE price > 20` |
| `>=` | Greater or equal | `FROM Book WHERE price >= 20` |
| `<` | Less than | `FROM Book WHERE year < 2020` |
| `<=` | Less or equal | `FROM Book WHERE price <= 50` |
| `in` | In collection | `FROM Book WHERE isbn IN ('ZZ', 'X1234')` |
| `like` | Wildcard match (JPA rules) | `FROM Book WHERE title LIKE '%Java%'` |
| `between` | Range | `FROM Book WHERE price BETWEEN 50 AND 100` |

### Boolean Conditions

```sql
-- AND has higher priority than OR
FROM org.infinispan.sample.Book WHERE title LIKE '%Data Grid%' OR author.name = 'Manik' AND description like '%clustering%'

-- Use parentheses to change precedence
FROM org.infinispan.sample.Book WHERE author.name = 'Manik' AND (title like '%Data Grid%' OR description like '%clustering%')
```

### Projections with SELECT

Return specific fields instead of entire entities. Results come back as `List<Object[]>`.

```java
// Project specific fields
Query<Object[]> query = cache.query(
    "SELECT title, publicationYear FROM org.infinispan.sample.Book WHERE title like '%Data Grid%'");
List<Object[]> results = query.execute().list();

// Project cache entry version
Query<Object[]> query = cache.query(
    "SELECT b.title, version(b) FROM org.infinispan.sample.Book b WHERE b.title like '%Data Grid%'");

// Project score (indexed queries only)
Query<Object[]> query = cache.query(
    "SELECT b, score(b) FROM org.infinispan.sample.Book b WHERE b.title : 'infinispan'");
```

### Sorting

```sql
FROM org.infinispan.sample.Book WHERE title like '%Data Grid%' ORDER BY publicationYear DESC, title ASC
```

### Nested Object Joins

When using `NESTED` embedded entities, you can use joins to query parent-child relationships:

```java
@Proto
@Indexed
public record Team(
    @Basic String name,
    @Embedded(structure = Structure.NESTED) List<Player> firstTeam,
    @Embedded(structure = Structure.FLATTENED) List<Player> replacements) {
}

@Proto
public record Player(@Basic String name, @Basic String color, @Basic Integer number) {
}
```

```sql
-- Find teams with a player having number 7 and color red or blue
SELECT t.name FROM model.Team t JOIN t.firstTeam p
WHERE (p.color = 'red' AND p.number = 7) OR (p.color = 'blue' AND p.number = 7)
```

**Important:** NESTED preserves the original object relationship structure. FLATTENED makes leaf fields multi-valued on the parent (no join support).

### Grouping and Aggregation

Supported functions: `avg`, `sum`, `count`, `max`, `min`.

```sql
SELECT author, COUNT(title) FROM org.infinispan.sample.Book WHERE title LIKE '%engine%' GROUP BY author
```

Rules:
- All projected fields must either be grouping fields or aggregated
- HAVING clause filters after grouping (can reference aggregated fields)
- WHERE clause filters before grouping (works on entity fields)

### DELETE Statements

```sql
DELETE FROM org.infinispan.sample.Book WHERE publicationYear < 2000
```

Execute with `query.executeStatement()`. Cannot use projections, grouping, aggregation, or ORDER BY.

### UPDATE Statements

```sql
-- SET replaces a value; ADD adds to a collection; REMOVE removes from a collection
UPDATE FROM org.infinispan.sample.Book SET title = 'Updated Title', ADD tags = ('sale', 'featured'), REMOVE tags = ['discount'] WHERE price > 100
```

Execute with `query.executeStatement()`. Cannot use projections, grouping, aggregation, or ORDER BY.

### Query Parameters

```java
Query<Book> query = cache.query("FROM book_sample.Book WHERE publicationYear > :year");
query.setParameter("year", 2015);
List<Book> results = query.execute().list();
```

## Full-Text Search

Full-text queries require indexed caches and fields annotated with `@Text` or `@Keyword`. Use the `:` operator instead of `=`.

### Full-Text Predicate Syntax

```sql
<analyzed field> : '<term or phrase>' [~distance]
<analyzed field> : [<lower bound> to <upper bound>]
<analyzed field> : /regular expression/
```

### Full-Text Query Types

| Type | Syntax | Example |
|------|--------|---------|
| Term match | `field : 'term'` | `WHERE description : 'clustering'` |
| Phrase match | `field : 'multiple words'` | `WHERE description : 'bus fare'` |
| Fuzzy | `field : 'term'~N` | `WHERE description : 'cofee'~2` |
| Proximity | `field : 'term1 term2'~N` | `WHERE description : 'canceling fee'~3` |
| Wildcard (single char) | `field : 'te?t'` | `WHERE description : 'te?t'` |
| Wildcard (multi char) | `field : 'test*'` | `WHERE description : 'test*'` |
| Range | `field : [lower to upper]` | `WHERE amount : [20 to 50]` |
| Regexp | `field : /regexp/` | `WHERE title : /[mb]oat/` |
| Boosting | `field : term^N` | `WHERE title : beer^3 OR wine` |

### Full-Text Java Example

```java
// Full-text search using ':' operator
Query<Book> fullTextQuery = cache.query(
    "FROM org.infinispan.sample.Book b WHERE b.title:'infinispan' AND b.authors.name:'sanne'");

// Combine full-text and non-full-text operators
Query<Book> query = cache.query(
    "FROM org.infinispan.sample.Book b WHERE b.isbn = '12345678' AND b.authors.name : 'sanne'");

// Boolean full-text operators (+required, -excluded)
Query<Book> query = cache.query(
    "FROM org.infinispan.sample.Book b WHERE b.description : (+'dark' -'tower')");

List<Book> found = query.execute().list();
```

**Important:** The `:` operator only works on analyzed/indexed fields (`@Text`, `@Keyword`). Use `=` for non-analyzed fields (`@Basic`).

## Vector Search (kNN)

Vector search enables k-nearest-neighbor queries for similarity search, embeddings, and AI/ML workloads. Requires indexed caches with `@Vector` annotated fields.

### Vector Field Mapping

```java
@Indexed
public class Item {
    @Basic
    @ProtoField(1)
    String name;

    @Vector(dimension = 3, similarity = VectorSimilarity.L2)
    @ProtoField(2)
    float[] floatVector;
}
```

### Vector Field Attributes

| Attribute | Description | Default |
|-----------|-------------|---------|
| `dimension` | Vector dimensions (required) | - |
| `similarity` | Distance metric: `L2`, `INNER_PRODUCT`, `MAX_INNER_PRODUCT`, `COSINE` | `L2` |
| `beamWidth` | Size of dynamic list during kNN graph creation. Higher = more accurate, slower indexing | 512 |
| `maxConnections` | Number of neighbors per node in HNSW graph. Keep between 2-100 | 16 |

### Similarity Metrics

| Metric | Use When |
|--------|----------|
| `L2` (Euclidean) | General purpose. Default. Score = 1/(1+d^2) |
| `INNER_PRODUCT` | Both index and search vectors are normalized |
| `MAX_INNER_PRODUCT` | Does not require vector normalization |
| `COSINE` | Angular similarity. Cannot use zero-magnitude vectors |

### kNN Query Syntax

```sql
-- Find 3 nearest neighbors to vector [7,7,7]
FROM play.Item i WHERE i.myVector <-> [7,7,7]~3
```

### Vector Search with Parameters

```java
// Pass vector as parameter
Query<Item> query = cache.query("FROM play.Item i WHERE i.floatVector <-> [:v]~:k");
query.setParameter("v", new float[]{7.1f, 7.0f, 3.1f});
query.setParameter("k", 3);

// Or pass individual components
Query<Item> query = cache.query("FROM play.Item i WHERE i.floatVector <-> [:a,:b,:c]~:k");
query.setParameter("a", 1);
query.setParameter("b", 4.3);
query.setParameter("c", 3.3);
query.setParameter("k", 4);
```

### Vector Search with Score Projection

```java
Query<Object[]> query = cache.query(
    "SELECT i, score(i) FROM play.Item i WHERE i.floatVector <-> [:a]~:k");
query.setParameter("a", new float[]{7.1f, 7.0f, 3.1f});
query.setParameter("k", 3);
List<Object[]> resultList = query.list();
// resultList[n][0] = entity, resultList[n][1] = score
```

### Vector Search with Filtering

Apply classic predicates to limit the search set before kNN:

```java
Query<Object[]> query = remoteCache.query(
    "SELECT score(i), i FROM Item i WHERE i.floatVector <-> [:a]~:k " +
    "FILTERING (i.category : 'electronics' OR i.text : 'code')");
query.setParameter("a", new float[]{7, 7, 7});
query.setParameter("k", 3);
```

**Important:** Only one vector predicate per query is allowed. The filtering clause cannot contain kNN predicates.

### Supported Vector Types

- `float` / `Float` (float vectors)
- `byte` / `Byte` (byte vectors)

## Spatial Search (Geo Queries)

Spatial queries enable geographic searches on indexed caches. They require `@GeoPoint` or `@GeoField` annotations.

**Important:** Spatial queries are only supported on indexed caches.

### Spatial Field Mapping - @GeoPoint

Use `@GeoPoint` at the type level with `@Latitude` and `@Longitude` on fields:

```java
@Proto
@Indexed
@GeoPoint(fieldName = "location", projectable = true, sortable = true)
public record Restaurant(
    @Keyword(normalizer = "lowercase", projectable = true, sortable = true) String name,
    @Text String description,
    @Text String address,
    @Latitude(fieldName = "location") Double latitude,
    @Longitude(fieldName = "location") Double longitude,
    @Basic Float score
) {
    @ProtoSchema(
        includeClasses = { Restaurant.class },
        schemaFileName = "geo.proto",
        schemaPackageName = "geo",
        syntax = ProtoSyntax.PROTO3
    )
    public interface RestaurantSchema extends GeneratedSchema {
        RestaurantSchema INSTANCE = new RestaurantSchemaImpl();
    }
}
```

Multiple spatial points per entity are supported (each must have a unique `fieldName`):

```java
@Proto
@Indexed
@GeoPoint(fieldName = "departure", sortable = true, projectable = true)
@GeoPoint(fieldName = "arrival", sortable = true, projectable = true)
public record TrainRoute(
    @Basic String name,
    @Latitude(fieldName = "departure") Double departureLat,
    @Longitude(fieldName = "departure") Double departureLon,
    @Latitude(fieldName = "arrival") Double arrivalLat,
    @Longitude(fieldName = "arrival") Double arrivalLon
) {}
```

### Spatial Field Mapping - @GeoField

Use `@GeoField` with `LatLng` type for a simpler mapping:

```java
// Embedded queries
import org.infinispan.api.annotations.indexing.model.LatLng;

@Indexed
public record Hiking(@Keyword String name, @GeoField LatLng start, @GeoField LatLng end) {}
```

```java
// Remote queries
import org.infinispan.commons.api.query.geo.LatLng;

@Proto
@Indexed
public record ProtoHiking(
    @Keyword String name,
    @GeoField LatLng start,
    @GeoField LatLng end
) {
    @ProtoSchema(
        includeClasses = { ProtoHiking.class },
        dependsOn = LatLng.LatLngSchema.class,  // Required dependency
        schemaFileName = "hiking.proto",
        schemaPackageName = "hiking"
    )
    public interface HikingSchema extends GeneratedSchema {}
}
```

### Spatial Predicates

**Within Circle** (find entities within a distance from a point):

```java
// Within 100 meters (default unit)
Query<Restaurant> query = cache.query(
    "FROM geo.Restaurant r WHERE r.location within circle(41.91, 12.46, :distance)");
query.setParameter("distance", 100);

// Within 0.1 kilometers
Query<Restaurant> query = cache.query(
    "FROM geo.Restaurant r WHERE r.location within circle(41.91, 12.46, :distance km)");
query.setParameter("distance", 0.1);
```

**Within Box** (find entities within a rectangle):

```java
// Arguments: topLeftLat, topLeftLon, bottomRightLat, bottomRightLon
Query<Restaurant> query = cache.query(
    "FROM geo.Restaurant r WHERE r.location within box(41.91, 12.45, 41.90, 12.46)");
```

**Within Polygon** (find entities within an arbitrary polygon):

```java
Query<Restaurant> query = cache.query(
    "FROM geo.Restaurant r WHERE r.location within polygon((41.91, 12.45), (41.91, 12.46), (41.90, 12.46), (41.90, 12.45))");
```

### Spatial Sorting

Sort by distance from a query point (spatial field must be `sortable = true`):

```java
Query<Restaurant> query = cache.query(
    "FROM geo.Restaurant r ORDER BY distance(r.location, 41.91, 12.46)");
```

### Spatial Distance Projection

Project distances (spatial field must be `projectable = true`):

```java
// Distance in meters (default)
Query<Object[]> query = remoteCache.query(
    "SELECT r.name, distance(r.location, 41.91, 12.46) FROM geo.Restaurant r");

// Distance in yards
Query<Object[]> query = remoteCache.query(
    "SELECT r.name, distance(r.location, 41.91, 12.46, yd) FROM geo.Restaurant r");
```

### Supported Distance Units

| Unit | Symbol |
|------|--------|
| Meters (default) | `m` |
| Kilometers | `km` |
| Miles | `mi` |
| Yards | `yd` |
| Nautical miles | `nm` |

## Indexing Configuration

### Enabling Indexing

Indexing is required for full-text, vector, and spatial queries. It significantly improves query performance.

**XML:**
```xml
<distributed-cache name="myCache">
  <encoding media-type="application/x-protostream"/>
  <indexing storage="filesystem">
    <indexed-entities>
      <indexed-entity>book_sample.Book</indexed-entity>
    </indexed-entities>
  </indexing>
</distributed-cache>
```

**YAML:**
```yaml
distributedCache:
  name: myCache
  encoding:
    mediaType: application/x-protostream
  indexing:
    storage: filesystem
    indexedEntities:
      - book_sample.Book
```

**Programmatic Java:**
```java
ConfigurationBuilder builder = new ConfigurationBuilder();
builder.indexing().enable()
       .storage(IndexStorage.FILESYSTEM)
       .addIndexedEntity(Book.class);
```

### Index Storage

| Storage | Description | Use When |
|---------|-------------|----------|
| `filesystem` | Persisted to disk. Default. Survives restarts. | Production |
| `local-heap` | In JVM memory. Lost on restart. | Development, small datasets |

### Index Startup Mode

| Mode | Behavior |
|------|----------|
| `AUTO` (default) | Checks index format; purges if data volatile + index persistent; reindexes if data persistent + index volatile |
| `PURGE` | Clears the index on cache start |
| `REINDEX` | Rebuilds the index on cache start (async, may show partial results during rebuild) |
| `NONE` | No indexing operation on startup |

```xml
<!-- Purge on startup -->
<indexing storage="filesystem" startup-mode="PURGE">
  <indexed-entities>
    <indexed-entity>book_sample.Book</indexed-entity>
  </indexed-entities>
</indexing>

<!-- Reindex on startup -->
<indexing storage="filesystem" startup-mode="REINDEX">
  <indexed-entities>
    <indexed-entity>book_sample.Book</indexed-entity>
  </indexed-entities>
</indexing>
```

### Indexing Mode

| Mode | Description |
|------|-------------|
| `auto` (default) | Changes are immediately propagated to indexes |
| `manual` | Indexes updated only on explicit reindex. Use for batch updates. |

### Index Sharding

Split index data into multiple shards for large datasets:

```xml
<indexing storage="filesystem" shards="3">
  <indexed-entities>
    <indexed-entity>book_sample.Book</indexed-entity>
  </indexed-entities>
</indexing>
```

### Index Writer Tuning

```xml
<indexing storage="filesystem">
  <index-writer commit-interval="2000"
                ram-buffer-size="64"
                queue-count="4"
                queue-size="4000">
    <index-merge max-entries="10000"
                 factor="10"
                 min-size="10"
                 max-size="1024"/>
  </index-writer>
  <indexed-entities>
    <indexed-entity>book_sample.Book</indexed-entity>
  </indexed-entities>
</indexing>
```

| Writer Attribute | Description | Default |
|-----------------|-------------|---------|
| `commit-interval` | Milliseconds between flushes to storage. Avoid very small values. | 1000 |
| `ram-buffer-size` | Max memory (MB) for buffering before flush | - |
| `max-buffered-entries` | Max entries buffered before flush | - |
| `queue-count` | Internal queues per indexed type (parallel processing) | 4 |
| `queue-size` | Max elements per queue. Too small causes backpressure exceptions. | 4000 |

### Index Reader Tuning

```xml
<indexing storage="filesystem">
  <index-reader refresh-interval="1000"/>
  <indexed-entities>
    <indexed-entity>book_sample.Book</indexed-entity>
  </indexed-entities>
</indexing>
```

- `refresh-interval=0` (default): reads the latest data before each query
- A value > 0: may return stale results but substantially increases throughput in write-heavy scenarios

### Rebuilding Indexes

```java
// Remote cache
remoteCacheManager.administration().reindexCache("MyCache");

// Embedded cache
Indexer indexer = Search.getIndexer(cache);
CompletionStage<Void> future = indexer.run();
```

Rebuild indexes when you change indexed type definitions, analyzers, or after deleting indexes.

**Warning:** Rebuilding can take a long time for large caches and may return partial results while in progress.

## Entity Mapping Annotations

Infinispan uses its own indexing annotations (in `org.infinispan.api.annotations.indexing`).

### Core Annotations

| Annotation | Use For | Key Attributes |
|------------|---------|----------------|
| `@Indexed` | Mark entity for indexing | - |
| `@Basic` | Numbers, short strings (no analysis) | searchable, sortable, projectable, aggregable, indexNullAs |
| `@Decimal` | Decimal numeric values | searchable, sortable, projectable, aggregable, indexNullAs, decimalScale |
| `@Keyword` | Exact match strings (not analyzed) | searchable, sortable, projectable, aggregable, indexNullAs, normalizer, norms |
| `@Text` | Full-text searchable text (analyzed) | searchable, projectable, norms, analyzer, searchAnalyzer |
| `@Vector` | Vector embeddings for kNN search | searchable, projectable, dimension, similarity, beamWidth, maxConnections |
| `@GeoField` | Spatial point using LatLng | projectable, sortable |
| `@Embedded` | Embedded objects (NESTED or FLATTENED) | structure |

### Entity Example - Remote (ProtoStream)

```java
import org.infinispan.api.annotations.indexing.Basic;
import org.infinispan.api.annotations.indexing.Indexed;
import org.infinispan.api.annotations.indexing.Text;
import org.infinispan.protostream.annotations.ProtoFactory;
import org.infinispan.protostream.annotations.ProtoField;

@Indexed
public class Book {
    @Text
    @ProtoField(number = 1)
    final String title;

    @Text
    @ProtoField(number = 2)
    final String description;

    @Basic
    @ProtoField(number = 3, defaultValue = "0")
    final int publicationYear;

    @ProtoFactory
    Book(String title, String description, int publicationYear) {
        this.title = title;
        this.description = description;
        this.publicationYear = publicationYear;
    }
}
```

### Entity Example - Embedded

```java
import org.infinispan.api.annotations.indexing.*;

@Indexed
public class Book {
    @Keyword
    String title;

    @Text
    String description;

    @Keyword
    String isbn;

    @Basic
    LocalDate publicationDate;

    @Embedded
    Set<Author> authors = new HashSet<>();
}
```

### Protobuf Schema Annotations

You can annotate Protobuf schema files directly:

```protobuf
/**
 * @Indexed
 */
message Book {
    /** @Text */
    optional string title = 1;

    /** @Basic */
    optional int32 publicationYear = 2;
}
```

## Analyzers

### Built-in Analyzers

| Analyzer | Description |
|----------|-------------|
| `standard` | Splits on whitespace and punctuation |
| `simple` | Tokenizes at non-letters, lowercases |
| `whitespace` | Splits on whitespace only |
| `keyword` | Treats entire field as single token |
| `stemmer` | English word stemming (Snowball Porter) |
| `ngram` | Generates n-gram tokens (default 3-grams) |
| `filename` | Like standard but larger tokens, lowercased |
| `lowercase` | Lowercases without tokenizing (normalizer) |

### Using Analyzers

```java
@Text(analyzer = "standard")
String description;

@Keyword(normalizer = "lowercase")
String category;
```

### Custom Analyzers

Create custom analyzer definitions by implementing `ProgrammaticSearchMappingProvider`:

1. Implement the `ProgrammaticSearchMappingProvider` API
2. Package in a JAR with the FQN in `META-INF/services/org.infinispan.query.spi.ProgrammaticSearchMappingProvider`
3. Copy JAR to `server/lib` directory
4. Restart Infinispan Server (classes loaded at startup only)

## Continuous Queries

Continuous queries provide real-time notifications about cache entries matching a query filter. They push events instead of requiring polling.

### Events

| Event | When |
|-------|------|
| `Join` | An entry matches the query |
| `Update` | A matching entry is modified and still matches |
| `Leave` | An entry no longer matches the query |

### Continuous Query Example (Embedded)

```java
import org.infinispan.query.api.continuous.ContinuousQuery;
import org.infinispan.query.api.continuous.ContinuousQueryListener;
import org.infinispan.query.Search;

Cache<Integer, Person> cache = ...;

// Create ContinuousQuery on the cache
ContinuousQuery<Integer, Person> continuousQuery = Search.getContinuousQuery(cache);

// Define query
Query query = cache.query("FROM Person p WHERE p.age < 21");

Map<Integer, Person> matches = new ConcurrentHashMap<>();

ContinuousQueryListener<Integer, Person> listener = new ContinuousQueryListener<>() {
    @Override
    public void resultJoining(Integer key, Person value) {
        matches.put(key, value);
    }

    @Override
    public void resultUpdated(Integer key, Person value) {
        // Handle update
    }

    @Override
    public void resultLeaving(Integer key) {
        matches.remove(key);
    }
};

// Register
continuousQuery.addContinuousQueryListener(query, listener);

// Later, remove the listener
continuousQuery.removeContinuousQueryListener(listener);
```

### Continuous Query (Remote)

```java
import org.infinispan.client.hotrod.Search;

ContinuousQuery<Integer, Person> continuousQuery =
    Search.getContinuousQuery(remoteCache);
```

### Continuous Query Limitations

- Cannot use grouping, aggregation, or sorting operations
- Evaluate each cache operation, so consider performance impact on write-heavy caches

## Embedded vs Remote Querying

| Feature | Embedded | Remote (Hot Rod) |
|---------|----------|------------------|
| Entity mapping | Java annotations directly | ProtoStream annotations + `.proto` schema upload |
| Encoding | Java objects or ProtoStream | ProtoStream required for indexing |
| Query API | `cache.query(ickleString)` | `remoteCache.query(ickleString)` |
| Entity name | Fully qualified Java class | Protobuf message name with package |
| Continuous queries | `org.infinispan.query.Search` | `org.infinispan.client.hotrod.Search` |
| Statistics | `Search.getSearchStatistics(cache)` | REST: `GET /rest/v2/caches/{name}/search/stats` |
| Index rebuild | `Search.getIndexer(cache).run()` | `remoteCacheManager.administration().reindexCache("name")` |
| Schema registration | Not needed | Upload `.proto` to server |
| LatLng type | `org.infinispan.api.annotations.indexing.model.LatLng` | `org.infinispan.commons.api.query.geo.LatLng` |

### Remote Query Setup

```java
// 1. Add ProtoStream annotations to your entity
@Indexed
public class Book {
    @Text @ProtoField(1)  String title;
    @Text @ProtoField(2)  String description;
    @Basic @ProtoField(3, defaultValue = "0") int publicationYear;
}

// 2. Create a SerializationContextInitializer
@ProtoSchema(includeClasses = Book.class,
             schemaFileName = "book.proto",
             schemaPackageName = "book_sample")
public interface RemoteQueryInitializer extends GeneratedSchema {}

// 3. Register with client and upload schema to server
ConfigurationBuilder clientBuilder = new ConfigurationBuilder();
clientBuilder.addServer().host("127.0.0.1").port(11222)
    .security().authentication().username("user").password("user")
    .addContextInitializers(new RemoteQueryInitializerImpl());

RemoteCacheManager rcm = new RemoteCacheManager(clientBuilder.build());

// Upload protobuf schema
Path proto = Paths.get(getClass().getClassLoader().getResource("proto/book.proto").toURI());
SchemaOpResult result = rcm.administration().schemas()
    .createOrUpdate(Schema.buildFromStringContent("book.proto", Files.readString(proto)));
if (result.hasError()) {
    System.out.println(result.getError());
}

// 4. Query
RemoteCache<Object, Object> cache = rcm.getCache("books");
Query<Book> query = cache.query("FROM book_sample.Book WHERE title:'java'");
List<Book> list = query.execute().list();
```

## Query Performance Tuning

### Checklist

1. **Ensure queries use indexes**: Non-indexed queries are much slower on large caches
2. **Index all queried fields**: Partially indexed caches return slower results
3. **Adjust commit-interval**: Default 1000ms. Larger values reduce indexing overhead but increase delay
4. **Adjust refresh-interval**: Default 0 (always fresh). Values > 0 improve throughput in write-heavy scenarios at the cost of slightly stale results
5. **Use sharding**: For large datasets, split indexes into multiple shards
6. **Use projections**: Return only needed fields instead of full entities
7. **Limit result sets**: Use `query.maxResults(N)` and `query.startOffset(N)` for pagination

### Getting Statistics

```java
// Embedded - local node
SearchStatistics statistics = Search.getSearchStatistics(cache);

// Embedded - cluster-wide
CompletionStage<SearchStatisticsSnapshot> statistics = Search.getClusteredSearchStatistics(cache);
```

```
// Remote - REST API
GET /rest/v2/caches/{cacheName}/search/stats
```

## Common Mistakes

| Mistake | Solution |
|---------|----------|
| Using `:` on non-analyzed fields | Use `=` for `@Basic` and `@Keyword` fields. `:` is for `@Text` fields. |
| Missing encoding for indexed cache | Set encoding to `application/x-protostream` (done implicitly for remote caches with indexing) |
| Not registering protobuf schema | Upload `.proto` schema to server before querying remote caches |
| Entity name mismatch | Remote: use Protobuf package + message name (e.g., `book_sample.Book`). Embedded: use FQN. |
| Full-text on non-indexed cache | Full-text search requires indexing. Enable it in cache configuration. |
| Sorting on non-sortable field | Add `sortable = true` to the annotation: `@Basic(sortable = true)` |
| Projecting non-projectable field | Add `projectable = true` to the annotation |
| Vector with wrong dimension | Vector query dimension must match the `dimension` attribute on `@Vector` |
| Multiple kNN predicates in one query | Only one vector predicate per query is allowed |
| Spatial query on non-indexed cache | Spatial queries only work on indexed caches |
| CacheBackpressureFullException during indexing | Increase `queue-size` in index writer config or set `queue-count` to 1 |
| Stale query results after writes | Decrease `refresh-interval` or set to 0 for always-fresh reads |

## Official Docs

- Query guide: https://infinispan.org/docs/stable/titles/query/query.html
- Indexing: https://infinispan.org/docs/stable/titles/query/query.html#indexing-caches
- Ickle syntax: https://infinispan.org/docs/stable/titles/query/query.html#ickle-query-language
