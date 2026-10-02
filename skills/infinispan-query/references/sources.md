# Infinispan Query - References

## Official Documentation

- Query guide: https://infinispan.org/docs/stable/titles/query/query.html
- Indexing caches: https://infinispan.org/docs/stable/titles/query/query.html#indexing-caches
- Ickle query language: https://infinispan.org/docs/stable/titles/query/query.html#ickle-query-language
- Full-text search: https://infinispan.org/docs/stable/titles/query/query.html#full-text-queries
- Vector search: https://infinispan.org/docs/stable/titles/query/query.html#vector-search
- Spatial search: https://infinispan.org/docs/stable/titles/query/query.html#spatial-search
- Continuous queries: https://infinispan.org/docs/stable/titles/query/query.html#continuous-queries
- Performance tuning: https://infinispan.org/docs/stable/titles/query/query.html#query-performance-tuning

## Key API Packages

- `org.infinispan.api.annotations.indexing` — Indexing annotations (`@Indexed`, `@Basic`, `@Text`, `@Keyword`, `@Decimal`, `@Vector`, `@GeoField`, `@GeoPoint`, `@Embedded`)
- `org.infinispan.api.annotations.indexing.option` — Options (`VectorSimilarity`)
- `org.infinispan.api.annotations.indexing.model` — Models (`LatLng`)
- `org.infinispan.query.dsl` — Query DSL (`QueryFactory`, `Query`)
- `org.infinispan.query` — Query utilities (`Search.getQueryFactory()`)
- `org.infinispan.query.api.continuous` — Continuous queries

## Tutorials

- Remote query tutorial: https://github.com/infinispan/infinispan-simple-tutorials/tree/main/remote-query
- Embedded query tutorial: https://github.com/infinispan/infinispan-simple-tutorials/tree/main/embedded-query

## Provenance

This skill targets **Infinispan 16.x**. The annotation API was introduced in Infinispan 14.0, with `@GeoField`, `@GeoPoint`, and spatial queries added in Infinispan 15.1. Content was derived from the official Infinispan documentation and verified against the source repository.
